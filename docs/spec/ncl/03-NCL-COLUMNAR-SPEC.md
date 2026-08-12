# NCL Columnar Storage Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §6; `ncl/01-NCL-BINARY-SPEC.md`

---

## Purpose

This document specifies the columnar storage layer of NCL: column types, encoding algorithms, column operations, and the physical layout of column data within NCL regions.

## Scope

Column type taxonomy, encoding algorithms (RAW, RLE, DELTA, DELTA_RLE, BITPACK, VARINT, DICT, SPARSE), column operations (scan, filter, project, join), and column region physical layout.

---

## 1. Column Type Taxonomy

### 1.1 Standard Column Types

| ID | Name | Element Size | Encoding Options | Region |
|---|---|---|---|---|
| 0x01 | CELL_ID | 32 bytes fixed | RAW | Cell |
| 0x02 | CELL_TYPE | 1 byte | RAW, RLE | Cell |
| 0x03 | SCHEMA_ID | 32 bytes fixed | RAW, RLE | Cell |
| 0x04 | REPRESENTATION_ID | 4 bytes | RAW, RLE | Cell |
| 0x05 | DEPENDENCY_OFFSET | Variable (VarInt) | VARINT DELTA | Cell |
| 0x06 | DEPENDENCY_COUNT | Variable (VarInt) | VARINT | Cell |
| 0x07 | DEPENDENCY_DATA | 32 bytes per | RAW | Cell |
| 0x08 | PAYLOAD_OFFSET | Variable (VarInt) | VARINT DELTA | Cell |
| 0x09 | PAYLOAD_SIZE | Variable (VarInt) | VARINT | Cell |
| 0x0A | PAYLOAD_CODEC | 1 byte | RAW | Cell |
| 0x0B | GRAPH_SOURCE | 32 bytes fixed | RAW | Graph |
| 0x0C | GRAPH_TARGET | 32 bytes fixed | RAW | Graph |
| 0x0D | GRAPH_RELATION | 1 byte | RAW, RLE | Graph |
| 0x0E | GRAPH_WEIGHT | 4 bytes | RAW | Graph |
| 0x0F | ROUTE_QUERY | Variable | RAW, DICT | Routing |
| 0x10 | ROUTE_TARGET | 32 bytes fixed | RAW | Routing |
| 0x11 | ROUTE_SCORE | 2 or 4 bytes | RAW | Routing |
| 0x12 | MEMORY_ID | 32 bytes fixed | RAW | Memory |
| 0x13 | MEMORY_TYPE | 1 byte | RAW, RLE | Memory |
| 0x14 | MEMORY_IMPORTANCE | 2 bytes | RAW | Memory |
| 0x15 | MEMORY_TIMESTAMP | Variable (VarInt) | VARINT | Memory |
| 0x16 | FLAGS | Bit-packed | BITPACK | Cell |

---

## 2. Encoding Algorithms

### 2.1 RAW Encoding

Unencoded binary data. No transformation.

```
Writer:
  write(element_bytes)

Reader:
  read(element_size) → element
```

**Use when:** Data has high entropy or is already compact.

### 2.2 RLE (Run-Length Encoding)

Compresses runs of identical values.

**Write:**
```
elements = [...]
runs = []
current_run = { value: elements[0], count: 1 }

for i in 1..elements.len():
    if elements[i] == current_run.value:
        current_run.count += 1
    else:
        runs.append(current_run)
        current_run = { value: elements[i], count: 1 }

runs.append(current_run)

// Encode runs
for run in runs:
    write_varint(run.count)
    write(run.value)
```

**Read:**
```
runs = []
while has_more():
    count = read_varint()
    value = read(element_size)
    runs.append({ count, value })

elements = []
for run in runs:
    elements.extend(repeat(run.value, run.count))
```

**Use when:** Column has long runs of identical values (e.g., repeated CellKind, repeated SchemaId).

### 2.3 DELTA Encoding

Compresses monotonically increasing sequences.

**Write:**
```
base = elements[0]
write(base)

prev = base
for i in 1..elements.len():
    delta = elements[i] - prev
    write_zigzag_varint(delta)
    prev = elements[i]
```

**Read:**
```
base = read(element_size)
elements = [base]
prev = base

while has_more():
    delta = read_zigzag_varint()
    value = prev + delta
    elements.append(value)
    prev = value
```

**Use when:** Column values increase monotonically (e.g., offsets, indices).

### 2.4 DELTA_RLE Encoding

Applies DELTA encoding, then RLE on the delta stream.

**Write:**
```
deltas = compute_deltas(elements)
rle_encode(deltas)
```

**Read:**
```
deltas = rle_decode()
elements = reconstruct_from_deltas(deltas)
```

**Use when:** Delta values have many repeats (e.g., regular spacing).

### 2.5 BITPACK Encoding

Packs values using minimal bits.

**Write:**
```
max_value = max(elements)
bit_width = ceil(log2(max_value + 1))

write_u8(bit_width)
bitstream = BitWriter()

for value in elements:
    bitstream.write(value, bit_width)

write(bitstream.bytes())
```

**Read:**
```
bit_width = read_u8()
bitstream = BitReader(read_bytes())

for i in 0..element_count:
    value = bitstream.read(bit_width)
    elements.append(value)
```

**Use when:** Values have small range (e.g., boolean flags, small enums).

### 2.6 VARINT Encoding

Variable-length unsigned integer encoding.

**Write:**
```
fn write_varint(value: u64):
    while value >= 0x80:
        write_u8((value & 0x7F) | 0x80)
        value >>= 7
    write_u8(value as u8)
```

