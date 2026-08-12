# NCL Reader Protocol Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §21.1–§21.3; `ncl/01-NCL-BINARY-SPEC.md`

---

## Purpose

This document specifies the NCL Reader: state machine, API contract, selective access guarantee, concurrency model, and the complete reader lifecycle.

## Scope

Reader state machine, reader API, selective access, concurrency, error handling, and the reader contract that enables independent implementation by separate teams.

---

## 1. Reader State Machine

### 1.1 States

```
CLOSED
  ↓ open()
OPENED
  ↓ validate_superblock()
VALIDATED
  ↓ load_index()
READY
  ↓ resolve/read operations
ACTIVE
  ↓ close()
CLOSED
```

### 1.2 State Transitions

| From | To | Operation | Guard |
|---|---|---|---|
| CLOSED | OPENED | `open(path)` | File exists, readable |
| OPENED | VALIDATED | `validate_superblock()` | Magic, version, structure valid |
| VALIDATED | READY | `load_index()` | Index hash valid |
| READY | ACTIVE | `read_*()` operations | None |
| ACTIVE | READY | Return from operation | None |
| READY | CLOSED | `close()` | None |
| ACTIVE | CLOSED | `close()` | None |
| Any | CLOSED | `drop()` | Automatic cleanup |

### 1.3 Invalid Transitions

Any transition not listed above is invalid and MUST produce `NCI-E-FORMAT`.

---

## 2. Reader API

### 2.1 Core Trait

```rust
trait NclReader {
    fn open(path: &Path) -> Result<Self>;
    fn manifest(&self) -> Result<&Manifest>;
    fn resolve_cell(&self, id: &CellId) -> Result<CellRef>;
    fn read_cell(&self, id: &CellId) -> Result<Cell>;
    fn read_column(&self, id: &ColumnId) -> Result<ColumnReader>;
    fn read_block(&self, id: &BlockId) -> Result<BlockReader>;
    fn read_column_predicate(
        &self,
        columns: &[ColumnId],
    ) -> Result<SelectiveColumnReader>;
    fn close(&mut self) -> Result<()>;
}
```

### 2.2 Async API

```rust
trait NclReaderAsync {
    async fn open_async(path: &Path) -> Result<Self>;
    async fn read_cell_async(&self, id: &CellId) -> Result<Cell>;
    async fn read_cells_batch_async(
        &self,
        ids: &[CellId],
    ) -> Result<Vec<Cell>>;
    async fn read_block_async(&self, id: &BlockId) -> Result<BlockReader>;
}
```

### 2.3 Return Types

```rust
struct CellRef {
    cell_id: CellId,
    kind: CellKind,
    schema: SchemaId,
    representation: RepresentationId,
    payload_ref: PayloadRef,
    dependencies: Vec<CellId>,
    attributes: AttributeSet,
}

struct ColumnReader {
    column_id: ColumnId,
    column_type: u16,
    encoding: u8,
    element_count: u64,
    data: Bytes,
}

struct BlockReader {
    block_id: BlockId,
    uncompressed_size: u64,
    data: Bytes,
}

struct SelectiveColumnReader {
    columns: Vec<ColumnReader>,
    row_range: Range<u64>,
}
```

---

## 3. Selective Access Guarantee

### 3.1 Principle

Reader MUST NOT load entire file for single Cell read:

```
FORBIDDEN:
request: read_cell(C042)
    ↓
load: entire model (1 TB) into RAM

REQUIRED:
request: read_cell(C042)
    ↓
index lookup: find C042 metadata
    ↓
locate: payload block
    ↓
read: only required block
    ↓
verify: BLAKE3-256
    ↓
decompress: if needed
    ↓
return: Cell C042
```

### 3.2 Implementation

```rust
fn read_cell(&self, id: &CellId) -> Result<Cell> {
    // 1. Index lookup (O(1) amortized)
    let cell_ref = self.cell_index.get(id)
        .ok_or(NCIError::NotFound)?;
    
    // 2. Read payload block (selective I/O)
    let block = self.read_block(cell_ref.payload_ref.block_id)?;
    
    // 3. Verify block checksum
    block.verify_checksum()?;
    
    // 4. Decompress if needed
    let payload = if block.is_compressed() {
        block.decompress()?
    } else {
        block.data
    };
    
    // 5. Verify Cell hash
    let computed_hash = blake3_256(&payload);
    if computed_hash != *id {
        return Err(NCIError::Corrupt);
    }
    
    // 6. Reconstruct Cell from columns + payload
    Ok(Cell::from_parts(cell_ref, payload))
}
```

### 3.3 Selective Column Access

