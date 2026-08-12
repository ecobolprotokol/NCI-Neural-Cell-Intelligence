# NCL Tensor Payload Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §11; `ncl/01-NCL-BINARY-SPEC.md`; `03-NCI-CELL-SPEC.md`

---

## Purpose

This document specifies tensor payload storage in NCL: tensor descriptors, tiling, block structure, data layout, and the rules governing tensor storage, retrieval, and verification.

## Scope

Tensor descriptor format, tiling algorithm, payload block structure, block addressing, tile addressing, data layout, and tensor integrity verification.

---

## 1. Tensor Descriptor

### 1.1 Descriptor Structure

```rust
#[repr(C, packed)]
struct TensorDescriptor {
    dtype: u8,                     // offset 0, size 1
    rank: u8,                      // offset 1, size 1
    num_elements: u64,             // offset 2, size 8
    layout: u8,                    // offset 10, size 1
    quantization_scheme: u8,       // offset 11, size 1
    scale_ref_hash: [u8; 32],      // offset 12, size 32
    zero_point_ref_hash: [u8; 32], // offset 44, size 32
    tile_count: u32,               // offset 76, size 4
    reserved: [u8; 4],            // offset 80, size 4
}
// Total: 84 bytes base + 8 × rank bytes for dimensions
```

### 1.2 Dimension Encoding

Following the base descriptor, dimensions are encoded:

```
[dim_0][dim_1]...[dim_rank-1]
```

Each dimension is `u64`, little-endian.

### 1.3 Data Type Values

| Value | Type | Bytes | Description |
|---|---|---|---|
| 0x01 | FP32 | 4 | IEEE 754 binary32 |
| 0x02 | BF16 | 2 | Brain float16 |
| 0x03 | FP16 | 2 | IEEE 754 binary16 |
| 0x04 | FP8_E4M3 | 1 | FP8 E4M3 |
| 0x05 | FP8_E5M2 | 1 | FP8 E5M2 |
| 0x06 | INT8 | 1 | Signed 8-bit integer |
| 0x07 | UINT8 | 1 | Unsigned 8-bit integer |
| 0x08 | INT4 | 0.5 | Signed 4-bit integer |
| 0x09 | INT3 | 0.375 | Signed 3-bit integer |
| 0x0A | BOOL | 1 | Boolean (0x00/0x01) |
| 0x0B | STRING | variable | Null-terminated UTF-8 |
| 0x0C | BYTES | variable | Length-prefixed bytes |

### 1.4 Layout Values

| Value | Layout | Description |
|---|---|---|
| 0x00 | RowMajor | Row-major (C order) |
| 0x01 | ColMajor | Column-major (Fortran order) |
| 0x02 | Blocked | Block-interleaved for hardware |
| 0x03 | SparseCSR | Compressed Sparse Row |
| 0x04 | SparseCSC | Compressed Sparse Column |

### 1.5 Quantization Scheme Values

| Value | Scheme | Description |
|---|---|---|
| 0x00 | None | No quantization |
| 0x01 | PerTensor | Single scale/zero-point per tensor |
| 0x02 | PerChannel | Scale/zero-point per output channel |
| 0x03 | PerGroup | Scale/zero-point per group of elements |

---

## 2. Tiling

### 2.1 Tiling Trigger

A tensor MUST be tiled when:
- `num_elements × element_size > 512 MiB` (max Cell size)
- `num_elements × element_size > target_cell_size` AND semantic boundary is soft

### 2.2 Tiling Strategy

**Row-Major Tiling (preferred for 1D–2D tensors):**

```
tile_size_elements = target_cell_size / element_size
split_dimension = 0 (largest dimension)
tile_shape[0] = min(dim[0], tile_size_elements / product(dim[1..rank]))
remaining = dim[0] / tile_shape[0]
```

**Block Tiling (for 2D+ tensors):**

```
block_size = floor(sqrt(target_cell_size / element_size))
tile_shape = [block_size, block_size]
grid = ceil(dim[0] / block_size) × ceil(dim[1] / block_size)
```

### 2.3 Tile Descriptor

```rust
#[repr(C, packed)]
struct TileDescriptor {
    tile_id: [u8; 32],            // offset 0, size 32 — BLAKE3-256 of tile
    parent_cell_id: [u8; 32],     // offset 32, size 32
    tile_index: u32,              // offset 64, size 4
    tile_rank: u8,                // offset 68, size 1
    // tile_shape: [u64; tile_rank] follows
    // tile_offset: [u64; tile_rank] follows
    dtype: u8,                    // follows dimensions
    representation: u32,          // follows dtype
    block_id: u32,                // follows representation
    block_offset: u32,            // follows block_id
    tile_size_bytes: u64,         // follows block_offset
}
```

