# NCI Cell Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §4–§5; `01-NCI-TERMINOLOGY.md`; `02-NCI-ARCHITECTURE.md`

---

## Purpose

This document specifies the Cell data model in complete detail: logical structure, physical storage, identity formation, Cell kinds, size constraints, and the rules governing Cell creation, composition, and lifecycle.

## Scope

Cell logical definition, physical columnar representation, identity contract, CellKind taxonomy, size rules, attribute sets, dependency semantics, and lifecycle.

---

## 1. Logical Cell Definition

### 1.1 Cell Structure

```rust
struct Cell {
    id: CellId,                      // 32-byte BLAKE3-256
    kind: CellKind,                  // semantic type (1 byte)
    schema: SchemaId,                // structural schema (32 bytes)
    representation: RepresentationId, // encoding variant (4 bytes)
    payload: PayloadRef,             // immutable data reference (16 bytes)
    dependencies: Vec<CellId>,       // upstream Cell IDs (32 bytes each)
    attributes: AttributeSet,        // metadata tags (variable)
}
```

### 1.2 Field Specifications

#### CellId

| Property | Value |
|---|---|
| Size | 32 bytes |
| Type | `u8[32]` |
| Encoding | BLAKE3-256 hash |
| Formation | `BLAKE3-256(canonical_cell_header \|\| canonical_cell_metadata \|\| canonical_uncompressed_payload)` |
| Textual | `b3:` + 64 hex chars, lowercase |
| Immutability | Immutable once committed |

**Validation:**
- MUST be exactly 32 bytes
- MUST be the BLAKE3-256 of the canonical payload
- Hash mismatch with stored payload MUST trigger `NCI-E-CORRUPT`

#### CellKind

| Property | Value |
|---|---|
| Size | 1 byte |
| Type | `u8` enum |
| Encoding | Raw byte, little-endian |

**Valid Values (Contract §4.4):**

| Value | Kind | Description |
|---|---|---|
| 0x01 | Weight | Generic weight tensor |
| 0x02 | Embedding | Embedding matrix |
| 0x03 | AttentionQ | Query projection |
| 0x04 | AttentionK | Key projection |
| 0x05 | AttentionV | Value projection |
| 0x06 | AttentionO | Output projection |
| 0x07 | MlpGate | MLP gate projection |
| 0x08 | MlpUp | MLP up-projection |
| 0x09 | MlpDown | MLP down-projection |
| 0x0A | Expert | Expert in mixture-of-experts |
| 0x0B | ExpertRouter | Router for mixture-of-experts |
| 0x0C | Normalization | Layer/rms normalization |
| 0x0D | Encoder | Encoder component |
| 0x0E | Decoder | Decoder component |
| 0x0F | Memory | Generic memory |
| 0x10 | SemanticMemory | Factual/relational memory |
| 0x11 | EpisodicMemory | Event/experience memory |
| 0x12 | ProceduralMemory | Pattern/routine memory |
| 0x13 | Routing | Routing decision |
| 0x14 | Composition | Composition relationship |
| 0x15 | Graph | Graph structure |
| 0x16 | Index | Index structure |
| 0x17 | Execution | Execution state |
| 0x18 | Policy | Policy definition |
| 0x19 | Metadata | Metadata |
| 0x1A | Provenance | Provenance/lineage |
| 0x1B–0xFE | Reserved | MUST NOT be used |
| 0xFF | Custom | Implementation-defined |

**Validation:**
- MUST be one of the values above (0x01–0x1B, 0xFF)
- Values 0x1B–0xFE MUST be rejected with `NCI-E-FORMAT`

#### SchemaId

| Property | Value |
|---|---|
| Size | 32 bytes |
| Type | `u8[32]` |
| Encoding | BLAKE3-256 of canonical schema definition |
| Formation | `BLAKE3-256(schema_canonical_bytes)` |

**Validation:**
- MUST be exactly 32 bytes
- Schema MUST be resolvable via the Schema Index

#### RepresentationId

| Property | Value |
|---|---|
| Size | 4 bytes |
| Type | `u32` |
| Encoding | Raw, little-endian |

**Valid Ranges:**

