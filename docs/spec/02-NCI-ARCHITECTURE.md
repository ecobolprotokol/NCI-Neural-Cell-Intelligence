# NCI Architecture Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §3, §29; `01-NCI-TERMINOLOGY.md`

---

## Purpose

This document specifies the complete NCI system architecture: layer definitions, dependency rules, crate organization, data flow, and the constraints governing inter-layer communication.

## Scope

Logical architecture, physical crate layout, dependency graph, and cross-layer contracts.

---

## 1. Architectural Layers

NCI consists of three logical layers unified by columnar storage, flowing into NCL Storage, then Runtime, then Hardware Backends.

### 1.1 Universal Cell Model Layer

**Role:** Defines the Cell abstraction, Cell identity, Cell kinds, and the canonical representation of all intelligence.

**Components:**
- `nci-core` — Foundational types: `CellId`, `CellKind`, `SchemaId`, `RepresentationId`, `PayloadRef`, `AttributeSet`
- Cell identity formation (BLAKE3-256)
- Canonical serialization rules

**Invariants:**
- All intelligence MUST be represented as Cell objects (NCI-P01)
- Cell identity MUST use BLAKE3-256 (NCL-INV-002)
- Cell identity MUST derive from canonical uncompressed payload (NCL-INV-003)

### 1.2 Column Layer

**Role:** Defines columnar storage organization, column types, encodings, and the column read/write interface.

**Components:**
- `nci-ncl` — NCL binary reader/writer
- `nci-storage` — Storage engine with column management

**Invariants:**
- All data MUST be organized in columnar format (NCI-P02)
- Columnar organization MUST be preserved at all layers (NCL-INV-016)

### 1.3 Graph Layer

**Role:** Manages Cell Graph relationships, edge types, graph traversal, and neighborhood queries.

**Components:**
- `nci-graph` — Cell Graph operations
- Edge types: dependency, routing, composition, correlation, causation
- Columnar graph storage

**Invariants:**
- Graph MUST support selective edge loading (Contract §13.3)
- Graph MUST support bounded-memory traversal (Contract §13.3)

### 1.4 Storage Layer

**Role:** NCL binary format, manifest, index, integrity verification, and revision management.

**Components:**
- `nci-ncl` — NCL format implementation
- `nci-storage` — Storage engine
- `nci-revision` — Revision DAG management
- `nci-integrity` — Integrity checking

### 1.5 Routing Layer

**Role:** Dynamic execution routing, difficulty estimation, compute budget enforcement, and Cell selection.

**Components:**
- `nci-routing` — Routing engine

### 1.6 Memory Layer

**Role:** Persistent memory architecture, working memory bounds, consolidation, and cross-session persistence.

**Components:**
- `nci-memory` — Memory architecture

### 1.7 Revision Layer

**Role:** Immutable revision snapshots, DAG management, incremental evolution, and merge conflict resolution.

**Components:**
- `nci-revision` — Revision/versioning

### 1.8 Runtime Layer

**Role:** Execution orchestration, backend dispatch, cache management, and observability.

**Components:**
- `nci-runtime` — Execution runtime
- `nci-cache` — Cache management
- `nci-observability` — Metrics/logging

### 1.9 Hardware Backend Layer

**Role:** Hardware-specific execution kernels for CPU, GPU, NPU.

**Components:**
- `cpu` — CPU execution backend
- `cuda` — NVIDIA GPU backend
- `metal` — Apple Metal backend
- `vulkan` — Vulkan cross-platform backend
- `npu` — Specialized NPU backend

---

## 2. Crate Organization

### 2.1 Monorepo Layout

```
nci/
├── crates/
│   ├── nci-core
│   ├── nci-ncl
│   ├── nci-storage
│   ├── nci-graph
│   ├── nci-routing
│   ├── nci-memory
│   ├── nci-runtime
│   ├── nci-cache
│   ├── nci-quant
│   ├── nci-compress
│   ├── nci-converter
│   ├── nci-revision
│   ├── nci-learning
│   ├── nci-integrity
│   ├── nci-observability
│   └── nci-api
├── backends/
│   ├── cpu
│   ├── cuda
│   ├── metal
│   ├── vulkan
│   └── npu
├── converters/
│   ├── nci-import-pytorch
│   ├── nci-import-safetensors
│   └── nci-import-gguf
├── tools/
│   ├── ncl-convert
│   ├── ncl-inspect
│   ├── ncl-check
│   ├── ncl-dump
│   ├── ncl-stats
│   ├── ncl-repack
│   └── ncl-benchmark
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── models/
│   └── conformance/
├── benches/
├── schemas/
├── docs/
└── examples/
```

### 2.2 Dependency Graph

The following dependency edges are mandatory:

```
nci-core
    ↓
nci-ncl
    ↓
nci-storage
    ├──→ nci-graph
    ├──→ nci-routing
    ├──→ nci-memory
    └──→ nci-cache
         ↓
       nci-runtime
         ├──→ [hardware backends]
         └──→ nci-observability
```

Separate dependency tree (converters):

```
nci-core (shared)
    ↓
nci-converter
    ├──→ nci-import-pytorch (isolated)
    ├──→ nci-import-safetensors (isolated)
    └──→ nci-import-gguf (isolated)
```

**Critical Invariant:** `nci-runtime` MUST NOT import any converter crate (NCI-P09, NCL-INV-015).

### 2.3 Per-Crate Responsibilities