**Read:**
```
fn read_varint() -> u64:
    result = 0
    shift = 0
    loop:
        byte = read_u8()
        result |= (byte & 0x7F) << shift
        shift += 7
        if byte & 0x80 == 0:
            break
    return result
```

**Use when:** Values are small or sparse (e.g., counts, offsets).

### 2.7 DICT Encoding

Dictionary encoding for low-cardinality columns.

**Write:**
```
unique_values = sorted(set(elements))
dictionary = { value: index for index, value in enumerate(unique_values) }

write_u32(len(unique_values))
for value in unique_values:
    write(value)

for element in elements:
    write_varint(dictionary[element])
```

**Read:**
```
dict_size = read_u32()
dictionary = []
for i in 0..dict_size:
    dictionary.append(read(element_size))

elements = []
for i in 0..element_count:
    index = read_varint()
    elements.append(dictionary[index])
```

**Use when:** Column has few unique values repeated many times (e.g., CellKind with many Cells of same kind).

### 2.8 SPARSE Encoding

Sparse representation for mostly-empty columns.

**Write:**
```
non_default = [(i, value) for i, value in enumerate(elements) if value != default]

write_u64(total_count)
write_u64(len(non_default))
for (index, value) in non_default:
    write_u64(index)
    write(value)
```

**Read:**
```
total_count = read_u64()
sparse_count = read_u64()
elements = [default] * total_count

for i in 0..sparse_count:
    index = read_u64()
    value = read(element_size)
    elements[index] = value
```

**Use when:** Most elements are default value (0, 0.0, etc.).

---

## 3. Column Operations

### 3.1 Scan

Sequential read of all elements in a column.

```
complexity: O(N)
```

### 3.2 Filter

Select elements matching a predicate.

```
fn filter(column: Column, predicate: impl Fn(&Element) -> bool) -> Vec<Element>:
    return [e for e in column.scan() if predicate(e)]
```

### 3.3 Project

Select a subset of columns by index.

```
fn project(row_index: u64, columns: &[ColumnId]) -> Row:
    return [column[row_index] for column in columns]
```

### 3.4 Join

Reconstruct logical Cell by joining columns on row index.

```
fn join(row_index: u64) -> Cell:
    return Cell {
        id: cell_id_column[row_index],
        kind: cell_type_column[row_index],
        schema: schema_column[row_index],
        representation: representation_column[row_index],
        // ... etc
    }
```

---

## 4. Column Region Physical Layout

### 4.1 Region Structure

```
┌──────────────────────────────────┐
│ ColumnRegionHeader (54 bytes)     │
├──────────────────────────────────┤
│ ColumnEntry[0] (35 bytes)        │
│ ColumnEntry[1] (35 bytes)        │
│ ...                              │
│ ColumnEntry[N-1] (35 bytes)      │
├──────────────────────────────────┤
│ Column Data[0]                   │
│ Column Data[1]                   │
│ ...                              │
│ Column Data[N-1]                 │
└──────────────────────────────────┘
```

### 4.2 Column Data Alignment

Each column data block MUST be aligned to 8-byte boundary. Padding bytes (0x00) are inserted as needed.

### 4.3 Column Data Contiguity

Column data blocks SHOULD be contiguous. Implementations MAY reorder columns for optimization, but the ColumnEntry data_offset MUST correctly reference the actual location.

---

## 5. Encoding Selection Algorithm

### 5.1 Writer Selection Rules

Writer SHOULD select encoding based on data characteristics:

```
fn select_encoding(column: &[Element]) -> Encoding:
    unique_count = len(set(column))
    
    if unique_count == 1:
        return RLE  // Single value, perfect run
    
    if unique_count <= max(16, len(column) / 100):
        return DICT  // Low cardinality
    
    if is_monotonically_increasing(column):
        delta_rle_size = estimate_delta_rle(column)
        raw_size = len(column) × element_size
        if delta_rle_size < raw_size × 0.7:
            return DELTA_RLE
    
    if max(column) < (1 << 32):
        bitpack_size = estimate_bitpack(column)
        raw_size = len(column) × element_size
        if bitpack_size < raw_size × 0.7:
            return BITPACK
    
    sparsity = count_default(column) / len(column)
    if sparsity > 0.8:
        return SPARSE
    
    return RAW  // Fallback
```

### 5.2 Reader Support

Reader MUST support ALL encodings defined in this specification. Reader MUST NOT assume a particular encoding for any column type.

---

## 6. Validation

1. Column data MUST NOT exceed declared data_size.
2. RLE counts MUST sum to element_count.
3. DELTA sequences MUST be decodable without overflow.
4. BITPACK bit_width MUST be ≤ 32.
5. DICT indices MUST be < dict_size.
6. SPARSE indices MUST be < total_count.
7. Column checksum MUST match CRC32 of column data.

---

## 7. Error Handling

| Condition | Error Code |
|---|---|
| RLE count sum != element_count | `NCI-E-CORRUPT` |
| DELTA overflow | `NCI-E-CORRUPT` |
| BITPACK invalid bit_width | `NCI-E-CORRUPT` |
| DICT index out of bounds | `NCI-E-CORRUPT` |
| SPARSE index out of bounds | `NCI-E-CORRUPT` |
| Column checksum mismatch | `NCI-E-CORRUPT` |
| Unknown column type | `NCI-E-FORMAT` |

---

## 8. Conformance Requirements

1. Writer MUST support all 8 encoding types.
2. Reader MUST support all 8 encoding types.
3. Encoding selection MUST be deterministic given identical input.
4. All encodings MUST be reversible (lossless).
5. Column checksum MUST be CRC32.
6. Column data MUST be decodable independently of other columns.
