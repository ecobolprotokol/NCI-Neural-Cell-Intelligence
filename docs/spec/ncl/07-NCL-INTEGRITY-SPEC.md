# NCL Integrity Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §22; `ncl/01-NCL-BINARY-SPEC.md`

---

## Purpose

This document specifies NCL integrity: verification protocols, corruption detection, quarantine procedures, recovery strategies, and the integrity chain from block to manifest.

## Scope

Integrity chain, verification order, hash verification, checksum verification, corruption isolation, quarantine protocol, recovery protocol, and evidence preservation.

---

## 1. Integrity Chain

### 1.1 Chain Structure

```
Superblock
    ↓ root_hash
Manifest
    ↓ manifest_hash
Index Region
    ↓ index_hash
Column Regions
    ↓ column_checksums
Payload Blocks
    ↓ block_checksums
Cell Payloads
    ↓ cell_id hash
Verification Complete
```

### 1.2 Verification Order

Reader MUST verify in this exact order:

1. **Superblock magic** — First 4 bytes MUST be `NCL\x01`
2. **Superblock structure** — All fields within valid ranges
3. **Manifest hash** — Computed hash MUST match `manifest_hash` in Superblock
4. **Root hash** — Computed hash MUST match `root_hash` in Superblock
5. **Index hash** — Index region hash MUST match declared hash
6. **Column checksums** — Each column's CRC32 MUST match
7. **Block checksums** — Each accessed block's CRC32 MUST match
8. **Cell payload hashes** — Decompressed payload BLAKE3-256 MUST match CellId

---

## 2. Hash Algorithms

### 2.1 BLAKE3-256

Used for:
- Cell identity (CellId)
- Manifest hash
- Root hash
- Index hash
- Representation identity
- Schema identity

**Output:** 32 bytes (256 bits)
**Streaming:** Supported — can compute hash incrementally

### 2.2 CRC32 (IEEE)

Used for:
- Column data checksums
- Block data checksums

**Polynomial:** 0x04C11DB7
**Initial Value:** 0xFFFFFFFF
**Final XOR:** 0xFFFFFFFF
**Reflect In/Out:** Yes

### 2.3 Hash Selection

| Use Case | Algorithm | Rationale |
|---|---|---|
| Content addressing | BLAKE3-256 | Cryptographic, collision-resistant |
| Data integrity check | CRC32 | Fast, hardware-accelerated |
| Index verification | BLAKE3-256 | Cryptographic, tamper-evident |

---

## 3. Block-Level Integrity

### 3.1 Block Checksum

Each block has a CRC32 checksum computed over the compressed data (or uncompressed if codec == 0):

```
block_checksum = CRC32(block_data_bytes)
```

### 3.2 Block Verification

```
1. Read block header (25 bytes)
2. Read block data (compressed_size bytes)
3. Compute CRC32 of block data
4. Compare with header.checksum
5. If mismatch → NCI-E-CORRUPT
6. Decompress if codec != 0
7. Validate decompressed size == header.uncompressed_size
8. Compute BLAKE3-256 of decompressed data
9. Compare with stored CellId
10. If mismatch → NCI-E-CORRUPT
```

### 3.3 Decompression Bomb Protection

Before decompression:
- `uncompressed_size` MUST be ≤ 16 MiB (block maximum)
- `compressed_size` MUST be ≤ `uncompressed_size` (unless very small)
- If `compressed_size / uncompressed_size < 0.001` (compression ratio > 1000:1), flag as potential decompression bomb and require explicit allow

---

## 4. Column-Level Integrity

### 4.1 Column Checksum

Each column has a CRC32 checksum computed over the column data:

```
column_checksum = CRC32(column_data_bytes)
```

### 4.2 Column Verification

```
1. Read column entry (35 bytes)
2. Read column data (data_size bytes)
3. Compute CRC32 of column data
4. Compare with entry.checksum
5. If mismatch → NCI-E-CORRUPT
```

---

## 5. Cell-Level Integrity

### 5.1 Cell Identity Verification

```
1. Reconstruct Cell from columns
2. Compute canonical byte sequence
3. Compute BLAKE3-256 of canonical sequence
4. Compare with stored CellId
5. If mismatch → NCI-E-CORRUPT
```

### 5.2 Payload Verification

For each Cell's payload:
1. Locate block via PayloadRef
2. Read and decompress block
3. Verify block checksum
4. Verify decompressed data hash matches CellId

---

## 6. Corruption Isolation

### 6.1 Principle

Corruption in one block MUST NOT invalidate entire file. Corruption is quarantined to affected Cells only.

### 6.2 Isolation Model

```
NCL Model
├── Block A: Valid ✓
├── Block B: Valid ✓
├── Block C: CORRUPT ✗
└── Block D: Valid ✓

Cells using Block A: AVAILABLE
Cells using Block B: AVAILABLE
Cells using Block C: UNAVAILABLE (quarantined)
Cells using Block D: AVAILABLE
```

### 6.3 Corruption Propagation

A corrupted block affects:
- All Cells whose payload references that block
- All Cells that depend on affected Cells (transitive)
- NOT other Cells or Blocks

---

## 7. Quarantine Protocol

