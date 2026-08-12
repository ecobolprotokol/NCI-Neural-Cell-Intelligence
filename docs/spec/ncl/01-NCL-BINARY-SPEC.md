# NCL Binary Format Specification

**Version:** 1.0.0
**Status:** Normative (PRIMARY SOURCE for physical format)
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §7; `01-NCI-TERMINOLOGY.md`

---

## Purpose

This document is the authoritative specification for the NCL binary format. It defines the complete physical layout: every byte offset, field size, type, encoding, alignment, endianness, valid range, validation rule, and error condition. All NCL implementations MUST conform to this specification.

## Scope

NCL file layout, superblock, region structure, column physical encoding, block structure, and the binary-level contracts governing all data in an NCL file.

---

## 1. Format Authority

NCL is a new, independent binary format. It is NOT a wrapper around PyTorch, Safetensors, GGUF, or any other format. It is NOT a ZIP container, TAR archive, or any archive format. The `.ncl` file extension is the canonical file extension.

---

## 2. File Layout

### 2.1 Region Order

An NCL file MUST contain the following regions in order:

```
Offset 0
┌──────────────────────────────────────┐
│ NCL SUPERBLOCK (4096 bytes fixed)    │
├──────────────────────────────────────┤
│ MANIFEST REGION                      │
├──────────────────────────────────────┤
│ CELL COLUMN REGION                   │
├──────────────────────────────────────┤
│ GRAPH COLUMN REGION                  │
├──────────────────────────────────────┤
│ ROUTING COLUMN REGION                │
├──────────────────────────────────────┤
│ MEMORY COLUMN REGION                 │
├──────────────────────────────────────┤
│ REPRESENTATION REGION                │
├──────────────────────────────────────┤
│ PAYLOAD REGION                       │
├──────────────────────────────────────┤
│ INDEX REGION                         │
├──────────────────────────────────────┤
│ REVISION REGION                      │
├──────────────────────────────────────┤
│ PROVENANCE REGION                    │
├──────────────────────────────────────┤
│ INTEGRITY REGION                     │
└──────────────────────────────────────┘
```

**Note:** Region order MAY be rearranged in physical implementation provided manifest offsets and dependencies remain correct. However, the Superblock MUST always be at offset 0.

### 2.2 Region Addressing

All region offsets in the Superblock are byte offsets from the beginning of the file. A region offset of 0 combined with a region size of 0 indicates the region is absent.

---

## 3. Superblock

### 3.1 Fixed Structure

The Superblock is exactly 4096 bytes. All fields are little-endian.

```rust
#[repr(C, packed)]
struct NclSuperblock {
    magic: [u8; 4],                // offset 0, size 4
    major_version: u16,            // offset 4, size 2
    minor_version: u16,            // offset 6, size 2
    flags: u32,                    // offset 8, size 4
    endianness: u8,                // offset 12, size 1
    hash_algorithm: u8,            // offset 13, size 1
    header_size: u16,              // offset 14, size 2
    file_size: u64,                // offset 16, size 8
    manifest_offset: u64,          // offset 24, size 8
    manifest_size: u64,            // offset 32, size 8
    cell_region_offset: u64,       // offset 40, size 8
    graph_region_offset: u64,      // offset 48, size 8
    routing_region_offset: u64,    // offset 56, size 8
    memory_region_offset: u64,     // offset 64, size 8
    representation_offset: u64,    // offset 72, size 8
    payload_offset: u64,           // offset 80, size 8
    index_offset: u64,             // offset 88, size 8
    revision_offset: u64,          // offset 96, size 8
    provenance_offset: u64,        // offset 104, size 8
    integrity_offset: u64,         // offset 112, size 8
    cell_count: u64,               // offset 120, size 8
    column_count: u64,             // offset 128, size 8
    block_count: u64,              // offset 136, size 8
    root_hash: [u8; 32],           // offset 144, size 32
    manifest_hash: [u8; 32],       // offset 176, size 32
    reserved: [u8; 3888],          // offset 208, size 3888
}
// Total: 4096 bytes
```

### 3.2 Field Specifications

#### magic

| Property | Value |
|---|---|
| Offset | 0 |
| Size | 4 bytes |
| Type | `u8[4]` |
| Encoding | Raw bytes |
| Value | `NCL\x01` = `[0x4E, 0x43, 0x4C, 0x01]` |
| Validation | MUST match exactly |
| Error | `NCI-E-FORMAT` |

