# NCL Index Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §8; `ncl/01-NCL-BINARY-SPEC.md`

---

## Purpose

This document specifies NCL Index: the lookup structures enabling O(1) amortized access to Cells, Payloads, Tiles, Schemas, Representations, and Columns.

## Scope

Index types, index structure, index construction, index verification, multi-level indexing for large files, and index physical layout.

---

## 1. Index Types

### 1.1 Required Indexes

| Index | Key | Value | Lookup Complexity |
|---|---|---|---|
| Cell Index | CellId (32 bytes) | Cell descriptor (row index) | O(1) amortized |
| Payload Index | CellId (32 bytes) | PayloadRef (block_id, offset, size) | O(1) amortized |
| Tile Index | TileId (32 bytes) | Tile descriptor | O(1) amortized |
| Schema Index | SchemaId (32 bytes) | Schema definition | O(1) amortized |
| Representation Index | (CellId, RepresentationId) | Representation entry | O(1) amortized |
| Column Index | ColumnId (2 bytes) | Column descriptor | O(1) amortized |
| Dedup Index | PayloadHash (32 bytes) | Vec<CellId> | O(1) amortized |

---

## 2. Index Structure

### 2.1 Hash Table Index

Primary index implementation using open addressing with linear probing.

```rust
struct HashIndex {
    capacity: u64,              // Number of slots (power of 2)
    load_factor: f64,           // Current load factor
    entries: Vec<IndexEntry>,
}

#[repr(C, packed)]
struct IndexEntry {
    key_hash: [u8; 32],         // BLAKE3-256 of key
    value_offset: u64,          // Offset to value data
    value_size: u32,            // Size of value data
    flags: u8,                  // 0=empty, 1=occupied, 2=deleted
}
// Total: 45 bytes per entry
```

### 2.2 Binary Search Index

Alternative for sorted arrays. Used for columnar data that is already sorted.

```rust
struct BinarySearchIndex {
    sorted_keys: Vec<[u8; 32]>,
    value_offsets: Vec<u64>,
}
```

### 2.3 Multi-Level Index

For files > 1 TB, multi-level indexing reduces memory requirements:

```
Level 0: Coarse index (every 1024 entries)
Level 1: Fine index (every 1 entry)
```

Lookup:
1. Binary search Level 0 to find coarse bucket
2. Binary search Level 1 within bucket

Target: O(1) amortized maintained.

---

## 3. Index Construction

### 3.1 Cell Index Construction

```
fn build_cell_index(cells: &[Cell]) -> HashIndex {
    let capacity = next_power_of_two(cells.len() * 2);
    let mut index = HashIndex::new(capacity);
    
    for (row_idx, cell) in cells.iter().enumerate() {
        let key_hash = blake3_256(&cell.id);
        let value = CellIndexValue {
            row_index: row_idx as u64,
            kind: cell.kind,
            schema: cell.schema,
            representation: cell.representation,
        };
        index.insert(key_hash, &value);
    }
    
    index
}
```

### 3.2 Payload Index Construction

```
fn build_payload_index(cells: &[Cell], blocks: &[Block]) -> HashIndex {
    let mut index = HashIndex::new(next_power_of_two(cells.len() * 2));
    
    for cell in cells {
        let key_hash = blake3_256(&cell.id);
        let value = cell.payload_ref;
        index.insert(key_hash, &value.as_bytes());
    }
    
    index
}
```

### 3.3 Dedup Index Construction

```
fn build_dedup_index(cells: &[Cell]) -> HashIndex {
    let mut index = HashIndex::new(next_power_of_two(cells.len()));
    
    for cell in cells {
        let payload_hash = blake3_256(&cell.canonical_payload());
        let existing = index.get(&payload_hash);
        
        match existing {
            Some(mut cell_ids) => {
                cell_ids.push(cell.id);
                index.insert(payload_hash, &cell_ids);
            }
            None => {
                index.insert(payload_hash, &[cell.id]);
            }
        }
    }
    
    index
}
```

---

## 4. Index Physical Layout

### 4.1 Index Region Structure

```
┌──────────────────────────────────────┐
│ IndexRegionHeader (44 bytes)          │
├──────────────────────────────────────┤
│ Index Entry[0]                        │
│ Index Entry[1]                        │
│ ...                                  │
│ Index Entry[N-1]                      │
├──────────────────────────────────────┤
│ Index Data[0] (value data)            │
│ Index Data[1]                         │
│ ...                                  │
└──────────────────────────────────────┘
```

### 4.2 Index Entry Format

```rust
#[repr(C, packed)]
struct IndexEntryBinary {
    key: [u8; 32],              // offset 0, size 32
    value_offset: u64,          // offset 32, size 8
    value_size: u32,            // offset 40, size 4
    slot_flags: u8,             // offset 44, size 1
}
// Total: 45 bytes per entry
```

### 4.3 Index Hash Verification

The entire Index Region MUST have a BLAKE3-256 hash stored in the Superblock's `manifest_hash` (or a dedicated index hash field).

```
index_hash = BLAKE3-256(index_region_bytes)
```

Reader MUST verify this hash before using the index.

