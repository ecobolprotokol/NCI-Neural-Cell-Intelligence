# NCL Test Vectors

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `ncl/01-NCL-BINARY-SPEC.md`; `ncl/02-NCL-CANONICAL-SERIALIZATION.md`

---

## Purpose

This document specifies concrete test vectors for NCL: binary layout, canonical serialization, hashing, column encoding, index, integrity, compression, quantization, and reader/writer behavior.

## Scope

Test vectors for all major NCL operations, with expected inputs, outputs, and hash values.

---

## 1. Superblock Test Vector

### 1.1 Input

```
magic: NCL\x01
major_version: 1
minor_version: 0
flags: 0x00000001 (HAS_COMPRESSION)
endianness: 1
hash_algorithm: 1
header_size: 4096
file_size: 1048576
manifest_offset: 4096
manifest_size: 272
cell_region_offset: 4368
graph_region_offset: 0
routing_region_offset: 0
memory_region_offset: 0
representation_offset: 0
payload_offset: 4368
index_offset: 500000
revision_offset: 800000
provenance_offset: 900000
integrity_offset: 950000
cell_count: 100
column_count: 12
block_count: 50
```

### 1.2 Expected Binary (first 32 bytes)

```
Offset 0x00: 4E 43 4C 01  // magic "NCL\x01"
Offset 0x04: 01 00        // major_version = 1
Offset 0x06: 00 00        // minor_version = 0
Offset 0x08: 01 00 00 00  // flags = HAS_COMPRESSION
Offset 0x0C: 01           // endianness = 1 (little-endian)
Offset 0x0D: 01           // hash_algorithm = 1 (BLAKE3-256)
Offset 0x0E: 00 10        // header_size = 4096
```

### 1.3 Validation

```
assert superblock[0:4] == [0x4E, 0x43, 0x4C, 0x01]
assert superblock[4:6] == [0x01, 0x00]
assert superblock[12] == 1
assert superblock[13] == 1
assert superblock[14:16] == [0x00, 0x10]
```

---

## 2. Canonical Serialization Test Vector

### 2.1 Input Cell

```
kind: Weight (0x01)
representation: FP32 (0x00000001)
schema: [32 bytes of zeros]
dependencies: []
attributes: {}
payload: [4 bytes of 0x3F800000] (FP32 value 1.0)
```

### 2.2 Canonical Byte Sequence

```
Header (28 bytes):
  01                    // kind = Weight
  01 00 00 00           // representation = FP32
  00 00 00 00           // dependency_count = 0
  00 00 00 00           // attribute_count = 0
  04 00 00 00 00 00 00 00  // payload_size = 4
  00 00 00 00 00 00 00  // reserved

Metadata:
  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  // schema (32 bytes)
  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

Payload:
  00 00 80 3F            // FP32 1.0 (little-endian)
```

### 2.3 Expected CellId

```
CellId = BLAKE3-256(above bytes)
= [calculated from canonical sequence]
```

---

## 3. BLAKE3-256 Hash Test Vector

### 3.1 Input

```
Data: "NCL" (3 bytes)
```

### 3.2 Expected Hash

```
BLAKE3-256("NCL") = [compute using BLAKE3 reference implementation]
```

### 3.3 Verification

```python
import blake3
hash = blake3.blake3(b"NCL").digest()
assert len(hash) == 32
```

---

## 4. Column Encoding Test Vectors

### 4.1 RAW Encoding

**Input:** [1, 2, 3, 4, 5] (u32)
**Expected:** `01 00 00 00 02 00 00 00 03 00 00 00 04 00 00 00 05 00 00 00`

### 4.2 RLE Encoding

**Input:** [5, 5, 5, 5, 5] (u32)
**Expected:** `05 00 00 00 05 00 00 00` (count=5, value=5)

### 4.3 DELTA Encoding

**Input:** [10, 20, 30, 40, 50] (u32)
**Expected:**
```
base: 0A 00 00 00       // 10
delta_1: 14             // ZigZag(10) = 20
delta_2: 14             // ZigZag(10) = 20
delta_3: 14             // ZigZag(10) = 20
delta_4: 14             // ZigZag(10) = 20
```