#### major_version

| Property | Value |
|---|---|
| Offset | 4 |
| Size | 2 bytes |
| Type | `u16` |
| Encoding | Little-endian |
| Valid Range | 1 (current) |
| Validation | MUST equal reader's major version |
| Error | `NCI-E-SCHEMA` if mismatch |

#### minor_version

| Property | Value |
|---|---|
| Offset | 6 |
| Size | 2 bytes |
| Type | `u16` |
| Encoding | Little-endian |
| Valid Range | ≥ 0 |
| Validation | Reader MAY reject if higher than known |
| Error | `NCI-E-SCHEMA` |

#### flags

| Property | Value |
|---|---|
| Offset | 8 |
| Size | 4 bytes |
| Type | `u32` |
| Encoding | Little-endian |
| Valid Range | Bitfield (see below) |

**Flag Bits:**

| Bit | Name | Meaning |
|---|---|---|
| 0 | HAS_COMPRESSION | File uses compression |
| 1 | HAS_DEDUPLICATION | File uses deduplication |
| 2 | HAS_QUANTIZATION | File uses quantization |
| 3 | HAS_REVISIONS | File contains revision history |
| 4 | HAS_ROUTING | File contains routing data |
| 5 | HAS_MEMORY | File contains memory data |
| 6 | HAS_GRAPH | File contains graph data |
| 7 | STREAMING_OK | File was written in streaming mode |
| 8–31 | Reserved | MUST be zero |

#### endianness

| Property | Value |
|---|---|
| Offset | 12 |
| Size | 1 byte |
| Type | `u8` |
| Valid Range | MUST be 1 (little-endian) |
| Validation | MUST equal 1 |
| Error | `NCI-E-FORMAT` |

#### hash_algorithm

| Property | Value |
|---|---|
| Offset | 13 |
| Size | 1 byte |
| Type | `u8` |
| Valid Range | MUST be 1 (BLAKE3-256) |
| Validation | MUST equal 1 |
| Error | `NCI-E-FORMAT` |

#### header_size

| Property | Value |
|---|---|
| Offset | 14 |
| Size | 2 bytes |
| Type | `u16` |
| Encoding | Little-endian |
| Valid Range | MUST be 4096 |
| Validation | MUST equal 4096 |
| Error | `NCI-E-FORMAT` |

#### file_size

| Property | Value |
|---|---|
| Offset | 16 |
| Size | 8 bytes |
| Type | `u64` |
| Encoding | Little-endian |
| Valid Range | ≥ 4096 |
| Validation | MUST equal actual file size |
| Error | `NCI-E-FORMAT` |

#### Region Offsets (manifest_offset through integrity_offset)

| Property | Value |
|---|---|
| Size | 8 bytes each |
| Type | `u64` |
| Encoding | Little-endian |
| Valid Range | 0 to file_size |
| Validation | If non-zero, offset + size ≤ file_size |
| Error | `NCI-E-FORMAT` |

#### cell_count, column_count, block_count

| Property | Value |
|---|---|
| Size | 8 bytes each |
| Type | `u64` |
| Encoding | Little-endian |
| Valid Range | ≥ 0 |
| Validation | Counts MUST match actual data |

#### root_hash

| Property | Value |
|---|---|
| Offset | 144 |
| Size | 32 bytes |
| Type | `u8[32]` |
| Encoding | BLAKE3-256 of Manifest content |
| Validation | MUST match computed hash |
| Error | `NCI-E-MANIFEST` |

#### manifest_hash

| Property | Value |
|---|---|
| Offset | 176 |
| Size | 32 bytes |
| Type | `u8[32]` |
| Encoding | BLAKE3-256 of entire Manifest region |
| Validation | MUST match computed hash |
| Error | `NCI-E-MANIFEST` |

#### reserved

| Property | Value |
|---|---|
| Offset | 208 |
| Size | 3888 bytes |
| Encoding | Zero-filled (0x00) |
| Validation | MUST be all zeros |
| Error | `NCI-E-FORMAT` (warning) |

---

## 4. Manifest Region

### 4.1 Structure