| Value Range | Meaning |
|---|---|
| 0x00000001 | FP32 representation |
| 0x00000002 | BF16 representation |
| 0x00000003 | FP16 representation |
| 0x00000004 | FP8 representation |
| 0x00000005 | INT8 representation |
| 0x00000006 | INT4 representation |
| 0x00000007 | INT3 representation |
| 0x00000008–0x000000FF | Reserved for future quantization formats |
| 0x00000100–0xFFFFFFFF | Custom representation (content-addressed) |

**Validation:**
- MUST be non-zero
- MUST resolve via Representation Index

#### PayloadRef

| Property | Value |
|---|---|
| Size | 16 bytes |
| Type | `PayloadRef` struct |
| Encoding | Packed struct, little-endian |

```rust
#[repr(C, packed)]
struct PayloadRef {
    block_id: u32,      // 4 bytes, little-endian — block index in Payload Region
    block_offset: u32,  // 4 bytes, little-endian — byte offset within block
    payload_size: u32,  // 4 bytes, little-endian — uncompressed payload size in bytes
    flags: u32,         // 4 bytes, little-endian — compression/quantization flags
}
```

**Flag Bits:**

| Bit | Name | Meaning |
|---|---|---|
| 0 | COMPRESSED | Payload is compressed |
| 1 | DEDUPLICATED | Payload is shared with other Cells |
| 2 | QUANTIZED | Payload is quantized |
| 3–31 | Reserved | MUST be zero |

**Validation:**
- `block_id` MUST reference valid block in Payload Region
- `block_offset + payload_size` MUST NOT exceed block boundary
- `payload_size` MUST be > 0

#### Dependencies

| Property | Value |
|---|---|
| Size | Variable (32 bytes per dependency) |
| Type | `Vec<CellId>` |
| Encoding | Sorted array of 32-byte CellIds |

**Canonical Ordering:**
- Dependencies MUST be sorted lexicographically by raw byte value
- Empty dependency list encodes as zero-length array (0 bytes)

**Validation:**
- All referenced CellIds MUST exist in the model (or be resolvable via index)
- Circular dependencies MUST be detected and rejected at creation time
- Maximum dependency count: 2^32 - 1 (effectively unlimited)

#### AttributeSet

| Property | Value |
|---|---|
| Size | Variable |
| Type | Key-value pairs |
| Encoding | Dictionary-encoded, canonical key ordering |

```rust
struct AttributeSet {
    count: u32,                         // 4 bytes, little-endian
    entries: Vec<AttributeEntry>,        // sorted by key
}

struct AttributeEntry {
    key_len: u16,                        // 2 bytes, little-endian
    key: Vec<u8>,                        // UTF-8 string
    value_type: u8,                      // 1 byte
    value_len: u32,                      // 4 bytes, little-endian
    value: Vec<u8>,                      // type-specific encoding
}
```

**Value Types:**

| Type Byte | Meaning | Encoding |
|---|---|---|
| 0x00 | Null | No value bytes |
| 0x01 | Bool | 1 byte (0x00=false, 0x01=true) |
| 0x02 | U64 | 8 bytes, little-endian |
| 0x03 | I64 | 8 bytes, little-endian, zigzag |
| 0x04 | F64 | 8 bytes, little-endian |
| 0x05 | String | UTF-8 bytes |
| 0x06 | Bytes | Raw bytes |
| 0x07 | CellId | 32 bytes |
| 0x08 | List | Length-prefixed list of same-type values |

**Canonical Ordering:**
- Attributes MUST be sorted by key (lexicographic byte order)
- Duplicate keys MUST be rejected with `NCI-E-FORMAT`

---

## 2. Physical Cell Storage (Columnar)

### 2.1 Column Mapping

Cells are stored in columnar form. Each logical Cell field maps to one or more columns:

```
Cell Logical Field        Column Name              Column Type
────────────────────────────────────────────────────────────────────
id                       CELL_ID                  RAW [32B × N]
kind                     CELL_TYPE                RAW or RLE [1B × N]
schema                   SCHEMA_COLUMN            RAW or RLE [32B × N]
representation           REPRESENTATION_COLUMN    RAW or RLE [4B × N]
payload.block_id         PAYLOAD_OFFSET           VARINT DELTA
payload.block_offset     (embedded in PAYLOAD_OFFSET)
payload.payload_size     PAYLOAD_SIZE             VARINT
payload.flags            FLAGS                    BITPACK
dependencies (count)     DEPENDENCY_COUNT         VARINT
dependencies (offset)    DEPENDENCY_OFFSET        VARINT DELTA
dependencies (data)      DEPENDENCY_DATA          RAW [32B per]
attributes               ATTRIBUTE_COLUMN         Variable
```

### 2.2 Column Layout Rules

1. All columns in the Cell Column Region MUST be indexed by row index (0-based).
2. Row index `i` corresponds to Cell `i` across all columns.
3. Columns with variable-length data (DEPENDENCY_DATA, ATTRIBUTE_COLUMN) use offset+length indirection.
4. Column data MUST be contiguous within the Cell Column Region.
5. Column headers (type, encoding, length) are stored in the Region Header.

### 2.3 Region Header

Each column in the Cell Column Region has a header:

```rust
#[repr(C, packed)]
struct ColumnHeader {
    column_type: u16,       // 2 bytes — column type enum (§2.1)
    encoding: u8,           // 1 byte — encoding type
    element_size: u32,      // 4 bytes — bytes per element (0 for variable)
    element_count: u64,     // 8 bytes — number of elements
    data_offset: u64,       // 8 bytes — offset from region start
    data_size: u64,         // 8 bytes — total data size in bytes
    checksum: u32,          // 4 bytes — CRC32 of column data
}
// Total: 35 bytes per column header
```

---

## 3. Cell Size Contract

### 3.1 Size Bounds

| Bound | Value | Enforcement |
|---|---|---|
| Minimum | 64 KiB (65,536 bytes) | Writer MUST reject smaller Cells |
| Target | 4 MiB (4,194,304 bytes) | Cell Planner SHOULD target this size |
| Maximum | 512 MiB (536,870,912 bytes) | Writer MUST reject larger Cells |

### 3.2 Size Determination

Cell size is determined by the Cell Planner based on:

1. **Tensor locality and shape** — Adjacent tensor elements SHOULD be co-located
2. **Compute access patterns** — Frequently co-accessed data SHOULD be in the same Cell
3. **Quantization overhead** — Quantized and unquantized versions MUST be separate Representations, not separate Cells
4. **Cache size** — Cell size SHOULD fit target cache level
5. **Hardware capability** — Target hardware memory constraints
6. **Dependency graph topology** — Highly connected Cells SHOULD be co-located
7. **Access frequency** — Hot Cells SHOULD be smaller for faster loading

### 3.3 Semantic Boundaries

Cell boundaries MUST be semantically meaningful. Random chunking of tensors is forbidden. The Cell Planner MUST respect:

- Layer boundaries in transformer architectures
- Attention head boundaries
- Expert boundaries in mixture-of-experts
- Embedding table boundaries
- Normalization parameter boundaries

---

## 4. Cell Lifecycle

### 4.1 States

```
PLANNING → CANONICALIZING → HASHING → COMPRESSING → WRITING → COMMITTED
                                                                    ↓
                                                              IMMUTABLE (permanent)
```

### 4.2 State Transitions

| From | To | Operation |
|---|---|---|
| PLANNING | CANONICALIZING | Cell Planner determines Cell boundaries and content |
| CANONICALIZING | HASHING | Canonical byte sequence produced |
| HASHING | COMPRESSING | BLAKE3-256 identity computed |
| COMPRESSING | WRITING | Compression applied (or skipped) |
| WRITING | COMMITTED | Atomic commit to NCL file |
| COMMITTED | IMMUTABLE | Permanent; no further transitions |

### 4.3 Immutability Guarantee

Once COMMITTED:
- Cell data MUST NOT be modified
- Cell identity MUST NOT change
- Cell MUST NOT be deleted (only garbage-collected via revision)
- New versions of Cell content produce new CellIds

---

## 5. Cell Composition

### 5.1 Cell Groups

A Cell MAY compose multiple sub-Cells:

```rust
struct CellGroup {
    group_id: CellId,
    members: Vec<CellId>,
    composition_type: CompositionType,
}

enum CompositionType {
    Sequential,     // Members execute in order
    Parallel,       // Members execute concurrently
    Conditional,    // Members execute based on condition
    Recursive,      // Members execute recursively
}
```

### 5.2 Composition Rules

1. Group Cell MUST have CellKind::Composition
2. Members MUST exist before group is committed
3. Circular composition is forbidden
4. Group dependencies include all member dependencies (transitive closure)

---

## 6. Cell Attributes

### 6.1 Standard Attributes

Implementations SHOULD use these standard attribute keys:

| Key | Type | Description |
|---|---|---|
| `nci.layer_index` | U64 | Layer index in model |
| `nci.head_index` | U64 | Attention head index |
| `nci.expert_index` | U64 | Expert index in MoE |
| `nci.tensor_name` | String | Original tensor name |
| `nci.source_format` | String | Source format (pytorch/safetensors/gguf) |
| `nci.quantization_group_size` | U64 | Quantization group size |
| `nci.importance_score` | F64 | Importance for routing |
| `nci.access_frequency` | F64 | Historical access frequency |
| `nci.hardware_target` | String | Target hardware (cpu/gpu/npu) |

### 6.2 Custom Attributes

Implementations MAY define custom attributes with keys prefixed by reverse-domain convention:

| Key Pattern | Description |
|---|---|
| `com.{org}.{project}.{key}` | Organization-specific attribute |

---

## 7. Validation

### 7.1 Writer Validation

Writer MUST validate before commit:

1. Cell size within bounds (64 KiB–512 MiB)
2. CellKind is valid enum value
3. SchemaId resolves to valid schema
4. RepresentationId resolves to valid representation
5. PayloadRef points to valid block with sufficient space
6. Dependencies are sorted and exist
7. Attributes have sorted keys and valid value types
8. Canonical serialization produces expected byte sequence
9. BLAKE3-256 identity matches computed hash

### 7.2 Reader Validation

Reader MUST validate before release to runtime:

1. CellId matches stored hash
2. Payload decompresses to expected size
3. Decompressed payload hash matches CellId
4. All dependencies exist and are valid
5. Schema is compatible with reader version

---

## 8. Error Handling

| Condition | Error Code | Action |
|---|---|---|
| Cell size < 64 KiB | `NCI-E-FORMAT` | Reject Cell |
| Cell size > 512 MiB | `NCI-E-FORMAT` | Reject Cell |
| Invalid CellKind | `NCI-E-FORMAT` | Reject Cell |
| Schema not found | `NCI-E-SCHEMA` | Reject Cell |
| Representation not found | `NCI-E-REPRESENTATION` | Reject Cell |
| Dependency not found | `NCI-E-NOTFOUND` | Reject Cell |
| Circular dependency | `NCI-E-FORMAT` | Reject Cell |
| Duplicate attribute key | `NCI-E-FORMAT` | Reject Cell |
| Hash mismatch | `NCI-E-CORRUPT` | Quarantine Cell |
| Payload not found | `NCI-E-NOTFOUND` | Quarantine Cell |

---

## 9. Compatibility

### 9.1 Version Compatibility

- Major version change: Cell structure MAY change (breaking)
- Minor version change: New CellKinds MAY be added (compatible)
- Patch version: Clarifications only (compatible)

### 9.2 Forward Compatibility

Readers MUST handle unknown CellKinds gracefully:
- Unknown CellKinds SHOULD be treated as opaque data
- Unknown Attributes MUST be preserved but MAY be ignored
- Unknown Representations MUST trigger `NCI-E-REPRESENTATION`

### 9.3 Backward Compatibility

Readers MUST NOT require fields that did not exist in earlier versions. All new fields MUST have default values or MUST be optional.

---

## 10. Conformance Requirements

1. Every Cell in an NCL file MUST be reconstructable from columnar data.
2. Cell identity MUST be reproducible across implementations given identical canonical payload.
3. Writer MUST reject Cells violating size or kind constraints.
4. Reader MUST quarantine Cells with hash mismatches.
5. All CellKinds 0x01–0x1A MUST be supported by conformant implementations.
6. CellKind 0xFF (Custom) MUST be handled opaquely.