| Crate | Responsibility | Key Types |
|---|---|---|
| `nci-core` | Foundational types, constants, errors | `CellId`, `CellKind`, `SchemaId`, `RepresentationId`, `DataType`, `NCIError` |
| `nci-ncl` | NCL binary format read/write | `NclReader`, `NclWriter`, `Superblock`, `Manifest` |
| `nci-storage` | Storage engine, column management | `ColumnStore`, `BlockStore`, `DedupIndex` |
| `nci-graph` | Cell Graph operations | `CellGraph`, `Edge`, `Traversal` |
| `nci-routing` | Routing engine | `Router`, `RouteDecision`, `DifficultyEstimate` |
| `nci-memory` | Memory architecture | `MemoryManager`, `WorkingMemory`, `Consolidation` |
| `nci-cache` | Cache management | `CellCache`, `PrefetchEngine` |
| `nci-quant` | Quantization/dequantization | `Quantizer`, `Dequantizer`, `QuantizationScheme` |
| `nci-compress` | Compression codecs | `Compressor`, `Decompressor`, `CodecRegistry` |
| `nci-converter` | Conversion orchestration | `ConversionPipeline`, `Cellizer`, `Normalizer` |
| `nci-revision` | Revision DAG management | `RevisionGraph`, `Revision`, `MergeStrategy` |
| `nci-learning` | Incremental learning | `CellRefinement`, `RoutingUpdate` |
| `nci-integrity` | Integrity checking | `IntegrityVerifier`, `CorruptionHandler` |
| `nci-observability` | Metrics/logging | `MetricsCollector`, `TraceLogger` |
| `nci-api` | Public API surface | High-level user-facing API |

---

## 3. Data Flow

### 3.1 Ingestion Flow

```
External Format (PyTorch/Safetensors/GGUF)
    ↓ [nci-import-* converter]
Canonical Normalizer
    ↓
Semantic Mapper
    ↓
Cell Planner
    ↓ [nci-converter]
Quantizer (optional)
    ↓
Compressor
    ↓
BLAKE3 Hasher
    ↓
NCL Writer [nci-ncl]
    ↓
NCL File
```

### 3.2 Execution Flow

```
NCL File
    ↓ [nci-ncl reader]
Manifest + Index
    ↓
Router [nci-routing]
    ↓ (difficulty estimation, budget)
Selected Cells
    ↓ [nci-storage]
Payload Blocks
    ↓
Decompress + Dequantize
    ↓ [backends]
Execution Result
    ↓
Result Validation
    ├─→ sufficient → Output
    └─→ insufficient → Router (recursive)
```

### 3.3 Persistence Flow

```
Execution Result + New Knowledge
    ↓
Memory Consolidation [nci-memory]
    ↓
New/Modified Cells
    ↓ [nci-ncl writer]
New Revision (immutable snapshot)
    ↓
Revision DAG Update [nci-revision]
```

---

## 4. Cross-Layer Contracts

### 4.1 Cell ↔ Column Contract

- Cell descriptors are stored in Cell Column Region
- Each Cell field maps to one or more columns
- Column encoding is independent per column
- Logical Cell is reconstructed by joining columns on row index

### 4.2 Column ↔ Block Contract

- Blocks contain physical data for column segments
- Block size: 64 KiB–16 MiB, target 1 MiB
- Block boundaries MUST align to column boundaries or be resolvable via offset tables

### 4.3 Block ↔ Payload Contract

- Payload blocks MAY be compressed
- Compression codec is specified in PAYLOAD_CODEC column
- Decompressed payload MUST match canonical payload for identity verification

### 4.4 Graph ↔ Cell Contract

- Graph edges reference CellIds
- Graph edges are stored columnar (SOURCE, TARGET, RELATION, WEIGHT, FLAGS)
- Graph edge count E determines column length

### 4.5 Router ↔ Runtime Contract

- Router produces: selected_cells, activation_order, confidence_scores, fallback_policy
- Runtime MUST enforce ComputeBudget limits
- Router MUST handle budget exhaustion explicitly

### 4.6 Memory ↔ Revision Contract

- Memory modifications produce new Cells
- New Cells are committed via atomic transaction
- New revision references new Memory Cells

---

## 5. Naming Conventions

### 5.1 Crate Names

- Prefix: `nci-`
- Suffix: descriptive noun (e.g., `nci-routing`, `nci-cache`)
- Converter crates: `nci-import-{format}`

### 5.2 Type Names

- Rust standard naming: `CamelCase` for types, `snake_case` for functions/fields
- Error types: `NCIError` with variant prefix matching error code category
- Identifiers: Suffix with type purpose (e.g., `CellId`, `SchemaId`)

### 5.3 File Format Constants

- Magic bytes: ASCII prefix + version byte (e.g., `NCL\x01`)
- Region offsets: `u64` little-endian
- Hashes: 32-byte BLAKE3-256

---

## 6. Conformance Requirements

1. All crates in the `nci/` monorepo MUST declare dependencies per the dependency graph in §2.2.
2. `nci-runtime` MUST NOT have direct or transitive dependencies on any converter crate.
3. Converter crates MUST NOT have dependencies on `nci-runtime`, `nci-routing`, `nci-memory`, or any hardware backend.
4. All cross-layer communication MUST occur through the types defined in `nci-core`.
5. New crate additions MUST be approved and MUST follow naming conventions in §5.
6. All public API types MUST be re-exported through `nci-api`.
