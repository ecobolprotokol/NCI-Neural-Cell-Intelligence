# NCL Writer Protocol Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §21.4–§21.6; `ncl/01-NCL-BINARY-SPEC.md`

---

## Purpose

This document specifies the NCL Writer: state machine, API contract, streaming guarantee, atomic commit semantics, and the complete writer lifecycle.

## Scope

Writer state machine, writer API, streaming input, transaction support, atomic commit, concurrent access, and the writer contract that enables independent implementation by separate teams.

---

## 1. Writer State Machine

### 1.1 States

```
CLOSED
  ↓ create()
CREATED
  ↓ begin_transaction()
WRITING
  ├── write_cell()
  ├── write_column()
  ├── write_payload()
  └── ...
  ↓ commit() or rollback()
VALIDATING
  ↓
FINALIZING
  ↓
COMMITTING
  ↓
COMMITTED
```

### 1.2 State Transitions

| From | To | Operation | Guard |
|---|---|---|---|
| CLOSED | CREATED | `create(path)` | Path writable |
| CREATED | WRITING | `begin_transaction()` | None |
| WRITING | WRITING | `write_*()` operations | Transaction active |
| WRITING | VALIDATING | `commit()` | Transaction active |
| WRITING | CLOSED | `rollback()` | Transaction active |
| VALIDATING | FINALIZING | Validation pass | No errors |
| VALIDATING | WRITING | Validation fail | Errors fixable |
| VALIDATING | CLOSED | Validation fail | Errors unfixable |
| FINALIZING | COMMITTING | Write superblock | None |
| COMMITTING | COMMITTED | fsync + close | None |
| COMMITTED | CLOSED | Close | None |

### 1.3 Invalid Transitions

Any transition not listed above is invalid and MUST produce `NCI-E-FORMAT`.

---

## 2. Writer API

### 2.1 Core Trait

```rust
trait NclWriter {
    fn create(path: &Path) -> Result<Self>;
    fn begin_transaction(&mut self) -> Result<TransactionId>;
    fn write_cell(&mut self, tx: TransactionId, cell: &CellPlan) -> Result<()>;
    fn write_column(&mut self, tx: TransactionId, column: &ColumnData) -> Result<()>;
    fn write_payload(&mut self, tx: TransactionId, block: &PayloadBlock) -> Result<()>;
    fn write_tensor(&mut self, tx: TransactionId, tensor: &TensorWrite) -> Result<()>;
    fn commit(&mut self, tx: TransactionId) -> Result<()>;
    fn rollback(&mut self, tx: TransactionId) -> Result<()>;
    fn finalize(&mut self) -> Result<()>;
    fn close(&mut self) -> Result<()>;
}
```

### 2.2 Streaming API

```rust
trait NclWriterStreaming {
    fn write_cell_streaming(
        &mut self,
        tx: TransactionId,
        cell_stream: impl Iterator<Item = CellPlan>,
    ) -> Result<()>;
    
    fn write_tensor_streaming(
        &mut self,
        tx: TransactionId,
        tensor_stream: impl Iterator<Item = TensorWrite>,
    ) -> Result<()>;
}
```

### 2.3 Return Types

```rust
struct TransactionId(u64);

struct CellPlan {
    kind: CellKind,
    schema: SchemaId,
    representation: RepresentationId,
    payload: Vec<u8>,
    dependencies: Vec<CellId>,
    attributes: AttributeSet,
}

struct ColumnData {
    column_type: u16,
    encoding: u8,
    elements: Vec<Vec<u8>>,
}

struct PayloadBlock {
    block_id: u32,
    codec: u8,
    data: Vec<u8>,
}

struct TensorWrite {
    descriptor: TensorDescriptor,
    data: Bytes,
}
```

---

## 3. Streaming Guarantee

### 3.1 Principle

Writer MUST accept streaming input without full model residency:

```
while let Some(tensor) = source.next()? {
    let cells = planner.plan(tensor)?;
    for cell in cells {
        writer.write_cell(tx, &cell)?;
    }
}
writer.commit(tx)?;
```

No intermediate buffer MUST hold entire model.

### 3.2 Implementation

```rust
fn write_tensor_streaming(
    &mut self,
    tx: TransactionId,
    mut tensor_stream: impl Iterator<Item = TensorWrite>,
) -> Result<()> {
    while let Some(tensor) = tensor_stream.next() {
        // Plan cells from tensor
        let cells = self.planner.plan(&tensor)?;
        
        for cell in cells {
            // Write cell without buffering entire model
            self.write_cell(tx, &cell)?;
        }
        
        // Flush intermediate buffers periodically
        if self.buffer_size > FLUSH_THRESHOLD {
            self.flush()?;
        }
    }
    
    Ok(())
}
```

### 3.3 Buffer Limits

| Buffer | Maximum Size | Flush Trigger |
|---|---|---|
| Cell buffer | 64 MiB | When exceeded |
| Payload buffer | 64 MiB | When exceeded |
| Column buffer | 64 MiB | When exceeded |
| Index buffer | 64 MiB | When exceeded |

---

## 4. Atomic Commit

### 4.1 Principle

Commit MUST be atomic: file is either completely valid or unchanged.