### 7.1 Quarantine Steps

Upon corruption detection:

```
1. LOG event with NCI-E-CORRUPT
   - File path
   - Block ID or Column ID
   - Offset
   - Expected hash
   - Actual hash
   - Timestamp

2. ISOLATE affected Cells
   - Mark Cell as UNAVAILABLE
   - Remove from active index
   - Notify dependent Cells

3. ATTEMPT alternate revision
   - If revision history exists, try prior revision
   - If prior revision available → use it
   - If not → affected Cells remain unavailable

4. ALERT operator (if automated system)
   - Emit NCI-E-CORRUPT metric
   - Log to observability system

5. DO NOT attempt repair
   - Immutable design prevents safe repair
   - Attempting repair would violate immutability

6. PRESERVE evidence
   - Do not overwrite corrupted blocks
   - Do not truncate file
   - Preserve for forensic analysis
```

### 7.2 Quarantine State

```rust
struct QuarantineState {
    corrupted_blocks: HashSet<u32>,
    corrupted_columns: HashSet<u16>,
    unavailable_cells: HashSet<CellId>,
    quarantine_timestamp: u64,
    alternate_revision: Option<Hash256>,
}
```

---

## 8. Recovery Protocol

### 8.1 Automatic Recovery

```
1. Detect corruption (NCI-E-CORRUPT)
2. Check if revision history exists
3. If yes:
   a. Locate nearest valid prior revision
   b. Verify prior revision integrity
   c. If valid → switch to prior revision
   d. Log recovery event
4. If no:
   a. Mark affected Cells as unavailable
   b. Continue with available Cells
   c. Log partial availability
```

### 8.2 Manual Recovery

For cases automatic recovery cannot handle:

```
1. Operator runs ncl-check on file
2. ncl-check reports corruption details
3. Operator runs ncl-dump to inspect
4. Operator decides:
   a. Use ncl-repack to create clean file (excludes corrupted blocks)
   b. Restore from backup
   c. Accept partial availability
```

### 8.3 Recovery Invariants

- Recovery MUST NOT modify original file
- Recovery MUST NOT lose valid data
- Recovery MUST preserve corruption evidence
- Recovery MUST be logged

---

## 9. Evidence Preservation

### 9.1 What to Preserve

- Original corrupted file (do not overwrite)
- Corruption detection log
- Hash mismatch details
- Block/column/offset information
- Recovery attempt results

### 9.2 Forensic Metadata

```rust
struct ForensicMetadata {
    corruption_type: CorruptionType,
    file_path: PathBuf,
    file_size: u64,
    corruption_offset: u64,
    expected_hash: [u8; 32],
    actual_hash: [u8; 32],
    detection_timestamp: u64,
    detection_method: DetectionMethod,
    recovery_attempted: bool,
    recovery_result: Option<RecoveryResult>,
}

enum CorruptionType {
    BlockChecksum,
    BlockDecompression,
    CellHashMismatch,
    ColumnChecksum,
    IndexCorruption,
    ManifestCorruption,
    SuperblockCorruption,
}

enum DetectionMethod {
    SuperblockValidation,
    ManifestValidation,
    BlockRead,
    ColumnRead,
    CellRead,
    SpotCheck,
    ManualCheck,
}
```

---

## 10. Verification Performance

### 10.1 Selective Verification

Reader MAY perform selective verification for performance:

- **Fast mode:** Verify only superblock, manifest, and index
- **Standard mode:** Verify superblock, manifest, index, and accessed blocks
- **Full mode:** Verify all blocks, columns, and cells

### 10.2 Lazy Verification

Reader MAY defer verification until data is actually accessed:
- Verify block checksum on first read
- Verify Cell hash on first use
- Cache verification results

---

## 11. Validation

1. All hash algorithms MUST produce deterministic results.
2. CRC32 MUST use IEEE polynomial.
3. BLAKE3-256 MUST produce 32-byte output.
4. Quarantine MUST be logged with full forensic metadata.
5. Recovery MUST NOT modify original file.
6. Verification order MUST be followed exactly.

---

## 12. Error Handling

| Condition | Error Code | Action |
|---|---|---|
| Superblock magic mismatch | `NCI-E-FORMAT` | Reject file |
| Manifest hash mismatch | `NCI-E-MANIFEST` | Reject file |
| Index hash mismatch | `NCI-E-CORRUPT` | Quarantine index |
| Column checksum mismatch | `NCI-E-CORRUPT` | Quarantine column |
| Block checksum mismatch | `NCI-E-CORRUPT` | Quarantine block |
| Cell hash mismatch | `NCI-E-CORRUPT` | Quarantine cell |
| Decompression bomb detected | `NCI-E-UNSAFE` | Reject block |
| Recovery failed | `NCI-E-RECOVERY` | Manual intervention |

---

## 13. Conformance Requirements

1. Reader MUST verify in the order specified in §1.2.
2. Corruption MUST be quarantined, not propagated.
3. Recovery MUST NOT modify original file.
4. Forensic metadata MUST be logged for all corruption events.
5. Decompression bomb protection MUST be implemented.
6. All implementations MUST support full verification mode.
7. Selective verification MUST be clearly documented.
