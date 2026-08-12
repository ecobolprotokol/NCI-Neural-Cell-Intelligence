# NCI Data Movement Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §16.4; `runtime/NCI-RUNTIME-SPEC.md`

---

## Purpose

This document specifies NCI Data Movement: the primary optimization metric, data movement minimization, prefetch strategies, and bandwidth efficiency.

## Scope

Data movement metric, prefetch protocol, data staging, bandwidth optimization, and memory-mapped I/O.

---

## 1. Primary Optimization Metric

### 1.1 Definition

```
bytes_moved_per_useful_compute = total_bytes_transferred / useful_floating_point_operations
```

### 1.2 Invariants

- This metric MUST drive all architectural decisions
- Lower is better
- NOT FLOPs/second (which ignores data movement)
- NOT memory usage (which ignores computation)

### 1.3 Measurement

```rust
struct DataMovementMetrics {
    bytes_from_storage: u64,
    bytes_from_l3: u64,
    bytes_from_l2: u64,
    bytes_from_l1: u64,
    bytes_from_registers: u64,
    useful_flops: u64,
}

impl DataMovementMetrics {
    fn bytes_per_useful_flop(&self) -> f64 {
        let total_bytes = self.bytes_from_storage + self.bytes_from_l3 +
            self.bytes_from_l2 + self.bytes_from_l1 + self.bytes_from_registers;
        
        if self.useful_flops == 0 {
            return f64::INFINITY;
        }
        
        total_bytes as f64 / self.useful_flops as f64
    }
}
```

---

## 2. Data Staging

### 2.1 Staging Hierarchy

```
NCL Storage (NVMe/SSD)
    ↓ (Block I/O)
RAM
    ↓ (Page I/O)
VRAM/GPU Memory
    ↓ (Register I/O)
Execution Engine (CPU/GPU/NPU registers)
```

### 2.2 Staging Strategy

```rust
fn stage_data(
    cell_id: &CellId,
    source: &StorageLayer,
    target: &ComputeLayer,
) -> Result<()> {
    // 1. Read block from storage
    let block = source.read_block(cell_id)?;
    
    // 2. Decompress if needed
    let data = if block.is_compressed() {
        block.decompress()?
    } else {
        block.data
    };
    
    // 3. Stage to RAM
    let ram_buffer = stage_to_ram(&data)?;
    
    // 4. Stage to VRAM (if GPU)
    if target.is_gpu() {
        let vram_buffer = stage_to_vram(&ram_buffer)?;
        target.load_to_registers(&vram_buffer)?;
    } else {
        target.load_to_registers(&ram_buffer)?;
    }
    
    Ok(())
}
```

---

## 3. Prefetch Protocol

### 3.1 Dependency-Based Prefetch

```rust
fn prefetch(
    &self,
    current_cell: &CellId,
    prefetch_engine: &mut PrefetchEngine,
) {
    // 1. Get dependencies
    let deps = self.reader.get_dependencies(current_cell);
    
    // 2. Prioritize
    for (i, dep_id) in deps.iter().enumerate() {
        let priority = match i {
            0..=2 => Priority::High,
            3..=5 => Priority::Medium,
            _ => Priority::Low,
        };
        
        // 3. Submit prefetch
        prefetch_engine.submit(PrefetchRequest {
            cell_id: *dep_id,
            priority,
            deadline: self.estimate_activation_time(dep_id),
        });
    }
}
```

### 3.2 Predictive Prefetch

```rust
fn predictive_prefetch(
    &self,
    context: &ExecutionContext,
    prefetch_engine: &mut PrefetchEngine,
) {
    // Use routing predictions to prefetch likely next cells
    let predictions = self.router.predict_next_cells(context);
    
    for prediction in predictions {
        prefetch_engine.submit(PrefetchRequest {
            cell_id: prediction.cell_id,
            priority: prediction.priority,
            deadline: prediction.estimated_time,
        });
    }
}
```

### 3.3 Prefetch Execution

```rust
async fn execute_prefetch(&self, request: PrefetchRequest) -> Result<()> {
    // 1. Check if already cached
    if self.cache.contains(&request.cell_id) {
        return Ok(());
    }
    
    // 2. Read from storage
    let cell = self.reader.read_cell_async(&request.cell_id).await?;
    
    // 3. Stage to appropriate cache level
    let key = CacheKey::new(request.cell_id, cell.representation);
    self.cache.insert(key, CachedCell::new(cell));
    
    Ok(())
}
```

---

## 4. Bandwidth Optimization

### 4.1 Coalesced Access

For GPU backends, data access MUST be coalesced:
- Adjacent threads read adjacent memory locations
- Minimum transaction size: 32 bytes (warp level)
- Alignment: 128 bytes (cache line)

### 4.2 Vectorized Access

For CPU backends, data access SHOULD be vectorized:
- SIMD operations (SSE, AVX, AVX-512)
- Alignment: 32 bytes (AVX) or 64 bytes (AVX-512)

### 4.3 Access Pattern Optimization

```rust
fn optimize_access_pattern(
    cells: &[CellId],
    backend: &BackendType,
) -> Vec<CellId> {
    match backend {
        BackendType::Gpu => {
            // Sort by memory address for coalesced access
            let mut sorted = cells.to_vec();
            sorted.sort_by_key(|id| self.get_memory_address(id));
            sorted
        }
        BackendType::Cpu => {
            // Sort by cache line alignment
            let mut sorted = cells.to_vec();
            sorted.sort_by_key(|id| self.get_cache_line(id));
            sorted
        }
        _ => cells.to_vec(),
    }
}
```

---

## 5. Memory-Mapped I/O

### 5.1 mmap for Large Files

For files > 1 GB, use memory-mapped I/O:

```rust
fn mmap_file(path: &Path) -> Result<MmapRegion> {
    let file = File::open(path)?;
    let mmap = unsafe { Mmap::map(&file)? };
    Ok(MmapRegion::new(mmap))
}
```

### 5.2 Page Fault-Based Loading

With mmap, data is loaded on page fault:
- No explicit read syscall
- OS manages page cache
- Lazy loading inherent

---

## 6. Validation

1. Data movement metric MUST be computed.
2. Prefetch MUST be asynchronous.
3. Coalesced access MUST be used for GPU.
4. Memory-mapped I/O MUST be used for large files.

---

## 7. Error Handling

| Condition | Error Code |
|---|---|
| Storage read failure | `NCI-E-STORE` |
| Prefetch failure | `NCI-E-STORE` |
| Memory mapping failure | `NCI-E-STORE` |

---

## 8. Conformance Requirements

1. Primary optimization metric MUST be bytes_moved_per_useful_compute.
2. Prefetch MUST be dependency-based.
3. GPU access MUST be coalesced.
4. CPU access SHOULD be vectorized.
5. Large files SHOULD use memory-mapped I/O.