```rust
#[repr(C, packed)]
struct Manifest {
    model_id: [u8; 32],            // offset 0, size 32
    revision_id: [u8; 32],         // offset 32, size 32
    cell_count: u64,               // offset 64, size 8
    column_count: u64,             // offset 72, size 8
    graph_root: [u8; 32],          // offset 80, size 32
    routing_root: [u8; 32],        // offset 112, size 32
    memory_root: [u8; 32],         // offset 144, size 32
    representation_root: [u8; 32], // offset 176, size 32
    index_root: [u8; 32],          // offset 208, size 32
    root_hash: [u8; 32],           // offset 240, size 32
}
// Total: 272 bytes minimum
```

All hash fields are BLAKE3-256 hashes of their respective region content. A zero hash indicates the region is absent.

---

## 5. Column Physical Encoding

### 5.1 Column Types

All column types are defined in Contract §6.1. Physical encoding follows.

### 5.2 RAW Encoding

```
[element_0][element_1]...[element_N]
```

Each element is `element_size` bytes. Total: `N × element_size`.

### 5.3 RLE Encoding

```
[count_0][value_0][count_1][value_1]...[count_K][value_K]
```

| Field | Type | Encoding |
|---|---|---|
| count | `u32` | Little-endian, number of consecutive identical values |
| value | `element_size` bytes | The repeated value |

Runs are sequential. Total element count = sum of all counts.

### 5.4 DELTA Encoding

```
[base_value][delta_0][delta_1]...[delta_N-1]
```

| Field | Type | Encoding |
|---|---|---|
| base_value | `element_size` bytes | First element value |
| delta_i | ZigZag VarInt | Signed difference from previous element |

### 5.5 DELTA_RLE Encoding

DELTA encoding with RLE on the delta stream.

### 5.6 BITPACK Encoding

```
[bit_width][bitstream]
```

| Field | Type | Encoding |
|---|---|---|
| bit_width | `u8` | Bits per value (1–32) |
| bitstream | Packed bits | Values packed sequentially |

Values are unsigned, packed LSB-first within each byte.

### 5.7 VARINT Encoding

Standard unsigned VarInt encoding:

```
while byte & 0x80 != 0:
    value |= (byte & 0x7F) << shift
    shift += 7
```

Each byte uses bit 7 as continuation flag.

### 5.8 VARINT DELTA Encoding

DELTA encoding where deltas are encoded as unsigned VarInts.

### 5.9 DICT Encoding

```
[dict_size][dict_entries...][indices...]
```

| Field | Type | Encoding |
|---|---|---|
| dict_size | `u32` | Number of unique values |
| dict_entries | Unique values, each `element_size` bytes | Sorted by value |
| indices | VarInt array | Index into dictionary for each element |

### 5.10 SPARSE Encoding

```
[total_count][sparse_count][index_0][value_0]...[index_K][value_K]
```

| Field | Type | Encoding |
|---|---|---|
| total_count | `u64` | Total elements (including defaults) |
| sparse_count | `u64` | Number of non-default elements |
| index_i | `u64` | Element index |
| value_i | `element_size` bytes | Element value |

Default value is 0 (or 0.0 for floats).

---

## 6. Payload Block Structure

### 6.1 Block Header

```rust
#[repr(C, packed)]
struct PayloadBlockHeader {
    block_id: u32,              // offset 0, size 4
    uncompressed_size: u64,     // offset 4, size 8
    compressed_size: u64,       // offset 12, size 8
    codec: u8,                  // offset 20, size 1
    checksum: u32,              // offset 21, size 4
}
// Total: 25 bytes
```

### 6.2 Field Specifications

#### block_id

| Property | Value |
|---|---|
| Size | 4 bytes |
| Type | `u32` |
| Encoding | Little-endian |
| Valid Range | 0 to block_count - 1 |

#### uncompressed_size

| Property | Value |
|---|---|
| Size | 8 bytes |
| Type | `u64` |
| Encoding | Little-endian |
| Valid Range | 65,536 (64 KiB) to 16,777,216 (16 MiB) |
| Validation | MUST match actual uncompressed data size |

#### compressed_size

| Property | Value |
|---|---|
| Size | 8 bytes |
| Type | `u64` |
| Encoding | Little-endian |
| Valid Range | 0 to file_size |
| Validation | MUST equal actual compressed data size on disk |
| If codec == 0 (uncompressed): compressed_size == uncompressed_size |

