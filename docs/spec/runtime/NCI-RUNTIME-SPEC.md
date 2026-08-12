# NCI Runtime Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §14, §16–§17; `02-NCI-ARCHITECTURE.md`

---

## Purpose

This document specifies the NCI Runtime: execution orchestration, dynamic execution model, compute budgets, backend dispatch, and the runtime contract.

## Scope

Runtime lifecycle, execution model, compute budgets, backend selection, hardware abstraction, and the runtime contract.

---

## 1. Runtime Lifecycle

### 1.1 States

```
UNINITIALIZED
  ↓ initialize()
INITIALIZED
  ↓ load_model()
MODEL_LOADED
  ↓ execute()
EXECUTING
  ↓
RESULT_READY
  ↓ (repeat execute or unload)
MODEL_LOADED
  ↓ unload()
INITIALIZED
  ↓ shutdown()
SHUTDOWN
```

### 1.2 State Transitions

| From | To | Operation |
|---|---|---|
| UNINITIALIZED | INITIALIZED | `initialize(config)` |
| INITIALIZED | MODEL_LOADED | `load_model(ncl_path, revision_id)` |
| MODEL_LOADED | EXECUTING | `execute(input)` |
| EXECUTING | RESULT_READY | Execution complete |
| RESULT_READY | EXECUTING | Next input |
| RESULT_READY | MODEL_LOADED | `reset()` |
| MODEL_LOADED | INITIALIZED | `unload_model()` |
| INITIALIZED | SHUTDOWN | `shutdown()` |

---

## 2. Dynamic Execution Model

### 2.1 Execution Flow

```
Input
  ↓
Encode
  ↓
Query Routing
  ↓
Select Cells
  ↓
Execute Selected Cells
  ↓
Evaluate Result
  ├─→ sufficient → Output
  └─→ insufficient → Query Routing (recursive)
```

### 2.2 Execution Loop

```rust
fn execute(&mut self, input: &Input) -> Result<Output> {
    let mut context = ExecutionContext::new(input);
    let mut depth = 0;
    
    loop {
        // 1. Route
        let route_decision = self.router.route(
            &context,
            &self.compute_budget,
        )?;
        
        // 2. Select cells
        let selected_cells = self.select_cells(&route_decision)?;
        
        // 3. Load cells
        let loaded_cells = self.load_cells(&selected_cells)?;
        
        // 4. Execute cells
        let result = self.execute_cells(&loaded_cells, &context)?;
        
        // 5. Evaluate
        match self.evaluate(&result, &context)? {
            Evaluation::Sufficient(output) => return Ok(output),
            Evaluation::Insufficient(new_context) => {
                context = new_context;
                depth += 1;
                
                if depth >= self.compute_budget.max_depth {
                    return Err(NCIError::Budget);
                }
            }
        }
    }
}
```

---

## 3. Compute Budget

### 3.1 Budget Structure

```rust
struct ComputeBudget {
    max_depth: u32,
    max_cells: usize,
    max_bytes: u64,
    max_compute: u64,
    max_latency_us: u64,
}
```

### 3.2 Budget Enforcement

```rust
struct BudgetTracker {
    budget: ComputeBudget,
    cells_used: usize,
    bytes_used: u64,
    compute_used: u64,
    start_time: u64,
}

impl BudgetTracker {
    fn can_proceed(&self) -> Result<()> {
        if self.cells_used >= self.budget.max_cells {
            return Err(NCIError::Budget);
        }
        if self.bytes_used >= self.budget.max_bytes {
            return Err(NCIError::Budget);
        }
        if self.compute_used >= self.budget.max_compute {
            return Err(NCIError::Budget);
        }
        if self.elapsed_us() >= self.budget.max_latency_us {
            return Err(NCIError::Budget);
        }
        Ok(())
    }
    
    fn consume_cells(&mut self, count: usize) {
        self.cells_used += count;
    }
    
    fn consume_bytes(&mut self, bytes: u64) {
        self.bytes_used += bytes;
    }
    
    fn consume_compute(&mut self, flops: u64) {
        self.compute_used += flops;
    }
}
```

### 3.3 Budget Exhaustion

When budget is exhausted:
1. Return `NCI-E-BUDGET` error
2. OR return best-effort result with confidence metric
3. MUST NOT silently truncate

---

## 4. Backend Selection

