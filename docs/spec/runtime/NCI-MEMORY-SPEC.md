# NCI Memory Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §15; `runtime/NCI-RUNTIME-SPEC.md`

---

## Purpose

This document specifies the NCI Memory Architecture: memory as Cell, working memory bounds, persistent memory, consolidation, and cross-session persistence.

## Scope

Memory types, working memory bounds, persistent memory, consolidation, and memory lifecycle.

---

## 1. Memory as Cell

### 1.1 Memory Types

```
Memory
    ├── Working Memory (current session state)
    ├── Episodic Memory (specific events/examples)
    ├── Semantic Memory (facts, relationships)
    ├── Procedural Memory (learned patterns)
    └── Consolidated Memory (long-term learned state)
```

### 1.2 Memory Cell Structure

Each memory type is represented as Cell(s):

```rust
struct MemoryCell {
    cell_id: CellId,
    memory_type: MemoryType,
    importance: f16,
    timestamp: u64,
    data: Vec<u8>,
}

enum MemoryType {
    Working = 0x01,
    Episodic = 0x02,
    Semantic = 0x03,
    Procedural = 0x04,
    Consolidated = 0x05,
}
```

---

## 2. Working Memory

### 2.1 Bounds

Working memory MUST be bounded:

```
Working Memory Capacity = MAX(fixed_limit, K × sqrt(seq_len))
```

Where:
- `fixed_limit`: Minimum capacity (e.g., 1024 entries)
- `K`: Tuning constant (e.g., 128)
- `seq_len`: Current sequence length

### 2.2 Working Memory Content

Working memory MUST contain:
- Active computation state
- Selected relevant history
- Confidence metrics
- Intermediate results only

Working memory MUST NOT be:
- Dump of entire context
- Unbounded history
- Raw input data

### 2.3 Working Memory Management

```rust
struct WorkingMemory {
    capacity: usize,
    entries: VecDeque<MemoryEntry>,
    total_size: u64,
}

impl WorkingMemory {
    fn insert(&mut self, entry: MemoryEntry) {
        // Check capacity
        while self.entries.len() >= self.capacity ||
              self.total_size + entry.size() > self.max_bytes {
            if let Some(oldest) = self.entries.pop_front() {
                self.total_size -= oldest.size();
            }
        }
        
        self.total_size += entry.size();
        self.entries.push_back(entry);
    }
    
    fn evict_by_importance(&mut self) {
        // Sort by importance, remove least important
        self.entries.make_contiguous().sort_by(|a, b| {
            a.importance.partial_cmp(&b.importance).unwrap()
        });
        
        while self.entries.len() > self.capacity / 2 {
            if let Some(least_important) = self.entries.pop_front() {
                self.total_size -= least_important.size();
            }
        }
    }
}
```

---

## 3. Persistent Memory

### 3.1 Persistence Model

Memory persists across sessions:

```
Session 1
    ↓
Memory Write
    ↓
Memory Cell (committed to NCL)
    ↓
New Revision
    ↓
Session 2 (Memory Cell accessible)
```

### 3.2 Persistence Process

```rust
fn persist_memory(
    &self,
    memory: &MemoryCell,
    writer: &mut NclWriter,
) -> Result<()> {
    // 1. Create Memory Cell
    let cell_plan = CellPlan {
        kind: CellKind::Memory,
        representation: RepresentationId::new(0x01),
        payload: memory.serialize()?,
        attributes: AttributeSet::from([
            ("nci.memory_type", memory.memory_type as u64),
            ("nci.importance", memory.importance as u64),
            ("nci.timestamp", memory.timestamp),
        ]),
        ..Default::default()
    };
    
    // 2. Compute identity
    let cell_id = blake3_256(&cell_plan.canonical_bytes());
    
    // 3. Write to NCL
    let tx = writer.begin_transaction()?;
    writer.write_cell(tx, &cell_id, &cell_plan)?;
    writer.commit(tx)?;
    
    Ok(())
}
```

### 3.3 Cross-Session Access

```rust
fn load_persistent_memory(
    &self,
    reader: &NclReader,
    memory_type: MemoryType,
) -> Result<Vec<MemoryCell>> {
    // 1. Query Memory Column
    let memory_cells = reader.query_column(
        ColumnId::MemoryType,
        &memory_type as u8,
    )?;
    
    // 2. Load each memory cell
    let mut memories = Vec::new();
    for cell_ref in memory_cells {
        let cell = reader.read_cell(&cell_ref.cell_id)?;
        memories.push(MemoryCell::deserialize(&cell.payload)?);
    }
    
    Ok(memories)
}
```

---

## 4. Memory Consolidation

### 4.1 Consolidation Process

```
Fragmented Memory State
    ↓
Consolidation
    ↓
Optimized Memory Cell
    ↓
New Revision
```

### 4.2 Consolidation Algorithm

```rust
fn consolidate(
    &self,
    memories: &[MemoryCell],
) -> Result<MemoryCell> {
    // 1. Group by similarity
    let clusters = self.cluster_memories(memories)?;
    
    // 2. Merge each cluster
    let consolidated = Vec::new();
    for cluster in clusters {
        let merged = self.merge_cluster(&cluster)?;
        consolidated.push(merged);
    }
    
    // 3. Create consolidated memory cell
    let consolidated_cell = MemoryCell {
        cell_id: CellId::random(),
        memory_type: MemoryType::Consolidated,
        importance: self.compute_importance(&consolidated),
        timestamp: now(),
        data: self.serialize_consolidated(&consolidated)?,
    };
    
    Ok(consolidated_cell)
}
```

### 4.3 Consolidation Invariants

1. Consolidation MUST NOT lose information.
2. Consolidation MUST preserve semantics.
3. Consolidation MUST produce new revision.
4. Original memories MUST be retained until GC.

---

## 5. Memory Lifecycle

### 5.1 States

```
CREATED → ACTIVE → CONSOLIDATED → GARBAGE_COLLECTED
```

### 5.2 State Transitions

| From | To | Condition |
|---|---|---|
| CREATED | ACTIVE | First access |
| ACTIVE | CONSOLIDATED | Consolidation run |
| CONSOLIDATED | GARBAGE_COLLECTED | GC policy |

---

## 6. Validation

1. Working memory MUST respect capacity bounds.
2. Persistent memory MUST be committed to NCL.
3. Consolidation MUST NOT lose information.
4. Memory lifecycle MUST be followed.

---

## 7. Error Handling

| Condition | Error Code |
|---|---|
| Working memory overflow | N/A (evict) |
| Persistence failure | `NCI-E-STORE` |
| Consolidation failure | `NCI-E-STORE` |
| Memory not found | `NCI-E-NOTFOUND` |

---

## 8. Conformance Requirements

1. Working memory MUST be bounded.
2. Persistent memory MUST be committed to NCL.
3. Consolidation MUST NOT lose information.
4. Memory MUST persist across sessions.
5. Memory consolidation MUST produce new revision.