#### codec

| Property | Value |
|---|---|
| Size | 1 byte |
| Type | `u8` |

**Valid Values:**

| Value | Codec |
|---|---|
| 0x00 | None (uncompressed) |
| 0x01 | Zstandard (zstd) — mandatory |
| 0x02–0xFE | Reserved for future codecs |
| 0xFF | Implementation-defined (non-standard) |

#### checksum

| Property | Value |
|---|---|
| Size | 4 bytes |
| Type | `u32` |
| Encoding | CRC32 of compressed data (or uncompressed if codec==0) |
| Polynomial | IEEE 0x04C11DB7 |
| Initial Value | 0xFFFFFFFF |
| Final XOR | 0xFFFFFFFF |

### 6.3 Block Data

After the 25-byte header, the block data follows immediately:

```
[PayloadBlockHeader (25 bytes)][Compressed/Uncompressed Data]
```

### 6.4 Block Alignment

Each block MUST be aligned to 8-byte boundary. Padding bytes (0x00) MAY be inserted between blocks but MUST NOT be counted in compressed_size.

---

## 7. Region Headers

### 7.1 Column Region Header

Each column region (CELL, GRAPH, ROUTING, MEMORY) begins with a region header:

```rust
#[repr(C, packed)]
struct ColumnRegionHeader {
    region_type: u16,           // offset 0, size 2
    column_count: u32,          // offset 2, size 4
    total_rows: u64,            // offset 6, size 8
    total_data_size: u64,       // offset 14, size 8
    region_hash: [u8; 32],      // offset 22, size 32
}
// Total: 54 bytes
```

### 7.2 Column Entry

Following the region header, column entries describe each column:

```rust
#[repr(C, packed)]
struct ColumnEntry {
    column_type: u16,           // offset 0, size 2
    encoding: u8,               // offset 2, size 1
    element_size: u32,          // offset 3, size 4
    element_count: u64,         // offset 7, size 8
    data_offset: u64,           // offset 15, size 8
    data_size: u64,             // offset 23, size 8
    checksum: u32,              // offset 31, size 4
}
// Total: 35 bytes per column entry
```

### 7.3 Index Region

```rust
#[repr(C, packed)]
struct IndexRegionHeader {
    index_count: u32,           // offset 0, size 4
    index_type: u16,            // offset 2, size 2
    total_entries: u64,         // offset 4, size 8
    index_hash: [u8; 32],       // offset 12, size 32
}
// Total: 44 bytes
```

### 7.4 Revision Region

```rust
#[repr(C, packed)]
struct RevisionRegionHeader {
    revision_count: u32,        // offset 0, size 4
    root_revision: [u8; 32],    // offset 4, size 32
    dag_hash: [u8; 32],         // offset 36, size 32
}
// Total: 68 bytes
```

---

## 8. Numeric Encoding Rules

### 8.1 Byte Order

All multi-byte integers and floats MUST be little-endian.

### 8.2 Fixed-Width Types

| Type | Size | Encoding |
|---|---|---|
| u8 | 1 byte | Raw |
| u16 | 2 bytes | Little-endian |
| u32 | 4 bytes | Little-endian |
| u64 | 8 bytes | Little-endian |
| u128 | 16 bytes | Little-endian |
| i8 | 1 byte | Raw |
| i16 | 2 bytes | Little-endian |
| i32 | 4 bytes | Little-endian |
| i64 | 8 bytes | Little-endian |
| f16 | 2 bytes | Little-endian (IEEE 754 binary16) |
| bf16 | 2 bytes | Little-endian (brain float16) |
| f32 | 4 bytes | Little-endian (IEEE 754 binary32) |
| f64 | 8 bytes | Little-endian (IEEE 754 binary64) |

### 8.3 ZigZag Encoding

Signed integers use ZigZag encoding before VarInt:

```
ZigZag(n) = (n << 1) ^ (n >> 63)
```

| Original | Encoded |
|---|---|
| 0 | 0 |
| -1 | 1 |
| 1 | 2 |
| -2 | 3 |
| 2 | 4 |

### 8.4 Float Special Values