### 2.4 Tile Ordering

Tiles MUST be stored in row-major order of their grid positions:
- Tile (0,0) → index 0
- Tile (0,1) → index 1
- ...
- Tile (1,0) → index grid_width
- etc.

---

## 3. Payload Block Structure

### 3.1 Block Header

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

### 3.2 Block Size Constraints

| Constraint | Value |
|---|---|
| Minimum | 64 KiB (65,536 bytes) |
| Default target | 1 MiB (1,048,576 bytes) |
| Maximum | 16 MiB (16,777,216 bytes) |

### 3.3 Block Alignment

Blocks MUST be aligned to 8-byte boundary. Padding bytes (0x00) are inserted between blocks.

### 3.4 Block Addressing

```
block_physical_offset = payload_region_offset + sum(block_sizes[0..block_id]) + padding
```

Blocks are addressed by `block_id` (0-indexed). The Index Region provides O(1) lookup from `block_id` to physical offset.

---

## 4. Tensor-to-Block Mapping

### 4.1 Untiled Tensor

Single tensor maps to single block:

```
Tensor T
  └── Block B (uncompressed_size = tensor_size)
```

### 4.2 Tiled Tensor

Tensor maps to multiple blocks (one per tile):

```
Tensor T
  ├── Tile 0 → Block B0
  ├── Tile 1 → Block B1
  ├── Tile 2 → Block B2
  └── ...
```

### 4.3 Shared Blocks (Deduplication)

Multiple Cells may reference the same block:

```
Cell C1 ──→ Block B0 (shared)
Cell C2 ──┘
```

Deduplication is handled by payload hash matching. See `ncl/07-NCL-INTEGRITY-SPEC.md`.

---

## 5. Tensor Data Layout

### 5.1 Row-Major Layout

For a 2D tensor [M, N]:

```
offset(i, j) = (i × N + j) × element_size
```

### 5.2 Column-Major Layout

For a 2D tensor [M, N]:

```
offset(i, j) = (j × M + i) × element_size
```

### 5.3 Blocked Layout

For a 2D tensor [M, N] with block size B:

```
block_row = i / B
block_col = j / B
local_row = i % B
local_col = j % B
offset = (block_row × ceil(N/B) × B × B + block_col × B × B + local_row × B + local_col) × element_size
```

---

## 6. Tensor Integrity

### 6.1 Verification Process

```
1. Read block from storage
2. Validate block checksum (CRC32)
3. Decompress if compressed
4. Validate uncompressed_size matches declared size
5. Compute BLAKE3-256 of uncompressed data
6. Compare with stored hash (CellId)
7. If match → release to runtime
8. If mismatch → NCI-E-CORRUPT, quarantine
```

### 6.2 Tile Verification

Each tile MUST be independently verifiable:
- Tile has its own tile_id (BLAKE3-256 of tile data)
- Tile block has its own checksum
- Tile data MUST decompress and verify independently

---

## 7. Quantized Tensor Storage

### 7.1 Scale and Zero-Point

For quantized tensors, scale and zero-point are stored as separate Cells:

```
Tensor T (INT8)
  ├── scale Cell (FP32 per-tensor or per-channel)
  └── zero_point Cell (INT8 per-tensor or per-channel)
```

Scale and zero-point are referenced by hash in the TensorDescriptor.

### 7.2 Group Quantization

For per-group quantization:
- Elements are divided into groups (default group size: 128)
- Each group has independent scale and zero-point
- Scale array has `ceil(num_elements / group_size)` entries
- Zero-point array has same count

---

## 8. Validation

1. Tensor descriptor MUST be fully parseable.
2. Dimensions MUST be > 0.
3. num_elements MUST equal product of dimensions.
4. Tile offsets MUST be within parent tensor bounds.
5. Block sizes MUST be within 64 KiB–16 MiB.
6. Block checksum MUST match.
7. Decompressed size MUST match uncompressed_size.

---

## 9. Error Handling

| Condition | Error Code |
|---|---|
| Invalid dtype | `NCI-E-FORMAT` |
| Invalid rank (0 or >16) | `NCI-E-FORMAT` |
| num_elements mismatch | `NCI-E-FORMAT` |
| Tile offset out of bounds | `NCI-E-CORRUPT` |
| Block checksum mismatch | `NCI-E-CORRUPT` |
| Decompressed size mismatch | `NCI-E-CORRUPT` |
| Hash mismatch | `NCI-E-CORRUPT` |

---

## 10. Conformance Requirements

1. All tensor data types in §1.3 MUST be supported.
2. Tiling MUST be deterministic given identical input and configuration.
3. Block verification MUST be performed before data release.
4. Tile verification MUST be independent of other tiles.
5. Quantized tensor storage MUST reference scale/zero-point by hash.
