# NCI Cell Planner Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §4.3, §18.2; `03-NCI-CELL-SPEC.md`

---

## Purpose

This document specifies the Cell Planner: the component responsible for determining Cell boundaries, tiling strategies, semantic assignments, and composition from external format tensor streams during conversion.

## Scope

Cell Planner algorithm, input processing, boundary determination, tiling rules, semantic mapping, size optimization, and integration with the conversion pipeline.

---

## 1. Cell Planner Role

### 1.1 Position in Pipeline

```
External Format Reader
    ↓ (tensor stream)
Normalizer
    ↓ (canonical tensors)
Semantic Mapper
    ↓ (semantic annotations)
CELL PLANNER  ← this document
    ↓ (Cell boundaries)
Quantizer
    ↓
Compressor
    ↓
NCL Writer
```

### 1.2 Responsibilities

The Cell Planner MUST:

1. Receive normalized tensor stream with semantic annotations
2. Determine Cell boundaries (which tensors belong to which Cell)
3. Determine tiling for large tensors
4. Assign CellKind to each Cell
5. Determine dependency relationships
6. Optimize Cell sizes for target hardware
7. Emit Cell descriptors to the NCL Writer

The Cell Planner MUST NOT:

1. Load entire model into memory
2. Modify tensor data
3. Perform quantization
4. Perform compression
5. Compute Cell identities (hashing is a separate stage)

---

## 2. Input Interface

### 2.1 Tensor Stream

```rust
struct TensorDescriptor {
    tensor_name: String,
    dtype: DataType,
    rank: u8,
    dimensions: Vec<u64>,
    layout: MemoryLayout,
    data_source: StreamingSource,
    semantic_hint: Option<SemanticHint>,
}
```

### 2.2 Semantic Hint

```rust
struct SemanticHint {
    layer_index: Option<u64>,
    head_index: Option<u64>,
    expert_index: Option<u64>,
    component: AttentionComponent,  // Q/K/V/O/Gate/Up/Down/Expert/Router/...
    original_format: String,        // "pytorch", "safetensors", "gguf"
    tensor_name_original: String,
}
```

### 2.3 Streaming Source

```rust
enum StreamingSource {
    File { path: PathBuf, offset: u64, length: u64 },
    Memory { ptr: *const u8, length: u64 },
    Mmap { region: MmapRegion },
}
```

---

## 3. Boundary Determination Algorithm

### 3.1 Primary Rules

The Cell Planner applies rules in priority order:

**Priority 1: Semantic Boundary**
- Each attention component (Q, K, V, O) SHOULD be a separate Cell
- Each MLP component (Gate, Up, Down) SHOULD be a separate Cell
- Each expert SHOULD be a separate Cell
- Each normalization layer SHOULD be a separate Cell
- Each embedding table SHOULD be a separate Cell

**Priority 2: Size Constraint**
- Target Cell size: 4 MiB (Contract §4.3)
- Minimum Cell size: 64 KiB
- Maximum Cell size: 512 MiB
- If a semantic unit exceeds 512 MiB, it MUST be tiled into sub-Cells

**Priority 3: Cache Optimization**
- Cells targeting L1 cache: ≤ 64 KiB
- Cells targeting L2 cache: ≤ 256 KiB
- Cells targeting L3 cache: ≤ 4 MiB
- Cells targeting RAM: ≤ 512 MiB

**Priority 4: Access Pattern**
- Frequently co-accessed tensors SHOULD share a Cell
- Sequential access patterns SHOULD be adjacent Cells
- Random access patterns SHOULD have independent Cells

### 3.2 Algorithm

```rust
fn plan_cells(
    tensors: impl Iterator<Item = TensorDescriptor>,
    config: PlannerConfig,
) -> Vec<CellPlan> {
    let mut cells = Vec::new();
    let mut current_group = CellGroup::new();

    for tensor in tensors {
        let estimated_size = tensor.estimated_uncompressed_size();

        // Rule 1: Semantic boundary forces new Cell
        if should_split_semantically(&tensor, &current_group) {
            if !current_group.is_empty() {
                cells.push(current_group.finalize());
                current_group = CellGroup::new();
            }
            current_group.add_tensor(tensor);
            continue;
        }

        // Rule 2: Size constraint forces new Cell
        if current_group.estimated_size() + estimated_size > config.target_cell_size {
            if !current_group.is_empty() {
                cells.push(current_group.finalize());
                current_group = CellGroup::new();
            }
        }

        // Rule 3: Maximum size check
        if estimated_size > config.max_cell_size {
            // Must tile
            let tiles = tile_tensor(&tensor, config.target_cell_size);
            for tile in tiles {
                cells.push(CellPlan::from_tile(tile));
            }
        } else {
            current_group.add_tensor(tensor);
        }
    }

    if !current_group.is_empty() {
        cells.push(current_group.finalize());
    }

    cells
}
```