### 4.1 Backend Discovery

```rust
fn discover_backends() -> Vec<BackendInfo> {
    let mut backends = Vec::new();
    
    if cpu::is_available() {
        backends.push(BackendInfo::Cpu);
    }
    if cuda::is_available() {
        backends.push(BackendInfo::Cuda);
    }
    if metal::is_available() {
        backends.push(BackendInfo::Metal);
    }
    if vulkan::is_available() {
        backends.push(BackendInfo::Vulkan);
    }
    if npu::is_available() {
        backends.push(BackendInfo::Npu);
    }
    
    backends
}
```

### 4.2 Backend Selection Algorithm

```rust
fn select_backend(
    cell: &Cell,
    backends: &[BackendInfo],
    policy: &ExecutionPolicy,
) -> Result<BackendType> {
    // 1. Check policy preference
    if let Some(preferred) = policy.preferred_backend {
        if backends.iter().any(|b| b.type == preferred) {
            return Ok(preferred);
        }
    }
    
    // 2. Check hardware capability
    for backend in backends {
        if backend.supports_dtype(cell.representation.dtype) {
            return Ok(backend.type);
        }
    }
    
    // 3. Fallback to CPU
    Ok(BackendType::Cpu)
}
```

### 4.3 Hardware Abstraction

```rust
trait Backend {
    fn execute(&self, cell: &Cell, input: &Tensor) -> Result<Tensor>;
    fn supports_dtype(&self, dtype: DataType) -> bool;
    fn memory_available(&self) -> u64;
    fn compute capability(&self) -> u64;
}
```

---

## 5. Cell Loading

### 5.1 Loading Strategy

```rust
fn load_cells(&self, cell_ids: &[CellId]) -> Result<Vec<LoadedCell>> {
    let mut loaded = Vec::new();
    
    for cell_id in cell_ids {
        // 1. Check cache
        if let Some(cached) = self.cache.get(cell_id) {
            loaded.push(cached);
            continue;
        }
        
        // 2. Load from NCL
        let cell = self.reader.read_cell(cell_id)?;
        
        // 3. Select representation
        let representation = self.select_representation(&cell)?;
        
        // 4. Decompress if needed
        let payload = self.decompress(&representation)?;
        
        // 5. Cache
        self.cache.insert(cell_id, LoadedCell::new(cell, payload));
        
        loaded.push(self.cache.get(cell_id).unwrap());
    }
    
    Ok(loaded)
}
```

### 5.2 Async Loading

```rust
async fn load_cells_async(&self, cell_ids: &[CellId]) -> Result<Vec<LoadedCell>> {
    let futures: Vec<_> = cell_ids.iter()
        .map(|id| self.load_cell_async(id))
        .collect();
    
    join_all(futures).await
}
```

---

## 6. Prefetch

### 6.1 Prefetch Strategy

```rust
fn prefetch(&self, current_cell: &CellId, depth: u32) {
    let dependencies = self.reader.get_dependencies(current_cell);
    
    for (i, dep_id) in dependencies.iter().enumerate() {
        let priority = if i < 3 { Priority::High } else { Priority::Medium };
        
        self.prefetch_engine.submit(PrefetchRequest {
            cell_id: *dep_id,
            priority,
            deadline: self.estimate_activation_time(dep_id),
        });
    }
}
```

---

## 7. Validation

1. Runtime MUST enforce compute budget.
2. Runtime MUST NOT silently truncate on budget exhaustion.
3. Backend selection MUST be deterministic.
4. Cell loading MUST use cache.
5. Prefetch MUST be asynchronous.

---

## 8. Error Handling

| Condition | Error Code |
|---|---|
| Budget exhausted | `NCI-E-BUDGET` |
| Backend unavailable | `NCI-E-HARDWARE` |
| Cell not found | `NCI-E-NOTFOUND` |
| Cell corrupt | `NCI-E-CORRUPT` |
| Routing failure | `NCI-E-ROUTING` |
| Execution timeout | `NCI-E-BUDGET` |

---

## 9. Conformance Requirements

1. Runtime MUST enforce compute budget.
2. Runtime MUST support dynamic execution with routing.
3. Runtime MUST select appropriate backend.
4. Runtime MUST use cache for cell loading.
5. Runtime MUST support async prefetch.
6. Runtime MUST NOT depend on external formats (NCI-P09).