### 4.2 Commit Protocol

```
1. VALIDATING
   - Verify all cells
   - Verify all columns
   - Verify all blocks
   - Compute manifest hash
   - Compute root hash

2. FINALIZING
   - Write manifest region
   - Write index region
   - Write revision region
   - Write provenance region
   - Write integrity region

3. COMMITTING
   - Write superblock (last)
   - fsync file
   - Close file

4. COMMITTED
   - File is valid
```

### 4.3 Crash Safety

If crash occurs during commit:
- Before superblock write: file is invalid (previous state preserved if backup exists)
- After superblock write + fsync: file is valid
- Partial superblock write: detected by magic/version validation

### 4.4 Backup Strategy

Writer SHOULD support backup:
```
1. Write new file to temp path
2. Commit to temp path
3. Rename original to backup
4. Rename temp to original
```

---

## 5. Transaction Support

### 5.1 Transaction Model

```
BEGIN (begin_transaction)
  ├── write_cell
  ├── write_column
  ├── write_payload
  └── ...
VALIDATE (internal)
  ├── verify integrity
  └── check deduplication
COMMIT or ROLLBACK
```

### 5.2 Transaction Isolation

- Each transaction is independent
- Multiple transactions MAY be active (for different data)
- Only ONE commit MAY be active at a time
- Partial commits are forbidden

### 5.3 Rollback

On rollback:
1. Discard all buffered data for this transaction
2. Do NOT write to output file
3. Transaction is closed
4. No partial data is visible

---

## 6. Deduplication

### 6.1 Writer Deduplication

Writer MUST check deduplication index before writing payload:

```
fn write_payload(&mut self, tx: TransactionId, block: PayloadBlock) -> Result<()> {
    let payload_hash = blake3_256(&block.data);
    
    if let Some(existing_block_id) = self.dedup_index.get(&payload_hash) {
        // Payload already exists, reference existing block
        self.reference_existing_block(tx, existing_block_id);
    } else {
        // New payload, write and index
        let block_id = self.write_new_block(tx, block)?;
        self.dedup_index.insert(payload_hash, block_id);
    }
    
    Ok(())
}
```

### 6.2 Deduplication Index

Writer MUST maintain in-memory deduplication index:
- Key: BLAKE3-256 of uncompressed payload
- Value: block_id
- Updated on each write
- Persisted in Index Region

---

## 7. Concurrent Access

### 7.1 Single Writer

- Only ONE writer MAY be active on a given NCL file at any time
- Writer acquires exclusive lock on file
- Lock is released on close()

### 7.2 Reader-Writer

- Readers access existing revisions (immutable)
- Writer creates new revision
- No interference between reader and writer
- Reader reading R1 is unaffected by writer creating R2

### 7.3 Lock Protocol

```rust
fn acquire_write_lock(path: &Path) -> Result<FileLock> {
    // 1. Try to create .lock file
    // 2. If exists, check if stale (PID file or timestamp)
    // 3. If stale, break lock
    // 4. If not stale, return NCI-E-LOCK
    // 5. Write PID/timestamp to .lock file
    // 6. Return lock handle
}
```

---

## 8. Superblock Writing

### 8.1 Superblock is LAST

The Superblock MUST be written last during commit:

```
1. Write all regions
2. Compute region offsets and sizes
3. Populate Superblock fields
4. Compute manifest_hash and root_hash
5. Write Superblock (4096 bytes)
6. fsync
```

### 8.2 Superblock Atomicity

Superblock write is a single 4096-byte write. On most filesystems, this is atomic for sector-aligned writes (512 bytes or 4096 bytes). NCL Superblock is 4096 bytes, matching common sector size.

---

## 9. Validation

### 9.1 Pre-Commit Validation

Writer MUST validate:
1. All cells are within size bounds
2. All CellKinds are valid
3. All schemas are resolvable
4. All representations are resolvable
5. All dependencies are acyclic
6. All payloads have valid checksums
7. Deduplication index is consistent
8. Column encodings are valid

### 9.2 Post-Commit Verification

Writer SHOULD verify:
1. Re-read superblock, validate
2. Spot-check random cells
3. Verify manifest hash

---

## 10. Error Handling

| Condition | Error Code | Action |
|---|---|---|
| File not writable | `NCI-E-STORE` | Abort |
| Transaction not active | `NCI-E-FORMAT` | Reject operation |
| Cell size out of bounds | `NCI-E-FORMAT` | Reject cell |
| Duplicate commit | `NCI-E-FORMAT` | Reject |
| Commit validation fail | `NCI-E-FORMAT` | Rollback |
| Write failure | `NCI-E-STORE` | Rollback |
| Lock acquisition fail | `NCI-E-LOCK` | Wait or abort |

---

## 11. Conformance Requirements

1. Writer MUST implement the state machine in §1.
2. Writer MUST support streaming input without full model residency.
3. Writer MUST provide atomic commit semantics.
4. Writer MUST implement deduplication.
5. Writer MUST enforce single-writer access.
6. Writer MUST write Superblock last.
7. Writer MUST verify all data before commit.
8. Two independently implemented writers MUST produce interoperable NCL files.
