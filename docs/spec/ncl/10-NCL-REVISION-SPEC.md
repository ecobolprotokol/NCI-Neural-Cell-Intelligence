# NCL Revision Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §19; `ncl/01-NCL-BINARY-SPEC.md`

---

## Purpose

This document specifies NCL Revision: immutable revision snapshots, DAG structure, revision resolution, incremental evolution, merge semantics, and garbage collection.

## Scope

Revision structure, revision DAG, revision resolution, incremental Cell sharing, merge conflict resolution, garbage collection, and revision integrity.

---

## 1. Revision Model

### 1.1 Revision Structure

```rust
#[repr(C, packed)]
struct Revision {
    revision_id: [u8; 32],        // offset 0, size 32 — BLAKE3-256
    parent_count: u32,            // offset 32, size 4
    // parent_ids: [[u8; 32]; parent_count] follows
    cell_count: u64,              // follows parents
    // cell_ids: [[u8; 32]; cell_count] follows
    timestamp: u64,               // follows cell_ids — milliseconds since epoch
    message_len: u32,             // follows timestamp
    // message: [u8; message_len] follows — UTF-8 commit message
}
```

### 1.2 Revision ID Formation

```
revision_id = BLAKE3-256(
    canonical_parent_ids
    || canonical_cell_ids
    || timestamp
    || message
)
```

Canonical ordering: parents sorted lexicographically, cells sorted lexicographically.

---

## 2. Revision DAG

### 2.1 DAG Structure

```
R1 ────┬────→ R3
│      │      ↑
└──→ R2 ┴────→ R4
```

### 2.2 DAG Invariants

1. Revisions form a DAG (Directed Acyclic Graph)
2. Cycles MUST be detected and rejected
3. Each revision MAY have multiple parents (merge)
4. Each revision MAY have multiple children (branch)
5. Root revision has zero parents
6. Leaf revision has zero children (tip of branch)

### 2.3 Revision Properties

| Property | Constraint |
|---|---|
| Parents | 0 (root) or ≥ 1 |
| Children | 0 (leaf) or ≥ 1 |
| Cells | ≥ 1 |
| Timestamp | Monotonically increasing |
| Message | UTF-8 string, 0–1024 bytes |

---

## 3. Revision Resolution

### 3.1 Resolution Process

Given a target revision_id:

```
1. Locate revision in Revision Region
2. Verify revision integrity (hash)
3. Collect all cell_ids in revision
4. For each cell_id, resolve via Cell Index
5. Verify each cell's integrity
6. Return resolved Cell set
```

### 3.2 Parent Resolution

For incremental evolution, the reader resolves the full Cell set by traversing parents:

```
fn resolve_revision(revision: &Revision, all_revisions: &RevisionIndex) -> HashSet<CellId> {
    let mut cells = HashSet::new();
    let mut stack = vec![revision];
    let mut visited = HashSet::new();
    
    while let Some(rev) = stack.pop() {
        if !visited.insert(rev.revision_id) {
            continue;
        }
        
        // Add this revision's cells
        for cell_id in &rev.cell_ids {
            cells.insert(*cell_id);
        }
        
        // Visit parents
        for parent_id in &rev.parent_ids {
            if let Some(parent) = all_revisions.get(parent_id) {
                stack.push(parent);
            }
        }
    }
    
    cells
}
```

### 3.3 Cell Override

When multiple revisions contribute cells, the most recent revision's cells override:

```
R1: {C1, C2, C3, C4}
R2 (parent: R1): {C3'}  // C3' replaces C3

Resolved R2: {C1, C2, C3', C4}
```

---

## 4. Incremental Evolution

### 4.1 Cell Sharing

When a model changes, only modified Cells produce new entries:

```
R1: {C1, C2, C3, C4}

Modify C3 → creates C3' with new identity

R2: {C1, C2, C3', C4}  // C1, C2, C4 shared, not copied
```

### 4.2 Sharing Invariants

1. Shared Cells are NOT duplicated
2. Shared Cells retain identical content
3. Shared Cells retain identical identity
4. Only changed Cells create new entries
5. New revision shares all unchanged Cells with parent

### 4.3 Storage Efficiency

For a model with N Cells and M changed Cells:
- Storage for new revision: M new Cell entries + metadata
- NOT: N Cell entries (full copy)

---

## 5. Merge Semantics

### 5.1 Merge Operation

When two branches converge:

```
R1 → R2a (changes: C1')
R1 → R2b (changes: C2')

Merge(R2a, R2b) → R3
```

### 5.2 Merge Resolution

R3 contains:
- All cells from R2a
- All cells from R2b
- Conflict resolution for cells modified in both branches

