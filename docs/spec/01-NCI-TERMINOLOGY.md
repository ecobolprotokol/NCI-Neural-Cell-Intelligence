# NCI Terminology Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §1–§33

---

## Purpose

This document defines the normative terminology used across all NCI specifications. Every term listed here carries a precise technical meaning. Implementations MUST use these definitions without deviation. Where a term appears in a MUST/SHALL/MUST NOT requirement, its meaning in this document governs interpretation.

## Scope

Applies to all NCI and NCL specifications. All other specification documents in this directory set reference this document for term definitions.

---

## 1. Core Terms

### 1.1 NCI

**Full Name:** Neural Cell Intelligence
**Definition:** An edge-AI architecture using Cell as the universal unit of intelligence and columnar storage as the organizing principle across all system layers.
**NOT:** A model format, a runtime library, or a deployment tool. It is an architectural paradigm encompassing format, storage, runtime, and learning.

### 1.2 NCL

**Full Name:** Neural Cell Language / Neural Cell Format
**Definition:** The canonical binary format for persisting NCI intelligence. File extension: `.ncl`.
**NOT:** A wrapper around PyTorch, Safetensors, GGUF, or any external format. It is an independent binary format.

### 1.3 Cell

**Definition:** The primitive unit of NCI intelligence. Everything in NCI — weights, metadata, routing decisions, memory, graph structure, computation state — is represented as a Cell.
**Invariants:** Immutable once committed (NCI-P04). Content-addressed via BLAKE3-256 (NCI-P03). Minimum size 64 KiB, target 4 MiB, maximum 512 MiB (Contract §4.3).

### 1.4 CellId

**Definition:** A 32-byte BLAKE3-256 hash derived from the canonical uncompressed payload of a Cell. Written in human-readable form as `b3:` followed by 64 hexadecimal characters.
**Formation:** `CellId = BLAKE3-256(canonical_cell_header || canonical_cell_metadata || canonical_uncompressed_payload)`

### 1.5 CellKind

**Definition:** The semantic type of a Cell. Determines the Cell's role in the model. One of: `Weight`, `Embedding`, `AttentionQ`, `AttentionK`, `AttentionV`, `AttentionO`, `MlpGate`, `MlpUp`, `MlpDown`, `Expert`, `ExpertRouter`, `Normalization`, `Encoder`, `Decoder`, `Memory`, `SemanticMemory`, `EpisodicMemory`, `ProceduralMemory`, `Routing`, `Composition`, `Graph`, `Index`, `Execution`, `Policy`, `Metadata`, `Provenance`, `Custom`.

### 1.6 SchemaId

**Definition:** A 32-byte identifier referencing the structural schema of a Cell. Defines the layout and interpretation of the Cell's payload. Content-addressed.

### 1.7 RepresentationId

**Definition:** A 4-byte identifier encoding the variant of a Cell's payload encoding (e.g., FP16, INT8, INT4). A single logical Cell MAY have multiple Representations.

### 1.8 PayloadRef

**Definition:** A reference from a Cell descriptor to its physical payload block(s). Contains block ID and offset information. Does not contain payload data inline.

### 1.9 Column

**Definition:** A contiguous, homogeneous storage region containing a single field across multiple Cells. Columns are the fundamental storage primitive in NCI.

### 1.10 Block

**Definition:** A physical storage unit within the Payload Region. Blocks contain compressed or uncompressed tensor data. Target size: 1 MiB. Range: 64 KiB–16 MiB.

### 1.11 Tile

**Definition:** A sub-tensor partition of a larger tensor. Tiling enables selective loading and columnar access patterns. Each tile is an independent payload unit addressable by TileId.

---

## 2. Identity and Hashing Terms

### 2.1 Content Addressing

**Definition:** An identity scheme where an object's identifier is derived from its content via cryptographic hash. NCI uses BLAKE3-256 exclusively for content addressing.

### 2.2 Canonical Serialization

**Definition:** The deterministic byte encoding of a Cell that serves as input to the identity hash. Canonicalization ensures identical logical Cells produce identical byte sequences regardless of implementation, compiler, or platform.

### 2.3 Canonical Payload