---

## 4. Tiling Rules

### 4.1 Tiling Trigger

A tensor MUST be tiled when:
- Uncompressed size > 512 MiB (maximum Cell size)
- Uncompressed size > target_cell_size AND semantic boundary is soft

### 4.2 Tiling Strategy

**Preferred: Row-major tile**
- Tile along the largest dimension
- Default tile size: 256 elements along the split dimension
- Each tile MUST be independently addressable

**Alternative: Block tile**
- For tensors with multiple large dimensions
- Tile in blocks of `tile_size × tile_size`
- Default block size: 256 × 256

### 4.3 Tile Descriptor

```rust
struct TileDescriptor {
    tile_id: CellId,           // Each tile is a Cell
    parent_tensor: String,     // Original tensor name
    tile_index: u64,           // Position in tile grid
    shape: Vec<u64>,           // Tile dimensions
    offset: Vec<u64>,          // Offset in parent tensor
    dtype: DataType,
    representation: RepresentationId,
    payload_block: BlockId,
}
```

### 4.4 Tile Size Calculation

```
tile_elements = product(tile_shape)
tile_bytes = tile_elements × element_size(dtype)
```

Where `element_size`:

| DataType | Bytes |
|---|---|
| FP32 | 4 |
| BF16 | 2 |
| FP16 | 2 |
| FP8 | 1 |
| INT8 | 1 |
| INT4 | 0.5 |
| INT3 | 0.375 |

For sub-byte types, tile size MUST be rounded up to byte boundary.

---

## 5. Dependency Resolution

### 5.1 Dependency Sources

Dependencies are derived from:

1. **Model architecture** — Layer ordering in transformer
2. **Semantic annotations** — Attention mechanism structure
3. **Data flow** — Output of one Cell feeds input of another
4. **Routing** — Router Cell depends on routing target Cells

### 5.2 Dependency Rules

1. Dependencies MUST be acyclic (DAG)
2. Dependencies MUST be sorted lexicographically by CellId
3. A Cell MAY have zero dependencies (root Cell)
4. Dependency graph MUST be validated before commit
5. Circular dependencies MUST be rejected with `NCI-E-FORMAT`

### 5.3 Common Dependency Patterns

**Transformer Layer:**
```
AttentionQ ← InputNorm
AttentionK ← InputNorm
AttentionV ← InputNorm
AttentionO ← AttentionQ, AttentionK, AttentionV
MLPGate ← AttentionO (or residual)
MLPUp ← MLPGate
MLPDown ← MLPUp
```

**Mixture of Experts:**
```
Expert_0 ← Router
Expert_1 ← Router
Expert_N ← Router
Output ← Expert_0, Expert_1, ..., Expert_N
```

---

## 6. Semantic Mapping

### 6.1 PyTorch Mapping

| PyTorch Pattern | CellKind |
|---|---|
| `model.layers.{i}.self_attn.q_proj.weight` | AttentionQ |
| `model.layers.{i}.self_attn.k_proj.weight` | AttentionK |
| `model.layers.{i}.self_attn.v_proj.weight` | AttentionV |
| `model.layers.{i}.self_attn.o_proj.weight` | AttentionO |
| `model.layers.{i}.mlp.gate_proj.weight` | MlpGate |
| `model.layers.{i}.mlp.up_proj.weight` | MlpUp |
| `model.layers.{i}.mlp.down_proj.weight` | MlpDown |
| `model.layers.{i}.input_layernorm.weight` | Normalization |
| `model.layers.{i}.input_layernorm.bias` | Normalization |
| `model.embed_tokens.weight` | Embedding |
| `lm_head.weight` | Weight |

### 6.2 Safetensors Mapping

Same as PyTorch (Safetensors uses PyTorch tensor naming conventions).

### 6.3 GGUF Mapping