```rust
fn read_column_predicate(
    &self,
    columns: &[ColumnId],
) -> Result<SelectiveColumnReader> {
    // 1. Locate column entries in Column Region
    let mut readers = Vec::new();
    
    for col_id in columns {
        let entry = self.column_index.get(col_id)
            .ok_or(NCIError::NotFound)?;
        
        // 2. Read only this column's data
        let data = self.read_column_data(entry)?;
        
        readers.push(ColumnReader {
            column_id: *col_id,
            column_type: entry.column_type,
            encoding: entry.encoding,
            element_count: entry.element_count,
            data,
        });
    }
    
    Ok(SelectiveColumnReader {
        columns: readers,
        row_range: 0..self.manifest.cell_count,
    })
}
```

---

## 4. Concurrency Model

### 4.1 Multiple Readers

- Multiple reader instances MAY access the same NCL file concurrently
- Each reader maintains independent state
- Readers access immutable data (no locking required for reads)
- Readers MAY share index data (memory-mapped or cached)

### 4.2 Reader-Writer Isolation

- Readers accessing revision R1 are unaffected by writer creating R2
- Writer creates new revision without modifying existing one
- No cross-revision interference

### 4.3 Thread Safety

```rust
// Reader MUST be Send + Sync
unsafe impl Send for NclReader {}
unsafe impl Sync for NclReader {}
```

Multiple threads MAY read concurrently from same reader instance.

---

## 5. Lazy Loading

### 5.1 Principle

Reader MAY defer loading until data is accessed:
- Superblock: loaded on open()
- Manifest: loaded on first manifest() call
- Index: loaded on first lookup
- Columns: loaded on first read_column()
- Blocks: loaded on first read_block()

### 5.2 Implementation

```rust
struct NclReader {
    file: File,
    superblock: Option<Superblock>,      // Loaded on open
    manifest: Option<Manifest>,           // Lazy
    index: Option<IndexRegion>,           // Lazy
    column_cache: HashMap<ColumnId, ColumnReader>,
    block_cache: HashMap<BlockId, BlockReader>,
}
```

---

## 6. Caching

### 6.1 Cache Strategy

Reader SHOULD implement LRU cache for:
- Frequently accessed Cells
- Frequently accessed Blocks
- Index entries

### 6.2 Cache Key

Cache key is `(CellId, RepresentationId)` or `(BlockId)`.

### 6.3 Cache Invalidation

Cache is invalidated when:
- Reader is closed
- File is modified (detected via file metadata)
- Explicit cache_clear() call

---

## 7. Error Handling

### 7.1 State-Dependent Errors

| State | Error Condition | Error Code |
|---|---|---|
| CLOSED | open() fails | `NCI-E-STORE` |
| OPENED | Magic mismatch | `NCI-E-FORMAT` |
| OPENED | Version mismatch | `NCI-E-SCHEMA` |
| VALIDATED | Index hash mismatch | `NCI-E-CORRUPT` |
| READY/ACTIVE | Cell not found | `NCI-E-NOTFOUND` |
| READY/ACTIVE | Block checksum mismatch | `NCI-E-CORRUPT` |
| READY/ACTIVE | Cell hash mismatch | `NCI-E-CORRUPT` |
| READY/ACTIVE | Decompression failure | `NCI-E-CORRUPT` |
| Any | Lock timeout | `NCI-E-LOCK` |

### 7.2 Error Recovery

Reader MUST NOT attempt automatic repair on corruption. Error response:
1. Return error to caller
2. Log error with details
3. Mark affected resources as unavailable
4. Allow caller to decide recovery strategy

---

## 8. File Monitoring

### 8.1 File Change Detection

Reader SHOULD detect file changes via:
- File modification timestamp
- File size change
- File metadata hash

### 8.2 On Change Detection

When file change is detected:
1. Invalidate all cached data
2. Re-read superblock
3. Re-validate
4. Re-load index

---

## 9. Validation

1. State machine MUST be followed exactly.
2. Selective access MUST NOT load entire file.
3. Concurrency MUST be thread-safe.
4. All errors MUST be explicit.
5. Corruption MUST NOT cause silent failure.

---

## 10. Conformance Requirements

1. Reader MUST implement the state machine in §1.
2. Reader MUST support selective Cell and Column access.
3. Reader MUST NOT require full-file loading for any single read.
4. Reader MUST be thread-safe.
5. Reader MUST verify all integrity checks before data release.
6. Reader MUST support async API.
7. Reader MUST handle corruption with quarantine semantics.
8. Two independently implemented readers MUST interoperate on same NCL file.