---

## 5. Index Operations

### 5.1 Lookup

```
fn lookup(index: &HashIndex, key: &[u8; 32]) -> Option<IndexEntry> {
    let hash = blake3_256(key);
    let slot = hash % index.capacity;
    
    for probe in 0..index.capacity {
        let entry = &index.entries[(slot + probe) % index.capacity];
        match entry.flags {
            0 => return None,  // Empty slot
            1 => {
                if entry.key_hash == hash {
                    return Some(entry.clone());
                }
            }
            2 => continue,  // Deleted slot
            _ => unreachable!(),
        }
    }
    
    None
}
```

### 5.2 Insert

```
fn insert(index: &mut HashIndex, key: &[u8; 32], value: &[u8]) -> Result<()> {
    if index.load_factor > 0.75 {
        index.resize(index.capacity * 2);
    }
    
    let hash = blake3_256(key);
    let slot = hash % index.capacity;
    
    for probe in 0..index.capacity {
        let entry = &mut index.entries[(slot + probe) % index.capacity];
        match entry.flags {
            0 | 2 => {
                entry.key_hash = hash;
                entry.value_offset = index.data_size;
                entry.value_size = value.len() as u32;
                entry.flags = 1;
                index.data.extend_from_slice(value);
                index.data_size += value.len() as u64;
                index.load_factor += 1.0 / index.capacity as f64;
                return Ok(());
            }
            _ => continue,
        }
    }
    
    Err(NCIError::IndexFull)
}
```

### 5.3 Resize

```
fn resize(index: &mut HashIndex, new_capacity: u64) {
    let old_entries = std::mem::take(&mut index.entries);
    index.capacity = new_capacity;
    index.entries = vec![IndexEntry::empty(); new_capacity as usize];
    index.load_factor = 0.0;
    
    for entry in old_entries {
        if entry.flags == 1 {
            insert_raw(index, &entry.key_hash, entry.value_offset, entry.value_size);
        }
    }
}
```

---

## 6. Index Verification

### 6.1 Verification Protocol

```
1. Load index header
2. Verify index hash against stored hash
3. Validate all slot_flags are 0, 1, or 2
4. Validate value_offset + value_size ≤ region_size
5. Count occupied slots, verify load_factor
6. Spot-check: select 100 random occupied slots, verify lookup correctness
7. If ANY check fails → NCI-E-CORRUPT
```

### 6.2 Spot-Check Algorithm

```
fn spot_check(index: &HashIndex, sample_size: usize) -> Result<()> {
    let occupied: Vec<_> = index.entries.iter()
        .enumerate()
        .filter(|(_, e)| e.flags == 1)
        .collect();
    
    let mut rng = ChaCha8Rng::seed_from_u64(index.capacity);  // Deterministic
    let sample = random_sample(&occupied, sample_size, &mut rng);
    
    for (slot_idx, entry) in sample {
        // Verify key can be found at expected slot
        let computed_slot = entry.key_hash % index.capacity;
        let found = lookup(index, &entry.key_hash);
        
        match found {
            Some(f) if f.value_offset == entry.value_offset => continue,
            _ => return Err(NCIError::IndexCorrupt),
        }
    }
    
    Ok(())
}
```

---

## 7. Large File Indexing

### 7.1 File Size Thresholds

| File Size | Index Strategy |
|---|---|
| < 1 GB | Single-level hash index in memory |
| 1 GB – 100 GB | Single-level hash index, memory-mapped |
| 100 GB – 1 TB | Multi-level index (2 levels) |
| > 1 TB | Multi-level index (3 levels) |

### 7.2 Memory-Mapped Index

For files that fit in virtual address space:
- Index is memory-mapped
- Lookup uses page-fault-based lazy loading
- No explicit streaming required

### 7.3 Disk-Resident Index

For files exceeding virtual address space:
- Index is stored on disk
- Lookup requires I/O (one or two seeks for multi-level)
- Cache frequently accessed entries in RAM

---

## 8. Validation

1. Index hash MUST match stored hash.
2. All slot_flags MUST be 0, 1, or 2.
3. Value offsets MUST be within region bounds.
4. Load factor MUST be ≤ 0.75.
5. Spot-check MUST pass for 100 random entries.
6. Total entries MUST match cell_count (for Cell Index).

---

## 9. Error Handling

| Condition | Error Code |
|---|---|
| Index hash mismatch | `NCI-E-CORRUPT` |
| Invalid slot_flags | `NCI-E-CORRUPT` |
| Value offset out of bounds | `NCI-E-CORRUPT` |
| Load factor > 0.75 | `NCI-E-CORRUPT` |
| Spot-check failure | `NCI-E-CORRUPT` |
| Index full (insert failure) | `NCI-E-STORE` |

---

## 10. Conformance Requirements

1. All implementations MUST support hash table index.
2. Index lookup MUST be O(1) amortized.
3. Index MUST be verifiable before use.
4. Multi-level indexing MUST maintain O(1) amortized lookup.
5. Index construction MUST be deterministic.
6. Spot-check MUST use deterministic RNG for reproducibility.
