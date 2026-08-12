# NCL Canonical Serialization Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §9; `ncl/01-NCL-BINARY-SPEC.md`; `03-NCI-CELL-SPEC.md`

---

## Purpose

This document specifies the deterministic canonical serialization used for Cell identity computation. Canonical serialization ensures that identical logical Cells produce identical byte sequences regardless of implementation, compiler, platform, or runtime state.

## Scope

Canonical encoding rules for Cell headers, metadata, payloads, NaN handling, padding, dimension encoding, metadata field ordering, dependency ordering, and the complete canonical byte sequence format.

---

## 1. Canonicalization Requirement

### 1.1 Identity Pipeline

```
Logical Cell
    ↓
Canonical Encoder (this document)
    ↓
Deterministic Byte Sequence
    ↓
BLAKE3-256
    ↓
CellId (32 bytes)
```

### 1.2 Invariants

1. Given identical logical Cell content, canonicalization MUST produce identical byte sequence across all implementations.
2. Canonicalization MUST be deterministic — no randomness, no timestamps, no memory addresses.
3. Canonicalization MUST be independent of compression codec.
4. Canonicalization MUST be independent of physical storage layout.
5. Canonicalization MUST be independent of file name, path, or offset.

---

## 2. Canonical Cell Byte Sequence

### 2.1 Top-Level Structure

```
canonical_cell_header (32 bytes)
canonical_cell_metadata (variable)
canonical_uncompressed_payload (variable)
```

The CellId is computed as:

```
CellId = BLAKE3-256(canonical_cell_header || canonical_cell_metadata || canonical_uncompressed_payload)
```

### 2.2 Canonical Cell Header

```rust
#[repr(C, packed)]
struct CanonicalCellHeader {
    kind: u8,                      // offset 0, size 1
    representation: u32,           // offset 1, size 4
    dependency_count: u32,         // offset 5, size 4
    attribute_count: u32,          // offset 9, size 4
    payload_size: u64,             // offset 13, size 8
    reserved: [u8; 7],            // offset 21, size 7
}
// Total: 28 bytes
```

**Note:** The SchemaId is NOT in the header — it is part of metadata. This ensures the header is fixed-size.

### 2.3 Canonical Cell Metadata

Following the header, metadata is serialized in this fixed order:

```
schema_id (32 bytes)
dependencies (32 bytes × dependency_count, sorted lexicographically)
attributes (variable, canonical key order)
```

**Critical:** The ordering above is MANDATORY. Implementations MUST NOT reorder these fields.

### 2.4 Canonical Uncompressed Payload

The payload follows metadata. Its format depends on the Cell's schema:

- For tensor Cells: canonical tensor format (§3)
- For graph Cells: canonical edge list (§4)
- For routing Cells: canonical routing table (§5)
- For memory Cells: canonical memory format (§6)
- For metadata Cells: canonical key-value format (§7)

---

## 3. Canonical Tensor Format

### 3.1 Tensor Descriptor

```rust
#[repr(C, packed)]
struct CanonicalTensorDescriptor {
    dtype: u8,                    // offset 0, size 1
    rank: u8,                     // offset 1, size 1
    num_elements: u64,            // offset 2, size 8
    strides_computed: u8,         // offset 10, size 1
    // dimensions follow (8 bytes × rank)
}
```

### 3.2 Dimension Encoding

Dimensions are encoded as `u64` values, one per rank, in order:

```
[dim_0][dim_1]...[dim_rank-1]
```

For a 2D tensor [4096, 4096]:
```
[0x00, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]  // 4096
[0x00, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]  // 4096
```

### 3.3 Element Encoding

Elements are encoded in row-major (C) order, contiguous, with no padding:

```
[element_0][element_1]...[element_N-1]
```

Element size depends on dtype:

| dtype | Size | Encoding |
|---|---|---|
| 0x01 (FP32) | 4 bytes | IEEE 754 binary32, little-endian |
| 0x02 (BF16) | 2 bytes | Brain float16, little-endian |
| 0x03 (FP16) | 2 bytes | IEEE 754 binary16, little-endian |
| 0x04 (FP8) | 1 byte | E4M3 or E5M2 (per schema), little-endian |
| 0x05 (INT8) | 1 byte | Signed 8-bit, two's complement |
| 0x06 (INT4) | 1 byte | Two INT4 values packed, low nibble first |
| 0x07 (INT3) | variable | Bit-packed, LSB first |

### 3.4 INT4 Packing

Two INT4 values per byte:
- Low nibble (bits 0-3): element at even index
- High nibble (bits 4-7): element at odd index
- If element count is odd, last byte has unused high nibble (zero-padded)

### 3.5 INT3 Packing

Three INT3 values per 9 bits, packed LSB first:
- Bits 0-2: element 0
- Bits 3-5: element 1
- Bits 6-8: element 2
- Padding to byte boundary as needed

---

## 4. Canonical NaN Handling

### 4.1 Rule

For identity computation, ALL NaN values in the payload MUST be replaced with a canonical NaN before hashing.

### 4.2 Canonical NaN Values