### 4.4 BITPACK Encoding

**Input:** [0, 1, 0, 1, 0, 1] (boolean)
**Expected:**
```
bit_width: 01           // 1 bit per value
bitstream: 2A           // 010101 binary = 0x2A
```

### 4.5 VARINT Encoding

**Input:** 300
**Expected:** `AC 02` (300 = 2 × 128 + 44)

### 4.6 DICT Encoding

**Input:** ["apple", "banana", "apple", "cherry", "banana"]
**Expected:**
```
dict_size: 03 00 00 00  // 3 unique values
dict_entries:
  05 00                 // key_len = 5
  61 70 70 6C 65        // "apple"
  06 00                 // key_len = 6
  62 61 6E 61 6E 61     // "banana"
  06 00                 // key_len = 6
  63 68 65 72 72 79     // "cherry"
indices:
  00                    // apple (index 0)
  01                    // banana (index 1)
  00                    // apple (index 0)
  02                    // cherry (index 2)
  01                    // banana (index 1)
```

### 4.7 SPARSE Encoding

**Input:** [0, 0, 5, 0, 0, 0, 3, 0] (u32)
**Expected:**
```
total_count: 08 00 00 00 00 00 00 00  // 8
sparse_count: 02 00 00 00 00 00 00 00  // 2
index_0: 02 00 00 00 00 00 00 00  // index 2
value_0: 05 00 00 00              // value 5
index_1: 06 00 00 00 00 00 00 00  // index 6
value_1: 03 00 00 00              // value 3
```

---

## 5. Integrity Test Vector

### 5.1 Block Checksum

**Input:** Block data = [0x4E, 0x43, 0x4C]
**Expected CRC32:** [compute using IEEE polynomial]

### 5.2 Cell Hash Verification

**Input:** Canonical cell bytes (from §2.2)
**Expected:** CellId matches BLAKE3-256 of canonical bytes

---

## 6. Compression Test Vector

### 6.1 Zstandard Compression

**Input:** [repeated pattern] (highly compressible)
**Expected:** Compressed size < input size
**Decompressed:** Must match original exactly

### 6.2 Incompressible Data

**Input:** [random bytes]
**Expected:** codec = 0x00 (None), compressed_size = uncompressed_size

---

## 7. Quantization Test Vector

### 7.1 INT8 Quantization

**Input:** FP32 array [1.0, 2.0, 3.0, 4.0]
**Scale:** 0.015625 (1/64)
**Zero-point:** 0
**Expected INT8:** [64, 128, -128, 0] (approximate)

### 7.2 INT4 Quantization

**Input:** FP32 array [0.0, 0.25, 0.5, 0.75, 1.0, -0.25, -0.5, -0.75]
**Expected INT4 (packed):** [0x00, 0x24, 0x46, 0x8A] (approximate)

---

## 8. Reader/Writer Test Vector

### 8.1 Write-Read Roundtrip

**Write:**
```
1. Create NCL file
2. Write superblock
3. Write manifest
4. Write 10 cells
5. Write column data
6. Write payload blocks
7. Write index
8. Commit
```

**Read:**
```
1. Open NCL file
2. Validate superblock
3. Load index
4. Read each cell
5. Verify cell hash
6. Verify payload
```

**Expected:** All cells read successfully, all hashes match.

### 8.2 Selective Read

**Write:** 100 cells
**Read:** Cell at index 50
**Expected:** Only block containing cell 50 is read (not all 100 cells)

---

## 9. Revision Test Vector

### 9.1 Revision Chain

**R1:** {C1, C2, C3, C4}
**R2 (parent: R1):** {C3'}
**Resolved R2:** {C1, C2, C3', C4}

### 9.2 Revision Hash

**Input:** R2 canonical bytes
**Expected:** R2.revision_id = BLAKE3-256(canonical bytes)

---

## 10. Conformance Requirements

1. All test vectors MUST be passable by conformant implementations.
2. Test vectors MUST be deterministic.
3. Test vectors MUST cover all major operations.
4. Test vectors MUST be updated when specification changes.
