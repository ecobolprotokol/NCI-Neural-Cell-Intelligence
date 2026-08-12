# NCI — Neural Cell Intelligence
## Engineering Contract: Final Technical & Architecture Specification
**Version 1.0.0 — Binding Specification**

---

## PREAMBLE

This document establishes the final, binding technical specification for NCI (Neural Cell Intelligence). This specification is authoritative, normative, and legally binding for all implementations, development, code review, testing, validation, interoperability verification, and maintenance of NCI systems and products. No deviations from requirements marked as MUST, SHALL, or MUST NOT are permitted without formal specification amendment through the NCI Change Control Board.

This is not a roadmap. This is not a phased plan. This is not a proposal or brainstorm document. Every clause herein is a binding engineering contract.

---

## 1. PROJECT IDENTITY & CANONICAL ARTIFACTS

### 1.1 Project Name and Scope
- **Official Name:** NCI — Neural Cell Intelligence
- **Definition:** Edge-AI architecture using Cell as the universal unit of intelligence and columnar storage as the organizing principle across all system layers.
- **Canonical Format:** NCL (Neural Cell Language/Format)
- **File Extension:** .ncl
- **Architectural Model:** Universal Cell + Columnar Storage

### 1.2 Canonical Representation
- **Primary Language:** Rust
- **Identity Scheme:** BLAKE3-256 content addressing
- **Storage Model:** Columnar Cell Store
- **Runtime Model:** Dynamic Cell Runtime
- **Model Representation:** Cell Graph
- **Memory Model:** Persistent Cell Memory
- **Versioning Model:** Cell-level Revision Directed Acyclic Graph (DAG)

### 1.3 Format Authority
The canonical, machine-readable format for all NCI intelligence MUST be .ncl. No external format (PyTorch checkpoint, Safetensors, GGUF, or other) SHALL be required by the NCI runtime for execution. All conversion from external formats to .ncl is mandatory for any model consumed by NCI systems.

---

## 2. FUNDAMENTAL PRINCIPLES

NCI is governed by twelve (12) mandatory architectural principles. Each principle is binding. Implementation MUST satisfy all twelve principles.

### NCI-P01: Universal Cell
All intelligence, computation, metadata, routing, memory, state, and graph structure MUST be represented as Cell objects. There are no exceptions to this principle.

### NCI-P02: Columnar Everything
All data that can be accessed or modified independently MUST be organized in columnar format. Mixed row-oriented and columnar storage is forbidden.

### NCI-P03: Content Addressing
All Cell identity and immutable object identity MUST use BLAKE3-256 cryptographic hashing of the canonical uncompressed payload. Identity MUST be deterministic and independent of physical location, storage filename, or offset.

### NCI-P04: Immutable Core
Once a Cell is committed to NCL, it MUST NOT be modified. Mutations produce new Cells with new identities. Immutability is a hard guarantee.

### NCI-P05: Selective Activation
The NCI runtime MUST activate only Cells required for the current computation. Full model residency is not required and is explicitly not mandated.

### NCI-P06: Dynamic Computation
Computation is NOT determined solely by static architecture. The execution path MUST adapt to input, routing decisions, difficulty estimation, and compute budget.

### NCI-P07: Persistent Memory
Memory is part of persistent intelligence, not a transient cache. Memory Cells MUST survive across sessions and support incremental refinement.

### NCI-P08: Streaming-First
All conversion, I/O, hashing, and format transformation MUST support streaming without requiring the entire model to be resident in RAM simultaneously.

### NCI-P09: Zero External Format Coupling
The NCI runtime MUST NOT depend on external checkpoint formats. The runtime MUST only understand NCL. No runtime code path SHALL directly parse PyTorch, Safetensors, GGUF, or any other external format.

### NCI-P10: Hardware Independence
Cell Store representation MUST NOT be bound to specific CPU, GPU, or NPU architectures. Hardware-specific Representations MAY exist, but they MUST NOT be required for correctness.

### NCI-P11: Incremental Evolution
Model changes MUST occur at Cell granularity. Entire model retraining is not required. Cell refinement, routing updates, and memory consolidation are the unit of change.

### NCI-P12: Data Movement First
Primary optimization focus MUST be on minimizing bytes moved per useful computation, not merely on reducing FLOPs. Memory bandwidth efficiency is the driving design constraint.

---

## 3. GLOBAL ARCHITECTURE

NCI consists of three (3) logical layers unified by columnar storage:

```
NCI ARCHITECTURE
                      │
        ┌─────────────┴─────────────┐
        │  Universal Cell Model     │
        └─────────────┬─────────────┘
                      │
    ┌─────────────────┼─────────────────┐
    ▼                 ▼                 ▼
Cell Layer      Column Layer       Graph Layer
    │                 │                 │
    └─────────────────┼─────────────────┘
                      │
                NCL Storage Layer
                      │
    ┌─────────────────┼─────────────────┐
    ▼                 ▼                 ▼
Routing          Memory              Revision
Layer            Layer               Layer
    │                 │                 │
    └─────────────────┼─────────────────┘
                      ▼
                NCI Runtime
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
      CPU           GPU            NPU
      Backend       Backend         Backend
```

Each layer MUST respect columnar organization. Each layer MUST be composable through the Cell abstraction.

---

## 4. UNIVERSAL CELL MODEL

### 4.1 Logical Definition
A Cell is the primitive unit of NCI intelligence. The logical structure is:

```rust
struct Cell {
    id: CellId,                    // 32-byte BLAKE3-256
    kind: CellKind,                // semantic type
    schema: SchemaId,              // structural schema
    representation: RepresentationId, // encoding variant
    payload: PayloadRef,           // immutable data reference
    dependencies: Vec<CellId>,     // upstream Cell IDs
    attributes: AttributeSet,      // metadata tags
}
```

### 4.2 Physical Storage
Cells MUST NOT necessarily be stored as contiguous structures. Instead, the columnar representation is mandatory:

```
CELL_ID_COLUMN          // [32B × N]
CELL_TYPE_COLUMN        // [1B × N]
SCHEMA_COLUMN           // [32B × N]
REPRESENTATION_COLUMN   // [4B × N]
DEPENDENCY_COLUMN       // [Variable]
PAYLOAD_REF_COLUMN      // [16B × N]
ATTRIBUTE_COLUMN        // [Variable]
```

The logical Cell is reconstructed by joining these columns on row index.

### 4.3 Cell Size Contract
- **Minimum:** 64 KiB
- **Target:** 4 MiB
- **Maximum:** 512 MiB

Cell size is NOT arbitrary. The Cell Planner MUST determine Cell boundaries based on:
- Tensor locality and shape
- Compute access patterns
- Quantization and compression overhead
- Available cache size
- Hardware capability
- Dependency graph topology
- Access frequency and recency