| Type | Canonical NaN | Encoding |
|---|---|---|
| FP32 | Quiet NaN | `0xFFC00000` |
| BF16 | Quiet NaN | `0x7FC0` |
| FP16 | Quiet NaN | `0x7FC0` |
| FP8 (E4M3) | NaN | `0x7F` |
| FP8 (E5M2) | NaN | `0x7FC` |

### 4.3 Canonicalization Process

```
for element in payload:
    if is_nan(element):
        element = canonical_nan(element.dtype)
```

### 4.4 Signed Zero

- Negative zero (-0.0) MUST be preserved (NOT canonicalized to +0.0)
- -0.0 and +0.0 are distinct values for identity computation

---

## 5. Canonical Padding

### 5.1 Rule

All padding bytes MUST be 0x00. Padding is used for alignment but MUST be consistent.

### 5.2 Alignment Padding

Between fields that require alignment, padding bytes of 0x00 are inserted:

```
[field_a][padding][field_b]
```

Padding size = `(alignment - (offset % alignment)) % alignment`

### 5.3 Padding Exclusion from Identity

**CRITICAL:** Padding bytes added for alignment in the physical format MUST NOT be included in the canonical byte sequence for hashing. The canonical sequence is紧凑 (compact) with no alignment padding.

```
Physical (with padding):  [A][00][00][B][00][00][00][C]
Canonical (no padding):   [A][B][C]
```

---

## 6. Canonical Attribute Ordering

### 6.1 Key Ordering

Attributes MUST be sorted by key in lexicographic byte order.

### 6.2 Encoding

```
attribute_count (u32, little-endian)
key_len_1 (u16, little-endian)
key_1 (UTF-8 bytes)
value_type_1 (u8)
value_len_1 (u32, little-endian)
value_1 (type-encoded bytes)
key_len_2 (u16, little-endian)
key_2 (UTF-8 bytes)
...
```

### 6.3 Value Encoding

| Type | Canonical Encoding |
|---|---|
| Null (0x00) | No bytes |
| Bool (0x01) | 1 byte: 0x00 or 0x01 |
| U64 (0x02) | 8 bytes, little-endian |
| I64 (0x03) | ZigZag encoded, VarInt |
| F64 (0x04) | 8 bytes, little-endian, NaN canonicalized |
| String (0x05) | UTF-8 bytes, no BOM |
| Bytes (0x06) | Raw bytes |
| CellId (0x07) | 32 bytes raw |
| List (0x08) | u32 count + repeated value encoding |

---

## 7. Canonical Dependency Ordering

### 7.1 Rule

Dependencies MUST be sorted by their raw CellId bytes in lexicographic (unsigned byte) order.

### 7.2 Encoding

```
dependency_0 (32 bytes)
dependency_1 (32 bytes)
...
dependency_N-1 (32 bytes)
```

### 7.3 Empty Dependencies

If dependency_count is 0, no bytes are emitted for dependencies.

---

## 8. Canonical Schema Encoding

### 8.1 Schema Definition

A schema is itself a content-addressed blob:

```
schema_canonical = 
    schema_version (u16, little-endian)
    field_count (u32, little-endian)
    fields_sorted_by_name:
        field_name_len (u16, little-endian)
        field_name (UTF-8 bytes)
        field_type (u8)
        field_size (u32, little-endian)
```

### 8.2 SchemaId

```
SchemaId = BLAKE3-256(schema_canonical)
```

---

## 9. Cross-Implementation Determinism Test

### 9.1 Test Protocol

Two implementations are conformant if:

1. Given identical logical Cell content
2. Both produce identical canonical byte sequence
3. Both compute identical CellId

### 9.2 Known Test Vector

```
Input Cell:
  kind: Weight (0x01)
  representation: FP32 (0x00000001)
  schema: [32 bytes of zeros]
  dependencies: []
  attributes: {}
  payload: [4 bytes of 0x3F800000] (FP32 value 1.0)

Canonical byte sequence:
  header: [0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]
  metadata: [32 bytes of zeros (schema)]
  payload: [0x00, 0x00, 0x80, 0x3F]

CellId = BLAKE3-256(above bytes)
```

---

## 10. Validation

1. Canonical byte sequence MUST be identical across implementations for identical input.
2. NaN canonicalization MUST produce consistent results.
3. Attribute ordering MUST be lexicographic.
4. Dependency ordering MUST be lexicographic by raw bytes.
5. Padding MUST NOT appear in canonical sequences.
6. SchemaId MUST be deterministic.

---

## 11. Error Handling

| Condition | Error Code | Action |
|---|---|---|
| Non-deterministic canonicalization | `NCI-E-FORMAT` | Reject implementation |
| NaN not canonicalized | `NCI-E-CORRUPT` | Reject payload |
| Attributes not sorted | `NCI-E-FORMAT` | Reject Cell |
| Dependencies not sorted | `NCI-E-FORMAT` | Reject Cell |

---

## 12. Conformance Requirements

1. All implementations MUST produce identical canonical byte sequences for identical logical Cells.
2. NaN canonicalization MUST follow §4 exactly.
3. Attribute and dependency ordering MUST follow §6 and §7 exactly.
4. Padding MUST NOT appear in canonical sequences (§5).
5. The test vector in §9.2 MUST be passable by all conformant implementations.