### 5.3 Conflict Detection

A conflict occurs when:
- Same cell_id modified in both branches
- Different new CellIds produced for same original Cell

### 5.4 Conflict Resolution

```
fn merge(rev_a: &Revision, rev_b: &Revision) -> Result<Revision, NCIError> {
    let cells_a = resolve_revision(rev_a);
    let cells_b = resolve_revision(rev_b);
    
    // Detect conflicts
    let original_a = rev_a.original_cells();  // Cells before modification
    let original_b = rev_b.original_cells();
    
    let conflicts = original_a.intersection(&original_b)
        .filter(|c| cells_a.contains(c) != cells_b.contains(c))
        .collect::<Vec<_>>();
    
    if !conflicts.is_empty() {
        return Err(NCIError::MergeConflict(conflicts));
    }
    
    // Merge
    let mut merged_cells = cells_a;
    merged_cells.extend(cells_b);
    
    Ok(Revision::new(vec![rev_a.id, rev_b.id], merged_cells))
}
```

### 5.5 Merge Conflict Error

On conflict:
1. Return `NCI-E-MERGE`
2. List conflicting CellIds
3. Operator must resolve manually
4. No automatic conflict resolution

---

## 6. Garbage Collection

### 6.1 When to GC

GC occurs when:
- Explicitly requested by operator
- Storage space is low
- Revision history is too long

### 6.2 GC Rules

1. GC MUST NOT delete any Cell referenced by a live revision
2. GC MAY delete Cells only referenced by garbage revisions
3. GC MUST preserve at least one root revision
4. GC MUST be logged

### 6.3 Live Revision

A revision is live if:
- It is the current HEAD
- It is reachable from HEAD via parent links
- It is explicitly pinned by operator

### 6.4 GC Algorithm

```
fn garbage_collect(revisions: &mut RevisionIndex, cells: &mut CellStore) {
    // 1. Mark live revisions
    let live_revisions = mark_live(revisions);
    
    // 2. Collect live cell IDs
    let live_cells: HashSet<CellId> = live_revisions.iter()
        .flat_map(|r| r.cell_ids.iter())
        .cloned()
        .collect();
    
    // 3. Delete dead cells
    let dead_cells: Vec<CellId> = cells.keys()
        .filter(|c| !live_cells.contains(c))
        .cloned()
        .collect();
    
    for cell_id in dead_cells {
        cells.remove(&cell_id);
    }
    
    // 4. Delete dead revisions
    let dead_revisions: Vec<Hash256> = revisions.keys()
        .filter(|r| !live_revisions.contains(r))
        .cloned()
        .collect();
    
    for rev_id in dead_revisions {
        revisions.remove(&rev_id);
    }
}
```

---

## 7. Revision Integrity

### 7.1 Verification

1. Compute revision hash from canonical data
2. Compare with stored revision_id
3. Verify all parent revisions exist and are valid
4. Verify all cell_ids exist in Cell Index
5. Verify timestamp is monotonically increasing

### 7.2 Chain Verification

```
fn verify_revision_chain(revision: &Revision, index: &RevisionIndex) -> Result<()> {
    // Verify this revision
    let computed_id = blake3_256(&revision.canonical_bytes());
    if computed_id != revision.revision_id {
        return Err(NCIError::Corrupt);
    }
    
    // Verify parents
    for parent_id in &revision.parent_ids {
        if let Some(parent) = index.get(parent_id) {
            verify_revision_chain(parent, index)?;
        } else {
            return Err(NCIError::RevisionBroken);
        }
    }
    
    Ok(())
}
```

---

## 8. Validation

1. Revision DAG MUST be acyclic.
2. Revision hash MUST match computed hash.
3. All parent revisions MUST exist.
4. All cell_ids MUST exist in Cell Index.
5. Timestamp MUST be monotonically increasing.
6. Merge conflicts MUST be detected and reported.

---

## 9. Error Handling

| Condition | Error Code |
|---|---|
| Revision hash mismatch | `NCI-E-CORRUPT` |
| Parent revision not found | `NCI-E-REVISION` |
| Cell not found in index | `NCI-E-NOTFOUND` |
| DAG cycle detected | `NCI-E-FORMAT` |
| Merge conflict | `NCI-E-MERGE` |
| GC would delete live cell | `NCI-E-STORE` |

---

## 10. Conformance Requirements

1. Revisions MUST be immutable.
2. DAG MUST be acyclic.
3. Revision resolution MUST traverse parents.
4. Cell sharing MUST be implemented.
5. Merge conflict detection MUST be implemented.
6. GC MUST NOT delete live cells.
7. All revisions MUST be verifiable.