A Cell MUST have semantic boundary. Random chunking is forbidden.

### 4.4 Supported Cell Kinds
The following Cell kinds are mandatory support:

```
Weight, Embedding,
AttentionQ, AttentionK, AttentionV, AttentionO,
MlpGate, MlpUp, MlpDown,
Expert, ExpertRouter,
Normalization,
Encoder, Decoder,
Memory, SemanticMemory, EpisodicMemory, ProceduralMemory,
Routing, Composition,
Graph, Index,
Execution, Policy,
Metadata, Provenance,
Custom
```

The runtime MUST NOT assume Cell contains only weights. Any semantic type is valid.

---

## 5. CELL IDENTITY CONTRACT

### 5.1 Identity Formation
Every Cell MUST have a unique 32-byte BLAKE3-256 identity derived from its canonical uncompressed payload:

```
CellId = BLAKE3-256(
    canonical_cell_header
    || canonical_cell_metadata
    || canonical_uncompressed_payload
)
```

### 5.2 Textual Representation
Cell identity in human-readable form MUST use the prefix `b3:` followed by 64 hexadecimal characters:

```
b3:7a0f...9c21
```

### 5.3 Identity Invariants
The following invariants are non-negotiable:

1. **Content-Derived:** Identity MUST derive only from canonical payload content.
2. **Compression Transparent:** Compression/decompression MUST NOT alter identity.
3. **Location Independent:** Physical storage location MUST NOT affect identity.
4. **Filename Independent:** NCL filename MUST NOT determine identity.
5. **Offset Independent:** Source offset in checkpoint MUST NOT affect identity.
6. **Immutable:** Cell identity is immutable once committed.
7. **Hash Verification:** Any hash mismatch with stored payload MUST trigger immediate corruption quarantine.

---

## 6. COLUMNAR ARCHITECTURE & STORAGE

### 6.1 Column Categories
NCI MUST implement the following column types:

| Column Type | Purpose | Encoding |
|---|---|---|
| CELL_ID | Cell identity reference | RAW (32B fixed) |
| CELL_TYPE | Cell semantic kind | RAW or RLE (1B) |
| SCHEMA_ID | Structural schema | RAW or RLE (32B) |
| REPRESENTATION_ID | Encoding variant | RAW or RLE (4B) |
| DEPENDENCY_OFFSET | Dependency list offset | VARINT DELTA |
| DEPENDENCY_COUNT | Dependency count | VARINT |
| DEPENDENCY_DATA | Dependency Cell IDs | RAW (32B per) |
| PAYLOAD_OFFSET | Payload block offset | VARINT DELTA |
| PAYLOAD_SIZE | Payload uncompressed size | VARINT |
| PAYLOAD_CODEC | Compression codec | RAW (1B) |
| GRAPH_SOURCE | Graph edge source | RAW (32B) |
| GRAPH_TARGET | Graph edge target | RAW (32B) |
| GRAPH_RELATION | Relationship type | RAW (1B) |
| GRAPH_WEIGHT | Edge weight | FLOAT32 |
| ROUTE_QUERY | Query token reference | RAW or DICT |
| ROUTE_TARGET | Routable Cell | RAW (32B) |
| ROUTE_SCORE | Routing score | FLOAT16 or FLOAT32 |
| MEMORY_ID | Memory Cell identity | RAW (32B) |
| MEMORY_TYPE | Memory category | RAW (1B) |
| MEMORY_IMPORTANCE | Importance score | FLOAT16 |
| MEMORY_TIMESTAMP | Access timestamp | VARINT (ms) |
| FLAGS | Cell flags (quantization, compression) | BITPACK |

### 6.2 Column Encoding Options
Writers MUST support the following encodings. Readers MUST support ALL encodings present in NCL version 1.x:

- **RAW:** Unencoded binary data
- **RLE:** Run-Length Encoding for repetitive columns
- **DELTA:** Delta-of-delta compression for monotonic sequences
- **DELTA_RLE:** Combined DELTA + RLE
- **BITPACK:** Bit-packing for low-entropy data
- **VARINT:** Variable-length integer encoding for sparse/small values
- **DICT:** Dictionary encoding for low-cardinality columns
- **SPARSE:** Sparse representation for mostly-empty columns

---

## 7. NCL BINARY FORMAT SPECIFICATION

### 7.1 Format Authority
The .ncl binary format is the canonical, machine-readable representation of NCI. It is NOT a wrapper around external checkpoints. It is NOT Safetensors + metadata. It is NOT a ZIP container. It is a new, independent binary format.

### 7.2 File Structure
An NCL file MUST contain the following regions in order:

```
┌──────────────────────────────────┐
│ NCL SUPERBLOCK (4096B fixed)     │
├──────────────────────────────────┤
│ MANIFEST REGION                  │
├──────────────────────────────────┤
│ CELL COLUMN REGION               │
├──────────────────────────────────┤
│ GRAPH COLUMN REGION              │
├──────────────────────────────────┤
│ ROUTING COLUMN REGION            │
├──────────────────────────────────┤
│ MEMORY COLUMN REGION             │
├──────────────────────────────────┤
│ REPRESENTATION REGION            │
├──────────────────────────────────┤
│ PAYLOAD REGION                   │
├──────────────────────────────────┤
│ INDEX REGION                     │
├──────────────────────────────────┤
│ REVISION REGION                  │
├──────────────────────────────────┤
│ PROVENANCE REGION                │
├──────────────────────────────────┤
│ INTEGRITY REGION                 │
└──────────────────────────────────┘
```

Region order MAY be rearranged in physical implementation provided manifest offsets and dependencies remain correct.

### 7.3 Numeric Encoding Rules
All integer encoding MUST comply with:

- **Byte Order:** Little-endian exclusively
- **Fixed-Width Types:** u8, u16, u32, u64, u128, i8, i16, i32, i64, f16, bf16, f32, f64
- **Variable-Width:** Unsigned VarInt or Signed ZigZag VarInt for efficient sparse encoding
- **Overflow Detection:** Reader MUST reject all integer overflow conditions with NCI-E-FORMAT

### 7.4 Alignment Rules
- **Default Alignment:** 8 bytes
- **Hardware-Specific:** May increase to 16, 32, or 64 bytes for CPU vectors, GPU, or NPU execution
- **Padding:** Zero-bytes only (0x00)
- **Padding Exclusion:** Padding MUST NOT be part of canonical payload for hashing

### 7.5 Superblock Structure
The NCL Superblock MUST be exactly 4096 bytes and contain:

| Offset | Size | Field | Requirement |
|---|---|---|---|
| 0 | 4 | magic | MUST be `NCL\x01` (0x4E 0x43 0x4C 0x01) |
| 4 | 2 | major_version | MUST be 1 |
| 6 | 2 | minor_version | MUST be ≥ 0 |
| 8 | 4 | flags | implementation-defined feature flags |
| 12 | 1 | endianness | MUST be 1 (little-endian) |
| 13 | 1 | hash_algorithm | MUST be 1 (BLAKE3-256) |
| 14 | 2 | header_size | MUST be 4096 |
| 16 | 8 | file_size | Total file size in bytes |
| 24 | 8 | manifest_offset | Byte offset to MANIFEST region |
| 32 | 8 | manifest_size | MANIFEST region size |
| 40 | 8 | cell_region_offset | Byte offset to CELL COLUMN region |
| 48 | 8 | graph_region_offset | Byte offset to GRAPH COLUMN region |
| 56 | 8 | routing_region_offset | Byte offset to ROUTING COLUMN region |
| 64 | 8 | memory_region_offset | Byte offset to MEMORY COLUMN region |
| 72 | 8 | representation_offset | Byte offset to REPRESENTATION region |
| 80 | 8 | payload_offset | Byte offset to PAYLOAD region |
| 88 | 8 | index_offset | Byte offset to INDEX region |
| 96 | 8 | revision_offset | Byte offset to REVISION region |
| 104 | 8 | provenance_offset | Byte offset to PROVENANCE region |
| 112 | 8 | integrity_offset | Byte offset to INTEGRITY region |
| 120 | 8 | cell_count | Total number of Cells |
| 128 | 8 | column_count | Total number of columns |
| 136 | 8 | block_count | Total number of payload blocks |
| 144 | 32 | root_hash | BLAKE3-256 of Manifest (integrity check) |
| 176 | 32 | manifest_hash | BLAKE3-256 of entire Manifest |
| 208 | (remaining) | reserved | MUST be zero-filled |

### 7.6 Magic Value Validation
Reader MUST validate magic as the first operation:

```
if superblock[0:4] != [0x4E, 0x43, 0x4C, 0x01] {
    return NCI-E-FORMAT
}
```

Failure to match MUST result in immediate rejection with NCI-E-FORMAT error.

### 7.7 Version Resolution
Version semantics:

- **Major version mismatch:** Incompatible. MUST reject with NCI-E-SCHEMA.
- **Minor version lower:** Compatible forward. Proceed.
- **Minor version higher:** MAY proceed with caution or reject depending on feature flags.

---

## 8. MANIFEST & INDEXING

### 8.1 Manifest Authority
The Manifest is the logical root that describes the entire NCL model state:

```rust
struct Manifest {
    model_id: Hash256,
    revision_id: Hash256,
    cell_count: u64,
    column_count: u64,
    graph_root: Hash256,
    routing_root: Hash256,
    memory_root: Hash256,
    representation_root: Hash256,
    index_root: Hash256,
    root_hash: Hash256,
}
```

The Manifest MUST be signed via root_hash = BLAKE3-256(manifest_content).

### 8.2 Index Guarantees
NCL MUST provide indexes enabling:

| Lookup | Complexity | Method |
|---|---|---|
| CellId → Cell descriptor | O(1) amortized | Hash table or binary search |
| CellId → Payload | O(1) amortized | Direct offset lookup |
| TileId → Block | O(1) amortized | Hash table |
| SchemaId → Schema definition | O(1) amortized | Hash table |
| RepresentationId → Metadata | O(1) amortized | Hash table |
| ColumnId → Column descriptor | O(1) amortized | Hash table |

For very large files (>1 TB), multi-level indexing MAY be used provided the target O(1) amortized lookup is preserved.

### 8.3 Index Verification
Reader MUST verify index integrity:

1. Load index header
2. Verify index hash against superblock
3. Validate pointer bounds
4. Perform spot checks on random entries
5. Reject if ANY corruption detected with NCI-E-CORRUPT

---

## 9. CANONICAL SERIALIZATION & HASHING

### 9.1 Canonicalization Requirement
Before ANY hashing, Cell content MUST be canonicalized:

```
Logical Cell
    ↓
Canonical Encoder
    ↓
Deterministic Byte Sequence
    ↓
BLAKE3-256
```

Canonicalization MUST ensure:
- Field order is fixed and globally specified
- Byte order is little-endian throughout
- NaN handling is deterministic (e.g., all NaNs canonicalize to 0xFF...FF)
- Padding is zero-filled and consistent
- Dimension encoding is standard
- Metadata field order is fixed
- Dependency ordering is deterministic (sorted)

### 9.2 Compression Below Identity
Identity MUST be computed BEFORE compression:

```
Canonical Uncompressed Payload
            ↓
        BLAKE3-256 ← identity (immutable)
            ↓
      Compression
            ↓
    Physical Block
```

NOT:

```
Canonical Payload
      ↓
 Compression
      ↓
 BLAKE3-256(compressed_bytes) ← forbidden
```

This ensures identical logical Cells have identical identities regardless of compression codec or storage efficiency choices.

### 9.3 Payload Representation Identity
A single logical Cell MAY have multiple Representations (FP16, INT8, INT4, INT3, etc.):

```
Cell C001
    ├── Representation_FP16 (identity R1)
    ├── Representation_INT8 (identity R2)
    ├── Representation_INT4 (identity R3)
    └── Representation_INT3 (identity R4)
```

Logical Cell identity remains constant. Representation identities differ. Runtime chooses appropriate Representation based on hardware and budget.

---

## 10. QUANTIZATION & COMPRESSION

### 10.1 Mandatory Quantization Support
NCI MUST support the following quantization formats:

- FP32 (full precision)
- BF16 (bfloat16)
- FP16 (half precision)
- FP8 (8-bit floating)
- INT8 (signed 8-bit integer)
- INT4 (4-bit integer)
- INT3 (3-bit integer)

Future additions require specification amendment.

### 10.2 Per-Cell Quantization
Quantization is applied per-Cell or Cell-group, NEVER required globally:

```
Attention-critical Cell    → FP16
Normal Dense Cell          → INT8
Large MLP Cell             → INT4
Rare Expert Cell           → INT3
Embedding Cell             → INT4 or FP16
Routing Cell               → FP16 or INT8
```

The model MAY use mixed quantization. The runtime MUST support dequantization to FP32 for computation if required.

### 10.3 Compression Support
NCL MUST support:

- **None:** Uncompressed
- **Zstandard (zstd):** Mandatory for all implementations
- **Implementation-approved:** Additional codecs MAY be added through codec registry with specification amendment

Compression codec MUST be specified in PAYLOAD_CODEC column.

### 10.4 Decompression Guarantees
Reader MUST decompress all payload before verification:

```
Physical Block (compressed)
        ↓
   Decompress
        ↓
Canonical Payload
        ↓
Verify BLAKE3-256
```

---

## 11. PAYLOAD & TENSOR STORAGE

### 11.1 Tensor Descriptor
Tensor Cells MUST include descriptor:

```rust
struct TensorDescriptor {
    dtype: DataType,
    rank: u8,
    dimensions: Vec<u64>,
    layout: MemoryLayout,
    quantization: QuantizationScheme,
    scale_ref: Option<Hash256>,
    zero_point_ref: Option<Hash256>,
}
```

### 11.2 Columnar Tensor Layout
Large tensors MUST NOT be stored as single contiguous blocks. Instead:

```
Tensor [4096 × 4096]
    ├── Tile [256 × 256] → Cell T1
    ├── Tile [256 × 256] → Cell T2
    ├── Tile [256 × 256] → Cell T3
    └── ... (256 tiles total)
```

Tiling reduces I/O footprint and enables selective loading.

### 11.3 Tile Descriptor
Each tile MUST have:

```rust
struct TileDescriptor {
    tile_id: Hash256,
    cell_id: Hash256,
    shape: Vec<u64>,
    offset: Vec<u64>,
    dtype: DataType,
    representation: RepresentationId,
    payload_block: BlockId,
}
```

### 11.4 Payload Block Structure
Payload blocks MUST have:

```rust
struct PayloadBlockHeader {
    block_id: u32,
    uncompressed_size: u64,
    compressed_size: u64,
    codec: u8,
    checksum: u32,  // CRC32 for block integrity
    data: Vec<u8>,
}
```

Block target:
- Minimum: 64 KiB
- Default: 1 MiB
- Maximum: 16 MiB

---

## 12. DEDUPLICATION CONTRACT

### 12.1 Content Addressing Deduplication
If two Cells have identical canonical payload:

```
Model A: Cell C1 → payload_hash H
Model B: Cell C2 → payload_hash H
```

Then:

```
Cell C1 ─┐
         ├─→ Payload H (stored once)
Cell C2 ─┘
```

The payload MUST be stored only once. Both Cells reference the same physical payload.

### 12.2 Deduplication Index
NCL MUST maintain a deduplication index:

```
PayloadHash → [CellId, CellId, CellId, ...]
```

Writer MUST check this index before writing payload.

### 12.3 Deduplication Immutability
Once deduplicated, the relationship MUST be immutable. Removing a Cell referencing deduplicated payload requires garbage collection in a new revision.

---

## 13. GRAPH ARCHITECTURE

### 13.1 Cell Graph Structure
NCI models are represented as Cell Graphs:

```
Cell A
 │
 ├── dependency ──→ Cell B
 │
 ├── routing ─────→ Cell C
 │
 └── composition ─→ Cell D
```

Relationships:
- **Dependency:** Upstream computation dependency
- **Routing:** Potential routing target
- **Composition:** Component composition
- **Correlation:** Statistical correlation
- **Causation:** Causal relationship

### 13.2 Graph Columnar Storage
Graph relationships MUST be stored columnar:

```
SOURCE[]      // [32B × E] Cell IDs
TARGET[]      // [32B × E] Cell IDs
RELATION[]    // [1B × E] relationship type
WEIGHT[]      // [4B × E] floating-point weight
FLAGS[]       // [1B × E] additional flags
```

Where E = number of edges.

### 13.3 Graph Lazy Loading
The runtime MUST NOT require the entire graph be resident in memory. Graph MUST support:

- Selective edge loading
- Neighborhood queries
- Lazy expansion
- Ancestor/descendant traversal with bounded memory

---

## 14. ROUTING ENGINE & DYNAMIC EXECUTION

### 14.1 Routing Authority
The Routing Engine determines which Cells execute given:

```
Input Query
    ├── context (previous activations)
    ├── difficulty estimation
    ├── hardware budget (max_cells, max_bytes)
    ├── latency budget (max_latency_us)
    ├── memory budget (max_bytes)
    └── energy budget (if applicable)

Routing Output:
    ├── selected_cells: [CellId, CellId, ...]
    ├── activation_order: [CellId, CellId, ...]
    ├── confidence_scores: [0.98, 0.94, 0.89, ...]
    └── fallback_policy: (recursive or early_exit)
```

### 14.2 Dynamic Execution Model
Execution MUST NOT assume fixed sequential pipeline:

```
FORBIDDEN:
Input → Layer1 → Layer2 → Layer3 → Output

REQUIRED:
Input
  ↓
Encode
  ↓
Query Routing
  ↓
Select Cells
  ↓
Execute Selected Cells
  ↓
Evaluate Result
  ├─→ sufficient ──→ Output
  │
  └─→ insufficient ──→ Query Routing (recursive)
```

Execution depth MUST vary based on input complexity and routing decisions.

### 14.3 Compute Budget
The runtime MUST enforce hard budgets:

```rust
struct ComputeBudget {
    max_depth: u32,
    max_cells: usize,
    max_bytes: u64,
    max_compute: u64,
    max_latency_us: u64,
}
```

When any budget is exhausted:

```
if budget_remaining == 0 {
    return NCI-E-BUDGET or current_best_result
}
```

Budget exhaustion MUST NOT cause silent truncation. It MUST produce explicit error or best-effort result with confidence metric.

### 14.4 Difficulty Estimation
Router MUST estimate difficulty and allocate Cells accordingly:

```
Difficulty    Typical Cell Count
Easy          5-10
Medium        50-100
Hard          500-1000
Very Hard     Recursive execution or adaptive depth
```

These are guidance targets, NOT fixed allocations. Router retains full discretion.

---

## 15. MEMORY ARCHITECTURE

### 15.1 Memory as Cell
Memory is not a cache. Memory is a Cell:

```
Memory
    ├── Working Memory (current session state)
    ├── Episodic Memory (specific events/examples)
    ├── Semantic Memory (facts, relationships)
    ├── Procedural Memory (learned patterns)
    └── Consolidated Memory (long-term learned state)
```

Each Memory category is represented as Cell(s).

### 15.2 Working Memory Bounds
Working memory MUST be bounded to prevent unbounded context growth:

```
Working Memory Capacity MUST remain:
    MAX(fixed_limit, K × sqrt(seq_len))
```

where K is a tuning constant and seq_len is sequence length.

Working memory MUST NOT be a dump of entire context. It MUST contain:
- Active computation state
- Selected relevant history
- Confidence metrics
- Intermediate results only

### 15.3 Persistent Memory
Memory MUST persist across sessions:

```
Session 1
    ↓
Memory Write
    ↓
Memory Cell (committed to NCL)
    ↓
New Revision
    ↓
Session 2 (Memory Cell accessible)
```

