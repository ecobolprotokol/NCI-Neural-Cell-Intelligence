# NCL Transaction Protocol Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §21.6–§21.7; `ncl/12-NCL-WRITER-PROTOCOL.md`

---

## Purpose

This document specifies the NCL Transaction Protocol: transaction lifecycle, isolation levels, commit semantics, rollback behavior, concurrent transactions, and the transaction contract.

## Scope

Transaction lifecycle, ACID properties, isolation levels, commit protocol, rollback protocol, concurrent transaction support, and transaction integrity.

---

## 1. Transaction Model

### 1.1 Transaction Lifecycle

```
IDLE
  ↓ begin()
ACTIVE
  ├── write operations
  ↓ commit() or rollback()
COMMITTING
  ↓
COMMITTED
  ↓
IDLE

or

ACTIVE
  ↓ rollback()
ROLLED_BACK
  ↓
IDLE
```

### 1.2 Transaction Properties

| Property | Guarantee |
|---|---|
| Atomicity | Commit is all-or-nothing |
| Consistency | All data valid after commit |
| Isolation | Transactions do not interfere |
| Durability | Committed data survives crash (after fsync) |

---

## 2. Transaction States

### 2.1 State Machine

```rust
enum TxState {
    Idle,
    Active,
    Committing,
    Committed,
    RolledBack,
}
```

### 2.2 State Transitions

| From | To | Operation |
|---|---|---|
| Idle | Active | `begin()` |
| Active | Active | `write_*()` |
| Active | Committing | `commit()` |
| Active | RolledBack | `rollback()` |
| Committing | Committed | `finalize()` |
| Committed | Idle | `close()` |
| RolledBack | Idle | `close()` |

---

## 3. Isolation Levels

### 3.1 Read-Uncommitted (Default)

- Writers MAY read uncommitted data from other transactions
- Readers see committed data only
- Simplest implementation

### 3.2 Read-Committed (Recommended)

- Writers read only committed data
- Readers see committed data only
- No dirty reads

### 3.3 Repeatable-Read

- Within a transaction, repeated reads return same data
- Prevents non-repeatable reads

### 3.4 Implementation

For NCL, isolation is simplified because:
- Data is immutable once committed
- Only one writer at a time
- Readers access immutable revisions

```
Isolation is inherently provided by immutability:
- Committed data cannot change
- New transactions write to new locations
- No read-write conflicts
```

---

## 4. Commit Protocol

### 4.1 Two-Phase Commit

```
Phase 1: PREPARE
  - Validate all buffered data
  - Compute all hashes
  - Prepare all regions
  - Verify integrity

Phase 2: COMMIT
  - Write all regions to file
  - Write superblock
  - fsync
  - Mark committed
```

### 4.2 Commit Atom

```rust
fn commit(&mut self, tx: TransactionId) -> Result<()> {
    // Phase 1: PREPARE
    let prepared = self.prepare(tx)?;
    
    // Phase 2: COMMIT
    self.write_regions(&prepared)?;
    self.write_superblock(&prepared)?;
    self.file.sync_all()?;
    
    // Mark committed
    self.state = TxState::Committed;
    
    Ok(())
}
```

### 4.3 Commit Validation

During PREPARE:
1. All cells satisfy size constraints
2. All CellKinds are valid
3. All schemas are resolvable
4. All representations are resolvable
5. Dependency graph is acyclic
6. All payloads have valid checksums
7. Deduplication index is consistent
8. Column encodings are valid

If ANY validation fails → ROLLBACK

---

## 5. Rollback Protocol

### 5.1 Rollback Process

```
1. Discard all buffered data for this transaction
2. Release all locks held by this transaction
3. Close transaction
4. No partial data is visible
5. No file modifications occur
```

### 5.2 Rollback Invariants

- Rollback MUST NOT modify output file
- Rollback MUST release all resources
- Rollback MUST be idempotent (multiple rollbacks safe)
- Rollback after commit is a no-op

---

## 6. Concurrent Transactions

### 6.1 Model

```
Transaction T1: [begin]---[write]---[write]---[commit]
Transaction T2:       [begin]---[write]---[commit]
Transaction T3:             [begin]---[rollback]
```

### 6.2 Rules

1. Multiple transactions MAY be active concurrently
2. Only ONE commit MAY be active at a time
3. Transactions do not interfere (write to different buffers)
4. Final file is the result of the last committed transaction

### 6.3 Commit Serialization

```
fn commit_serialized(&mut self, tx: TransactionId) -> Result<()> {
    // Acquire commit lock
    let _guard = self.commit_lock.lock();
    
    // Only one commit at a time
    self.commit(tx)
}
```

---

## 7. Transaction Integrity

### 7.1 Pre-Commit Integrity

All data written during a transaction MUST be verified before commit:

```
for cell in transaction.cells() {
    verify_cell_integrity(cell)?;
}

for block in transaction.blocks() {
    verify_block_checksum(block)?;
}

for column in transaction.columns() {
    verify_column_checksum(column)?;
}
```

### 7.2 Post-Commit Integrity

After commit, the file MUST be fully valid:
- Superblock hash is correct
- Manifest hash is correct
- All regions are properly written
- All indexes are consistent

---

## 8. Transaction Timeout

### 8.1 Timeout Policy

Transactions SHOULD have a maximum duration:
- Default: 30 minutes
- Configurable by operator
- On timeout: automatic rollback

### 8.2 Timeout Detection

```
fn check_timeout(&self, tx: &Transaction) -> bool {
    let elapsed = now() - tx.start_time;
    elapsed > self.config.tx_timeout
}
```

### 8.3 Timeout Response

1. Log timeout event
2. Automatic rollback
3. Release all resources
4. Return `NCI-E-LOCK` or specific timeout error

---

## 9. Transaction Recovery

### 9.1 Crash Recovery

If crash occurs during commit:
1. Check if superblock is valid
2. If valid → committed (use new data)
3. If invalid → not committed (use old data or backup)
4. Log recovery event

### 9.2 Partial Write Recovery

If commit fails midway:
1. Old file is preserved (backup strategy)
2. New file is incomplete → discard
3. Operator decides recovery strategy

---

## 10. Validation

1. Transaction state machine MUST be followed.
2. Commit MUST be atomic.
3. Rollback MUST NOT modify output file.
4. Concurrent commits MUST be serialized.
5. Transaction timeout MUST be enforced.
6. Crash recovery MUST be deterministic.

---

## 11. Error Handling

| Condition | Error Code | Action |
|---|---|---|
| Transaction not active | `NCI-E-FORMAT` | Reject operation |
| Commit validation fail | `NCI-E-FORMAT` | Rollback |
| Commit write fail | `NCI-E-STORE` | Rollback |
| Commit lock timeout | `NCI-E-LOCK` | Wait or abort |
| Transaction timeout | `NCI-E-LOCK` | Auto-rollback |
| Rollback failure | `NCI-E-STORE` | Log, force cleanup |

---

## 12. Conformance Requirements

1. Transaction MUST follow state machine in §2.
2. Commit MUST be atomic (all-or-nothing).
3. Rollback MUST NOT modify output file.
4. Concurrent commits MUST be serialized.
5. Transaction timeout MUST be implemented.
6. Crash recovery MUST be deterministic.
7. Two independently implemented transaction managers MUST interoperate.