| Value | Canonical Encoding |
|---|---|
| NaN | `0xFFC00000` (f32), `0x7FC0` (f16) |
| +Infinity | `0x7F800000` (f32), `0x7C00` (f16) |
| -Infinity | `0xFF800000` (f32), `0xFC00` (f16) |
| -0.0 | `0x80000000` (f32), `0x8000` (f16) |

Canonical NaN: ALL NaN values MUST canonicalize to the canonical NaN for hashing. For identity computation, all NaNs are replaced with the canonical NaN before hashing.

### 8.5 Overflow Detection

Reader MUST reject:
- Unsigned integer overflow during VarInt decoding
- Signed integer overflow during ZigZag decoding
- Field values exceeding their declared valid ranges
- Offset calculations exceeding file_size

All overflow conditions MUST produce `NCI-E-FORMAT`.

---

## 9. Alignment Rules

### 9.1 Default Alignment

All data structures MUST be aligned to 8-byte boundaries.

### 9.2 Padding

- Padding bytes MUST be 0x00
- Padding MUST NOT be included in canonical payload for hashing
- Padding MUST NOT be counted in data sizes

### 9.3 Hardware-Specific Alignment

Implementations MAY increase alignment to 16, 32, or 64 bytes for:
- SIMD vector operations
- GPU memory transfer optimization
- NPU alignment requirements

Increased alignment MUST NOT break cross-implementation compatibility.

---

## 10. Validation

### 10.1 Reader Validation Sequence

1. Read first 4 bytes, validate magic (`NCL\x01`)
2. Read entire Superblock (4096 bytes)
3. Validate endianness (MUST be 1)
4. Validate hash_algorithm (MUST be 1)
5. Validate header_size (MUST be 4096)
6. Validate file_size matches actual file size
7. Validate all region offsets are within file_size
8. Compute Manifest hash, compare with manifest_hash
9. Compute root hash, compare with root_hash
10. Load and validate Index Region
11. Spot-check random index entries

### 10.2 Column Validation

For each column:
1. Validate column_type is known
2. Validate encoding is known
3. Validate element_size is correct for type
4. Validate data_offset + data_size ≤ file_size
5. Validate checksum matches column data
6. Validate element_count is consistent

### 10.3 Block Validation

For each accessed block:
1. Validate block_id < block_count
2. Validate compressed_size > 0
3. Validate checksum matches block data
4. Decompress if compressed
5. Validate uncompressed_size matches declared size
6. Compute BLAKE3-256 of uncompressed data

---

## 11. Error Handling

| Condition | Error Code | Severity |
|---|---|---|
| Magic mismatch | `NCI-E-FORMAT` | FATAL |
| Major version mismatch | `NCI-E-SCHEMA` | FATAL |
| Endianness not 1 | `NCI-E-FORMAT` | FATAL |
| File size mismatch | `NCI-E-FORMAT` | FATAL |
| Region offset out of bounds | `NCI-E-FORMAT` | FATAL |
| Manifest hash mismatch | `NCI-E-MANIFEST` | FATAL |
| Root hash mismatch | `NCI-E-MANIFEST` | FATAL |
| Column checksum mismatch | `NCI-E-CORRUPT` | CORRUPTION |
| Block checksum mismatch | `NCI-E-CORRUPT` | CORRUPTION |
| Cell hash mismatch | `NCI-E-CORRUPT` | CORRUPTION |
| Unknown column type | `NCI-E-FORMAT` | WARNING (skip) |
| Unknown codec | `NCI-E-FORMAT` | FATAL |
| Decompression failure | `NCI-E-CORRUPT` | CORRUPTION |
| Integer overflow | `NCI-E-FORMAT` | FATAL |

---

## 12. Conformance Requirements

1. All field offsets, sizes, and types MUST match this specification exactly.
2. All multi-byte integers MUST be little-endian.
3. Superblock MUST be exactly 4096 bytes.
4. Magic bytes MUST be `NCL\x01`.
5. Reader MUST perform all validation steps in §10.1 before releasing any data.
6. Writer MUST produce valid CRC32 checksums for all columns and blocks.
7. Writer MUST produce valid BLAKE3-256 hashes for manifest and root.
8. Implementations MUST NOT rely on reserved fields having any particular value.
9. This document is the authoritative source for all binary layout details.