Memory persistence requires Cell identity, allowing across-session refinement.

### 15.4 Memory Consolidation
Runtime MAY consolidate memory in background:

```
Fragmented Memory State
    ↓
Consolidation
    ↓
Optimized Memory Cell
    ↓
New Revision
```

Consolidation MUST NOT lose information and MUST preserve semantics.

---

## 16. CACHE & DATA MOVEMENT OPTIMIZATION

### 16.1 Cache Hierarchy
NCI assumes the following cache hierarchy:

```
NCL Storage
    ↓ (Block I/O)
NVMe/SSD
    ↓ (Page I/O)
RAM
    ↓ (Register I/O)
VRAM/GPU Memory
    ↓ (Kernel I/O)
Execution Engine
```

(or subsets depending on hardware configuration)

### 16.2 Cache Addressing
Cache keys MUST be:

```
(CellId, RepresentationId)
```

NOT file offset, NOT block index, NOT hardware-specific layout.

### 16.3 Prefetch Protocol
Runtime MUST use dependency graph to anticipate:

```
Current Cell
    ↓
Dependency Graph
    ├── likely next Cell (priority HIGH)
    ├── likely next Cell (priority MEDIUM)
    └── unlikely Cell (do not prefetch)

Prefetch high-priority Cells asynchronously.
```

### 16.4 Data Movement Minimization
The PRIMARY optimization metric MUST be:

```
bytes_moved_per_useful_compute
```

NOT merely FLOPs/second.

All architectural decisions MUST optimize for this metric.

---

## 17. EDGE MEMORY MODEL & HETEROGENEOUS HARDWARE

### 17.1 Non-Resident Model Guarantee
A 1 TB model MUST be runnable on a 16 GB RAM device with 1 TB NVMe:

```
Total Model: 1 TB
    ├── NVMe: ~1 TB (full model stored)
    ├── RAM: 16 GB (active working set)
    └── VRAM: 4 GB (execution working set)

Active Computation: ~500 MB (4-10% of model)
```

Full model residency is NOT required. Full model MUST be accessible via I/O.

### 17.2 Performance Caveats
Non-residency does NOT guarantee:
- No latency overhead
- No bandwidth bottleneck
- No I/O wait time

It means the architecture PERMITS non-residency without requiring full loading. Performance depends on:
- I/O bandwidth
- Prefetch accuracy
- Access patterns
- Quantization efficiency

### 17.3 Hardware Abstraction
NCL representation MUST be hardware-independent:

```
NCL Cell
    ├── CPU representation (INT8)
    ├── GPU representation (FP16)
    ├── NPU representation (INT4)
    └── generic representation (FP16)
```

Runtime selects appropriate representation at load time based on available hardware.

---

## 18. CONVERTER PIPELINE

### 18.1 Conversion Streaming Requirement
Conversion from external formats (PyTorch, Safetensors, GGUF, etc.) MUST be streaming:

```
FORBIDDEN:
Load entire 1TB model into RAM → Process → Write NCL

REQUIRED:
Stream from external format
    ↓
Transform / normalize
    ↓
Cellize
    ↓
Columnarize
    ↓
Write to NCL
    ↓
(no intermediate full residency)
```

### 18.2 Conversion Pipeline
Mandatory conversion stages:

```
External Format Reader
    ↓ (tensor stream)
Normalizer
    ↓ (canonical tensor representation)
Semantic Mapper
    ↓ (identify Cell semantics)
Cell Planner
    ↓ (determine Cell boundaries and tiling)
Quantizer
    ↓ (optional; apply quantization)
Compressor
    ↓ (apply compression)
BLAKE3 Hasher
    ↓ (compute Cell identities)
NCL Writer
    ↓ (columnar output)
NCL Model
```

### 18.3 Zero Format Coupling in Runtime
The NCI runtime MUST NOT depend on external formats:

```
FORBIDDEN in nci-runtime:
├── PyTorch import
├── Safetensors parsing
├── GGUF decoding
├── pickle/pickle-equivalent unpacking
└── any format-specific code

ALLOWED only in converters:
├── nci-import-pytorch
├── nci-import-safetensors
├── nci-import-gguf
    └── (all in separate crates, never linked by runtime)
```

### 18.4 Converter Isolation
Converters MUST be isolated:

```
nci-converter (main converter orchestrator)
    ├── nci-import-pytorch (crate) [no runtime dep]
    ├── nci-import-safetensors (crate) [no runtime dep]
    ├── nci-import-gguf (crate) [no runtime dep]
    └── nci-core (only shared crate)

nci-runtime (core runtime)
    ├── nci-core
    ├── nci-ncl
    ├── nci-storage
    └── (NO converter dependencies)
```

---

## 19. REVISION & INCREMENTAL LEARNING

### 19.1 Revision Immutability
Revisions are immutable snapshots. To change a model:

```
Revision R1
    ├── C1 (immutable)
    ├── C2 (immutable)
    ├── C3 (immutable)
    └── C4 (immutable)

Modify C3 → creates C3' with new identity

Revision R2
    ├── C1 ← shared (not copied)
    ├── C2 ← shared (not copied)
    ├── C3' ← new Cell
    └── C4 ← shared (not copied)
```

Shared Cells are NOT duplicated. Only changed Cells create new entries.

### 19.2 Revision DAG
Revisions form a Directed Acyclic Graph (DAG):

```
    R1 ─────┬─────→ R3
    ↓       │       ↑
    └──→ R2 ┴──────→ R4
```

Reader specifies target revision_id. Runtime resolves Cell chain from that revision.

### 19.3 Incremental Learning
Model refinement MUST occur at Cell granularity:

```
Learn Pattern
    ↓
Identify affected Cells
    ↓
Refine/Create new Cell
    ↓
Update Routing
    ↓
New Revision
    ↓
Consolidation (optional)
```

NOT:

```
Full model retraining
    ↓
Full model replacement
```

---

## 20. OBSERVABILITY & METRICS

### 20.1 Mandatory Metrics
NCI implementations MUST collect and expose the following metrics:

| Metric | Type | Unit | Importance |
|---|---|---|---|
| nci_cells_total | Gauge | count | HIGH |
| nci_cells_active | Gauge | count | HIGH |
| nci_cells_loaded | Gauge | count | HIGH |
| nci_bytes_loaded | Counter | bytes | HIGH |
| nci_bytes_moved | Counter | bytes | CRITICAL |
| nci_bytes_moved_per_token | Histogram | bytes/token | CRITICAL |
| nci_cache_hit_rate | Histogram | ratio [0,1] | HIGH |
| nci_gpu_residency | Gauge | ratio [0,1] | MEDIUM |
| nci_npu_residency | Gauge | ratio [0,1] | MEDIUM |
| nci_active_parameter_ratio | Gauge | ratio [0,1] | HIGH |
| nci_execution_depth | Histogram | depth | MEDIUM |
| nci_route_count | Counter | count | MEDIUM |
| nci_memory_bytes | Gauge | bytes | MEDIUM |
| nci_revision_count | Gauge | count | LOW |
| nci_corrupt_cells | Counter | count | CRITICAL |