**Definition:** The uncompressed, canonicalized byte representation of a Cell's data. Identity is computed on canonical payload, NOT compressed payload.

### 2.4 Compression Transparency

**Definition:** The guarantee that compression and decompression do not alter Cell identity. Identity is computed BEFORE compression and verified AFTER decompression.

---

## 3. Storage Terms

### 3.1 Columnar Storage

**Definition:** Organization of data where each field across many records is stored contiguously, as opposed to row-oriented storage where all fields of a single record are contiguous.

### 3.2 Region

**Definition:** A logical section of an NCL file. Regions include: Superblock, Manifest, Cell Column, Graph Column, Routing Column, Memory Column, Representation, Payload, Index, Revision, Provenance, Integrity.

### 3.3 Superblock

**Definition:** The fixed 4096-byte header at the beginning of every NCL file. Contains magic bytes, version, region offsets, counts, and integrity hashes.

### 3.4 Manifest

**Definition:** The logical root describing the entire NCL model state. Contains model identity, revision, counts, and root hashes for all major regions.

### 3.5 Index

**Definition:** A lookup structure enabling O(1) amortized access to Cells, Payloads, Tiles, Schemas, Representations, and Columns by their identifiers.

### 3.6 Deduplication

**Definition:** The storage optimization where Cells with identical canonical payload share a single physical payload block. The deduplication index maps payload hashes to sets of CellIds.

---

## 4. Runtime Terms

### 4.1 Selective Activation

**Definition:** The runtime principle that only Cells required for the current computation are loaded and activated. Full model residency is explicitly not required.

### 4.2 Routing

**Definition:** The process by which the NCI runtime determines which Cells to activate for a given input, based on context, difficulty estimation, and compute budgets.

### 4.3 Compute Budget

**Definition:** A constraint set on execution including maximum depth, maximum cells, maximum bytes, maximum compute operations, and maximum latency. Budget exhaustion produces explicit errors or best-effort results.

### 4.4 Difficulty Estimation

**Definition:** The router's assessment of input complexity, used to determine how many Cells to activate and at what depth.

### 4.5 Fallback Policy

**Definition:** The strategy employed when initial routing does not produce sufficient results. MAY be recursive (re-route with more Cells) or early-exit (return best-effort result with confidence).

### 4.6 Bytes Moved Per Useful Compute

**Definition:** The primary optimization metric in NCI. Defined as the ratio of bytes transferred to the number of useful floating-point operations performed. All architectural decisions MUST optimize for this metric.

---

## 5. Memory Terms

### 5.1 Working Memory

**Definition:** Transient computation state for the current session. Bounded by `MAX(fixed_limit, K × sqrt(seq_len))`. Contains active computation state, selected relevant history, confidence metrics, and intermediate results.

### 5.2 Episodic Memory

**Definition:** Memory Cells encoding specific events or examples from past interactions.

### 5.3 Semantic Memory

**Definition:** Memory Cells encoding facts, relationships, and general knowledge.

### 5.4 Procedural Memory

**Definition:** Memory Cells encoding learned patterns, routines, and procedures.

### 5.5 Consolidated Memory

**Definition:** Long-term learned state produced by background consolidation of fragmented memory. Consolidation MUST NOT lose information.

---

## 6. Versioning Terms

### 6.1 Revision

**Definition:** An immutable snapshot of the model state at a point in time. Revisions form a DAG.

### 6.2 Revision DAG

**Definition:** A Directed Acyclic Graph of revisions. Each revision MAY have multiple parents (merge) and multiple children (branch). The reader specifies a target revision_id to resolve the model state.

### 6.3 Incremental Evolution

**Definition:** The principle that model changes occur at Cell granularity. Modified Cells produce new Cells with new identities; unchanged Cells are shared between revisions.

---

## 7. Converter Terms

### 7.1 Converter

**Definition:** A standalone component that transforms external format models (PyTorch, Safetensors, GGUF) into NCL. Converters MUST be isolated from the runtime.

### 7.2 Cellizer

**Definition:** The conversion stage that determines Cell boundaries, semantic types, and tiling from external format tensor streams.