| GGUF Key Pattern | CellKind |
|---|---|
| `blk.{i}.attn_q.weight` | AttentionQ |
| `blk.{i}.attn_k.weight` | AttentionK |
| `blk.{i}.attn_v.weight` | AttentionV |
| `blk.{i}.attn_output.weight` | AttentionO |
| `blk.{i}.ffn_gate.weight` | MlpGate |
| `blk.{i}.ffn_up.weight` | MlpUp |
| `blk.{i}.ffn_down.weight` | MlpDown |
| `blk.{i}.attn_norm.weight` | Normalization |
| `token_embd.weight` | Embedding |
| `output.weight` | Weight |

### 6.3 Fallback Mapping

For unrecognized patterns:
- 1D tensor (bias): CellKind::Weight
- 2D tensor (weight matrix): CellKind::Weight
- 3D+ tensor: CellKind::Weight
- The semantic mapper SHOULD log a warning for unmapped tensors

---

## 7. Size Optimization

### 7.1 Small Cell Merging

Cells below 64 KiB SHOULD be merged if:
- They share the same CellKind
- They are from the same layer
- Merged size does not exceed target_cell_size
- Merging does not break semantic boundaries

### 7.2 Large Cell Splitting

Cells above 512 MiB MUST be split. Splitting strategy:
1. Tile along largest dimension
2. Each tile becomes an independent Cell
3. Tiles maintain parent relationship via attributes
4. Tile Cells share the same CellKind as parent

### 7.3 Cross-Layer Optimization

For models with repeated layer structure:
- Cells from different layers with identical shapes and semantics SHOULD be separate Cells
- Deduplication handles identical payloads automatically
- Routing between layers uses Cell dependencies

---

## 8. Output Interface

### 8.1 Cell Plan

```rust
struct CellPlan {
    cell_index: u64,
    kind: CellKind,
    schema: SchemaId,
    representation: RepresentationId,
    tensors: Vec<TensorAllocation>,
    dependencies: Vec<u64>,       // indices of dependency Cells
    attributes: AttributeSet,
    estimated_size: u64,
}

struct TensorAllocation {
    tensor_name: String,
    dtype: DataType,
    shape: Vec<u64>,
    offset_in_cell: u64,
    tile_info: Option<TileInfo>,
}
```

### 8.2 Streaming Output

```rust
trait CellPlannerSink {
    fn emit_cell(&mut self, plan: CellPlan) -> Result<()>;
    fn emit_metadata(&mut self, metadata: ModelMetadata) -> Result<()>;
    fn finalize(&mut self) -> Result<PlannerStats>;
}
```

---

## 9. Validation

### 9.1 Pre-commit Validation

Cell Planner MUST verify:
1. All Cells are within size bounds
2. All CellKinds are valid
3. Dependency graph is acyclic
4. All dependencies reference earlier Cells (topological order)
5. No Cell is empty (zero tensors)
6. All tensors have valid dtype and shape
7. Tile offsets are within parent tensor bounds

### 9.2 Statistics

Cell Planner MUST emit statistics:

```rust
struct PlannerStats {
    total_cells: u64,
    total_tiled_cells: u64,
    total_dependencies: u64,
    size_distribution: SizeDistribution,
    kind_distribution: KindDistribution,
    avg_cell_size: u64,
    max_cell_size: u64,
    min_cell_size: u64,
}
```

---

## 10. Error Handling

| Condition | Error Code | Action |
|---|---|---|
| Tensor exceeds max Cell size and cannot tile | `NCI-E-FORMAT` | Reject tensor |
| Circular dependency detected | `NCI-E-FORMAT` | Reject model |
| Invalid semantic mapping | `NCI-E-IMPORT` | Use fallback (Weight) |
| All tensors below minimum size and cannot merge | `NCI-E-FORMAT` | Reject model |
| Streaming source error | `NCI-E-STORE` | Abort conversion |

---

## 11. Conformance Requirements

1. Cell Planner MUST NOT load entire model into memory.
2. Cell Planner MUST emit Cell plans that satisfy size constraints.
3. Cell Planner MUST produce acyclic dependency graphs.
4. Cell Planner MUST handle tensors of any shape up to 2^63 elements.
5. Cell Planner MUST support streaming input and output.
6. Cell Planner MUST emit valid semantic mappings for PyTorch, Safetensors, and GGUF formats.
7. Cell Planner output MUST be deterministic given identical input and configuration.