### 20.2 Primary Optimization Metric
The metric that MUST drive all optimization decisions:

```
bytes_moved_per_useful_compute
```

All architectural and implementation decisions MUST be evaluated against this metric.

---

## 21. READER/WRITER CONTRACT

### 21.1 Reader Lifecycle
NCL Reader MUST implement the following state machine:

```
CLOSED
  ↓ open()
OPENED
  ↓ validate_superblock()
VALIDATED
  ↓ load_index()
READY
  ↓ resolve/read
ACTIVE
  ↓ close()
CLOSED
```

Each transition MUST be explicit. Reader MUST NOT implicitly transition.

### 21.2 Reader API Contract
Reader MUST provide:

```rust
trait NclReader {
    fn open(path: &Path) -> Result<Self>;
    fn manifest(&self) -> Result<&Manifest>;
    fn resolve_cell(&self, id: &CellId) -> Result<CellRef>;
    fn read_cell(&self, id: &CellId) -> Result<Cell>;
    fn read_column(&self, id: &ColumnId) -> Result<ColumnReader>;
    fn read_block(&self, id: &BlockId) -> Result<BlockReader>;
    fn read_column_predicate(
        &self,
        columns: &[ColumnId]
    ) -> Result<SelectiveColumnReader>;
}
```

Async variants MUST also be provided:

```rust
async fn read_cell_async(&self, id: &CellId) -> Result<Cell>;
async fn read_cells_batch_async(
    &self,
    ids: &[CellId]
) -> Result<Vec<Cell>>;
```

### 21.3 Reader Selective Access Guarantee
Reader MUST NOT load entire file for single Cell read:

```
FORBIDDEN:
request: read_cell(C042)
    ↓
load: entire model (1 TB) into RAM

REQUIRED:
request: read_cell(C042)
    ↓
index lookup: find C042 metadata
    ↓
locate: payload block
    ↓
read: only required block
    ↓
verify: BLAKE3-256
    ↓
decompress: if needed
    ↓
return: Cell C042
```

### 21.4 Writer Lifecycle
NCL Writer MUST implement:

```
CLOSED
  ↓ create()
CREATED
  ↓ begin_transaction()
WRITING
  ├── write_cell()
  ├── write_column()
  ├── write_payload()
  └── ...
  ↓
VALIDATING
  ↓
FINALIZING
  ↓
COMMITTING
  ↓
COMMITTED
```

### 21.5 Writer Streaming Guarantee
Writer MUST accept streaming input without full residency:

```
while let Some(tensor) = source.next()? {
    let cells = planner.plan(tensor)?;
    for cell in cells {
        writer.write_cell(cell)?;
    }
}
writer.finalize()?;
```

No intermediate buffer MUST hold entire model.

### 21.6 Writer Transactional Guarantee
Writer MUST provide atomic transactions:

```
BEGIN
  ├── write data
  ├── validate schema
  └── compute hashes
    ↓
VALIDATE
  ├── verify integrity
  └── check deduplication
    ↓
COMMIT or ROLLBACK
```

Partial commits are forbidden. File MUST be either completely valid or unchanged.

### 21.7 Concurrent Access Model
NCL supports:

- **Multiple Readers:** Concurrent reads of same revision (RO lock if applicable)
- **Single Writer:** Exclusive write access, creates new revision
- **Reader-Writer:** Readers access immutable revisions; writer creates new revision in parallel

A reader reading revision R1 is unaffected by writer creating R2.

---

## 22. INTEGRITY & CORRUPTION HANDLING

### 22.1 Integrity Verification
All payload MUST be verified before use:

```
Read Block
    ↓
Decompress
    ↓
Compute BLAKE3-256
    ↓
Compare with stored hash
    ├─ MATCH → release to runtime
    ├─ MISMATCH → NCI-E-CORRUPT (quarantine)
    └─ MISSING → NCI-E-CORRUPT (quarantine)
```

### 22.2 Corruption Isolation
Corruption in one block MUST NOT invalidate entire file:

```
NCL Model
├── Block A: Payload valid ✓
├── Block B: Payload valid ✓
├── Block C: Payload CORRUPT ✗
└── Block D: Payload valid ✓

Cells depending on Block C: UNAVAILABLE
Cells depending on blocks A,B,D: AVAILABLE
```

### 22.3 Mandatory Integrity Checks
Reader MUST verify:

1. Superblock magic
2. Superblock hash
3. Manifest hash
4. Index hash
5. Column checksums
6. Payload block checksums
7. Cell payload hashes
8. Revision integrity

Any mismatch MUST trigger quarantine.

### 22.4 Recovery Protocol
Upon corruption detection:

1. **Log event** with NCI-E-CORRUPT
2. **Isolate affected Cells**
3. **Attempt prior revision** (if available)
4. **Alert operator** (if automated system)
5. **Do NOT attempt repair** (immutable design prevents safe repair)
6. **Preserve evidence** (do not overwrite corrupted blocks)

---

## 23. SECURITY REQUIREMENTS

### 23.1 Importer Safety
Importer from external format MUST:

- ✗ NEVER execute code from checkpoint (no pickle eval, no torch.load without restrictions)
- ✓ Perform bounds checking on all dimensions
- ✓ Limit memory allocation per tensor
- ✓ Prevent path traversal attacks
- ✓ Validate tensor dimensions (no negative, no overflow)
- ✓ Validate integer operations (detect overflow/underflow)
- ✓ Verify all payload against declared sizes
- ✓ Reject malformed NCL with clear error

### 23.2 PyTorch Importer Restrictions
PyTorch import MUST use restricted loading:

```rust
// FORBIDDEN:
torch.load(checkpoint_path)  // arbitrary code execution risk

// REQUIRED:
torch.load(
    checkpoint_path,
    weights_only=True,  // PyTorch 2.0+ feature
    torch_function_enabled=False,
)
```

### 23.3 Codec Validation
Compression codec MUST be validated:

- Only codecs in official codec registry allowed
- Custom/unknown codecs MUST be rejected
- Decompression MUST have size limits (no decompression bomb)

---

## 24. DETERMINISM CONTRACT

### 24.1 Determinism Guarantee
Given:
- Same input
- Same NCL revision
- Same runtime policy
- Same hardware execution constraints