### 7.3 Normalizer

**Definition:** The conversion stage that transforms format-specific tensor representations into canonical NCI tensor representation.

### 7.4 Semantic Mapper

**Definition:** The conversion stage that identifies Cell semantics (kind, role, relationships) from external format metadata and structure.

---

## 8. Integrity Terms

### 8.1 Corruption Quarantine

**Definition:** The isolation of Cells or Blocks whose integrity verification fails. Quarantined Cells are marked unavailable but NOT deleted. The system attempts alternate revisions.

### 8.2 Integrity Check

**Definition:** A verification operation including: magic validation, hash comparison, checksum verification, and bounds checking. ALL integrity checks MUST be performed before data is released to the runtime.

### 8.3 Decompression Bomb

**Definition:** A compressed payload that expands to an unexpectedly large size, potentially exhausting memory or compute resources. NCL decompression MUST enforce size limits.

---

## 9. Concurrency Terms

### 9.1 Multiple Readers

**Definition:** Concurrent access by multiple reader instances to the same revision. Readers access immutable data and require no mutual exclusion beyond initialization.

### 9.2 Single Writer

**Definition:** Exclusive write access to create a new revision. Only one writer MAY be active on a given NCL file at any time.

### 9.3 Reader-Writer Isolation

**Definition:** Readers accessing revision R1 are unaffected by a writer creating revision R2. The writer creates a new revision without modifying the existing one.

---

## 10. Quantization Terms

### 10.1 Per-Cell Quantization

**Definition:** Quantization applied independently to each Cell. Global quantization is forbidden. Different Cells MAY use different quantization levels.

### 10.2 Dequantization

**Definition:** Conversion of quantized representation back to FP32 for computation. The runtime MUST support dequantization for all supported quantization formats.

### 10.3 Quantization Scheme

**Definition:** The complete specification of a quantization including: data type, scale reference, zero-point reference, group size, and rounding mode.

---

## 11. Error Taxonomy

All NCI error codes are defined in Contract §25.1. Implementations MUST map all error conditions to these codes:

| Code | Category | Meaning |
|---|---|---|
| `NCI-E-CORRUPT` | Storage | Payload corruption detected |
| `NCI-E-MANIFEST` | Format | Manifest invalid or hash mismatch |
| `NCI-E-SCHEMA` | Format | Schema incompatible |
| `NCI-E-STORE` | Storage | Storage operation failed |
| `NCI-E-LOCK` | Concurrency | Lock timeout or conflict |
| `NCI-E-REVISION` | Version | Revision chain broken |
| `NCI-E-MERGE` | Revision | Merge conflict |
| `NCI-E-BUDGET` | Runtime | Budget exhausted |
| `NCI-E-NOTFOUND` | Storage | Cell/block not found |
| `NCI-E-IMPORT` | Conversion | External format conversion failed |
| `NCI-E-UNSAFE` | Security | Unsafe operation detected |
| `NCI-E-RECOVERY` | Recovery | Recovery attempt failed |
| `NCI-E-HARDWARE` | Hardware | Hardware capability missing |
| `NCI-E-REPRESENTATION` | Format | Representation not supported |
| `NCI-E-ROUTING` | Runtime | Routing failure |
| `NCI-E-FORMAT` | Format | Generic format error |

---

## 12. RFC 2119 Terminology

All requirements in NCI specifications use RFC 2119 language:

| Term | Definition |
|---|---|
| **MUST** | Absolute requirement. Non-negotiable. |
| **MUST NOT** | Absolute prohibition. Non-negotiable. |
| **SHALL** | Same as MUST. |
| **SHALL NOT** | Same as MUST NOT. |
| **SHOULD** | Recommended. Deviation requires documented justification. |
| **SHOULD NOT** | Not recommended. Use requires documented justification. |
| **MAY** | Optional. Pure implementation choice. |

---

## Conformance Requirements

1. Implementations MUST use these term definitions when interpreting NCI specifications.
2. Where a term conflicts between this document and another NCI specification, this document is authoritative for terminology.
3. Custom extensions MUST NOT redefine existing terms.
4. All public API documentation SHOULD reference this terminology document.