Result MUST be deterministic in deterministic mode.

### 24.2 Non-Deterministic Hardware
Some hardware kernels (GPU, NPU) may have inherent non-determinism:
- Float rounding
- Parallel reduction order
- Atomic operation order

Implementations MAY provide deterministic mode via:
- Reduced precision
- Sequential execution
- Deterministic kernel library

Or MAY explicitly reject determinism guarantee for that hardware.

### 24.3 Specification of Policy
All policies affecting computation MUST be explicit:

```rust
struct ExecutionPolicy {
    deterministic: bool,
    quantization_rounding: RoundingMode,
    memory_access_order: AccessOrder,
    reduction_order: ReductionOrder,
}
```

---

## 25. ERROR MODEL & EXCEPTION HANDLING

### 25.1 NCI Error Codes
All errors MUST map to official NCI error taxonomy:

| Code | Category | Meaning | Recovery |
|---|---|---|---|
| NCI-E-CORRUPT | Storage | Payload corruption detected | Quarantine; try alternate revision |
| NCI-E-MANIFEST | Format | Manifest invalid/hash mismatch | Reject file; use backup |
| NCI-E-SCHEMA | Format | Schema incompatible | Upgrade reader or reject |
| NCI-E-STORE | Storage | Storage operation failed | Retry or fail-over |
| NCI-E-LOCK | Concurrency | Lock timeout or conflict | Retry or timeout |
| NCI-E-REVISION | Version | Revision chain broken | Use alternate revision |
| NCI-E-MERGE | Revision | Merge conflict (parallel evolution) | Manual intervention |
| NCI-E-BUDGET | Runtime | Compute/latency/memory budget exhausted | Return partial result or error |
| NCI-E-NOTFOUND | Storage | Cell/block not found | Check manifest; use alternate revision |
| NCI-E-IMPORT | Conversion | External format conversion failed | Inspect input file; report error |
| NCI-E-UNSAFE | Security | Unsafe operation detected | Reject operation |
| NCI-E-RECOVERY | Recovery | Recovery attempt failed | Manual intervention |
| NCI-E-HARDWARE | Hardware | Hardware capability missing | Use alternate backend |
| NCI-E-REPRESENTATION | Format | Representation not supported | Convert to supported representation |
| NCI-E-ROUTING | Runtime | Routing failure | Fallback strategy or error |
| NCI-E-FORMAT | Format | Generic format error | Reject file |

All error handling MUST be explicit. Silent failure is forbidden.

---

## 26. TOOLING & CLI

### 26.1 Mandatory CLI Tools
All NCI implementations MUST provide:

```
ncl-convert <input> <output>
    Convert external format to NCL

ncl-inspect <ncl-file>
    Display NCL structure and metadata

ncl-check <ncl-file>
    Verify integrity and structure

ncl-dump <ncl-file> [--cell-id ID]
    Dump NCL contents (metadata or payload)

ncl-stats <ncl-file>
    Display statistics (sizes, counts, ratios)

ncl-repack <ncl-file> <output>
    Optimize/compact NCL (deduplication, compression tuning)

ncl-benchmark <ncl-file> [--cells N]
    Benchmark read/seek performance
```

### 26.2 Exit Codes
All tools MUST use standard exit codes:

```
0: success
1: generic error
2: usage error
3: file not found
64: corrupt file
65: format version unsupported
255: internal error
```

---

## 27. CONFORMANCE & CERTIFICATION

### 27.1 NCI Conformance
An implementation is NCI-1.0 conformant if it satisfies ALL of:

✓ Universal Cell representation
✓ Columnar storage throughout
✓ NCL reader (complete API)
✓ NCL writer (streaming, atomic)
✓ BLAKE3-256 identity
✓ Cell payload integrity
✓ Cell resolver (metadata + payload)
✓ Selective Cell loading
✓ Dynamic execution with routing
✓ Persistent memory
✓ Revision system (immutable, DAG)
✓ Converter from ≥1 major format (PyTorch, Safetensors, GGUF)
✓ Corruption detection & quarantine
✓ Atomic commit semantics
✓ Comprehensive test suite (≥95% coverage)

### 27.2 Conformance Testing
All implementations MUST pass:

```
Unit tests:
  ✓ Cell creation/identity
  ✓ Column encoding/decoding (all types)
  ✓ BLAKE3 hashing
  ✓ Reader state machine
  ✓ Writer transaction
  ✓ Deduplication
  ✓ Revision resolution

Integration tests:
  ✓ Full pipeline: External → NCL → Runtime
  ✓ Large model: 1B, 7B, 13B, 70B, 100B+, 1T parameters
  ✓ Multi-format: PyTorch, Safetensors, GGUF
  ✓ Selective loading
  ✓ Corruption handling
  ✓ Concurrent reader/writer
  ✓ Memory constraints (edge device simulation)

Compatibility tests:
  ✓ Cross-implementation interoperability
  ✓ Version compatibility (1.0 → 1.1, etc.)
```

---

## 28. PERFORMANCE CONTRACT

### 28.1 Target Performance
NCI implementations SHOULD achieve:

| Metric | Target |
|---|---|
| Full model residency | NOT REQUIRED |
| Cell lookup | O(1) after index load |
| Index lookup | O(1) amortized |
| Graph traversal | O(k) for k neighbors |
| Conversion | Streaming (no full RAM load) |
| Hashing | Streaming (no full RAM load) |
| Cell loading | Asynchronous, with prefetch |
| Cache | Cell-addressed, hardware-agnostic |
| Deduplication | Content-addressed (automatic) |
| Active parameter ratio | Target <10% for well-designed models |
| Context growth | Sublinear to sequence length |
| Revision overhead | Cell-level delta (not full copy) |

### 28.2 Performance Non-Guarantees
These are NOT guaranteed:

- ✗ Specific latency values
- ✗ Specific throughput numbers
- ✗ Specific memory usage
- ✗ Acceleration on specific hardware without explicit support

Performance depends on:
- Model design
- Input characteristics
- Hardware configuration
- Quantization efficiency
- Compression effectiveness
- Prefetch accuracy

---

## 29. REPOSITORY STRUCTURE & DEPENDENCIES

### 29.1 Monorepo Crate Organization
Core NCI repository MUST use Rust monorepo structure:

```
nci/
├── crates/
│   ├── nci-core              [foundational types]
│   ├── nci-ncl               [NCL binary reader/writer]
│   ├── nci-storage           [storage engine]
│   ├── nci-graph             [Cell Graph operations]
│   ├── nci-routing           [routing engine]
│   ├── nci-memory            [memory architecture]
│   ├── nci-runtime           [execution runtime]
│   ├── nci-cache             [cache management]
│   ├── nci-quant             [quantization]
│   ├── nci-compress          [compression codecs]
│   ├── nci-converter         [conversion orchestration]
│   ├── nci-revision          [revision/versioning]
│   ├── nci-learning          [incremental learning]
│   ├── nci-integrity         [integrity checking]
│   ├── nci-observability     [metrics/logging]
│   └── nci-api               [public API]
│
├── backends/
│   ├── cpu                   [CPU execution]
│   ├── cuda                  [NVIDIA GPU]
│   ├── metal                 [Apple Metal]
│   ├── vulkan                [Vulkan cross-platform]
│   └── npu                   [Specialized NPU]
│
├── converters/
│   ├── nci-import-pytorch
│   ├── nci-import-safetensors
│   ├── nci-import-gguf
│   └── ...
│
├── tools/
│   ├── ncl-convert
│   ├── ncl-inspect
│   ├── ncl-check
│   ├── ncl-dump
│   ├── ncl-stats
│   ├── ncl-repack
│   └── ncl-benchmark
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── models/          [test models: 1B, 7B, 70B, 1T]
│   └── conformance/
│
├── benches/
│   └── [performance benchmarks]
│
├── schemas/
│   └── [NCL schema definitions]
│
├── docs/
│   └── [technical documentation]
│
└── examples/
    └── [reference implementations]
```

### 29.2 Dependency Rules
Dependency graph MUST follow these rules:

```
nci-core
    ↓
nci-ncl
    ↓
nci-storage
    ├─→ nci-graph
    ├─→ nci-routing
    ├─→ nci-memory
    └─→ nci-cache
        ↓
      nci-runtime
        ├─→ [hardware backends]
        └─→ nci-observability

nci-converter (SEPARATE)
    ├─→ nci-core (only)
    ├─→ nci-import-pytorch (isolated)
    ├─→ nci-import-safetensors (isolated)
    └─→ nci-import-gguf (isolated)
        ↓
      (converters MUST NOT depend on nci-runtime)
```

**Critical Invariant:** nci-runtime MUST NOT import any converter crate.

---

## 30. TESTING & VALIDATION REQUIREMENTS

### 30.1 Mandatory Test Coverage
All implementations MUST achieve:

- Unit test coverage: ≥90% of critical paths
- Integration test coverage: All major workflows
- Conformance test pass rate: 100%
- Large-model validation: 1B → 1T parameter range

### 30.2 Test Categories

**Unit Tests (per crate):**
- Serialization/deserialization
- Column encoding/decoding
- Cell identity hashing
- Index operations
- Graph traversal
- Routing decisions
- Memory management

**Integration Tests:**
- Full pipeline: PyTorch → Converter → NCL → Runtime
- Multi-model: Different architectures, sizes, quantizations
- Selective loading: Verify only needed Cells loaded
- Corruption scenarios: Verify quarantine behavior
- Concurrent access: Readers + Writer
- Edge cases: Tiny devices (256MB RAM), Large devices (1TB+)

**Conformance Tests:**
- Schema validation
- Binary format compliance
- Error code mapping
- Recovery procedures
- Cross-implementation compatibility

### 30.3 Large-Model Test Matrix
Test MUST cover:

```
Model Size    Format          Quantization    Conversion Time
1B            PyTorch         FP16            < 1 hour
7B            Safetensors     INT8            < 2 hours
13B           GGUF            Mixed           < 3 hours
70B           PyTorch         INT4            < 4 hours
100B          Safetensors     INT3            < 6 hours
1T            (simulated)     Mixed           < 12 hours
```

Conversion MUST NOT load entire model into RAM.

---

## 31. CRITICAL INVARIANTS (IMMUTABLE)

The following invariants are inviolable and binding forever:

**NCL-INV-001:** .ncl MUST be readable without external format knowledge.

**NCL-INV-002:** Cell identity MUST use BLAKE3-256.

**NCL-INV-003:** Cell identity MUST derive from canonical uncompressed payload.

**NCL-INV-004:** Compression MUST NOT alter Cell identity.

**NCL-INV-005:** Reader MUST support selective Cell access without full-file loading.

**NCL-INV-006:** Reader MUST support selective Column access.

**NCL-INV-007:** Writer MUST support streaming input without full-model residency.

**NCL-INV-008:** Committed Cells MUST be immutable.

**NCL-INV-009:** Corrupt payload MUST NOT reach execution.

**NCL-INV-010:** Index corruption MUST be detectable and trigger quarantine.

**NCL-INV-011:** Commit MUST be atomic (all-or-nothing).

**NCL-INV-012:** Revision MUST be immutable snapshot.

**NCL-INV-013:** Physical location MUST NOT determine Cell identity.

**NCL-INV-014:** External filename MUST NOT determine Cell identity.

**NCL-INV-015:** Runtime MUST NOT depend on external checkpoint formats.

**NCL-INV-016:** Columnar organization MUST be preserved at all layers.

---

## 32. CHANGE CONTROL & AMENDMENT

### 32.1 Change Authority
Any amendment to this specification MUST be approved by the NCI Engineering Steering Committee and formally documented.

### 32.2 Version Management
- **Major version change:** Incompatible binary format
- **Minor version change:** Backward-compatible extension
- **Patch version:** Clarification, bug fix, non-normative

### 32.3 Deprecation Policy
Removal of features requires:

1. Explicit deprecation notice in specification (1 major version)
2. Clear error messages in implementation
3. Documentation of migration path
4. Minimum 6-month notice period

---

## 33. CONCLUSION

This Engineering Contract establishes the final, binding specification for NCI — Neural Cell Intelligence. Every requirement marked MUST, SHALL, or MUST NOT is mandatory for all implementations. Every principle is non-negotiable.

NCI is defined by:

1. **Universal Cell:** Everything is a Cell.
2. **Columnar Organization:** All data is columnar.
3. **Streaming Architecture:** No full-model residency required.
4. **Content Addressing:** BLAKE3-256 identity.
5. **Immutable Revisions:** Changes produce new revisions, not mutations.
6. **Selective Activation:** Runtime chooses active Cells.
7. **Dynamic Execution:** Depth and routing adapt to input.
8. **Persistent Memory:** Memory survives sessions.
9. **Data Movement First:** Minimize bytes moved.
10. **Zero External Coupling:** Runtime knows only NCL.

All implementations, tests, code reviews, and system validations MUST comply with this specification. Deviations require formal amendment.

**This specification is binding and enforceable effective immediately.**

---

**Document Signature:**
- **Specification:** NCI Engineering Contract v1.0.0
- **Date:** August 12, 2026
- **Status:** FINAL BINDING SPECIFICATION
- **Authority:** NCI Engineering Steering Committee
