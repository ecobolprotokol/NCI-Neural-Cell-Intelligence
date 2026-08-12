# NCI Converter Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §18; `02-NCI-ARCHITECTURE.md`; `04-NCI-CELL-PLANNER-SPEC.md`

---

## Purpose

This document specifies the NCI Converter: the pipeline for transforming external format models (PyTorch, Safetensors, GGUF) into NCL, including streaming requirements, isolation rules, and the complete conversion pipeline.

## Scope

Converter pipeline, streaming requirement, isolation rules, converter stages, error handling, and the converter contract.

---

## 1. Converter Pipeline

### 1.1 Pipeline Stages

```
External Format Reader
    ↓ (tensor stream)
Normalizer
    ↓ (canonical tensor representation)
Semantic Mapper
    ↓ (identify Cell semantics)
Cell Planner
    ↓ (determine Cell boundaries and tiling)
Quantizer (optional)
    ↓ (apply quantization)
Compressor
    ↓ (apply compression)
BLAKE3 Hasher
    ↓ (compute Cell identities)
NCL Writer
    ↓ (columnar output)
NCL Model
```

### 1.2 Stage Responsibilities

| Stage | Input | Output | Responsibility |
|---|---|---|---|
| External Format Reader | File bytes | Tensor stream | Parse format-specific structure |
| Normalizer | Tensor stream | Canonical tensors | Normalize to NCI tensor format |
| Semantic Mapper | Canonical tensors | Annotated tensors | Assign Cell semantics |
| Cell Planner | Annotated tensors | Cell plans | Determine Cell boundaries |
| Quantizer | Cell plans | Quantized plans | Apply quantization (optional) |
| Compressor | Quantized plans | Compressed data | Apply compression |
| BLAKE3 Hasher | Compressed data | Cell identities | Compute BLAKE3-256 hashes |
| NCL Writer | Cell identities + data | NCL file | Write columnar NCL format |

---

## 2. Streaming Requirement

### 2.1 Principle

Conversion MUST be streaming. Full model residency is forbidden:

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

### 2.2 Memory Budget

| Component | Maximum Memory |
|---|---|
| External format reader | 64 MiB buffer |
| Normalizer | 64 MiB buffer |
| Cell Planner | 64 MiB buffer |
| Quantizer | 64 MiB buffer |
| Compressor | 64 MiB buffer |
| NCL Writer | 64 MiB buffer |
| Total | ≤ 512 MiB |

### 2.3 Streaming Implementation

```rust
fn convert_streaming(
    input: impl Read,
    output: &Path,
    config: &ConversionConfig,
) -> Result<()> {
    let mut reader = ExternalFormatReader::new(input)?;
    let mut normalizer = Normalizer::new()?;
    let mut mapper = SemanticMapper::new(config)?;
    let mut planner = CellPlanner::new(config)?;
    let mut quantizer = Quantizer::new(config)?;
    let mut hasher = Hasher::new()?;
    let mut writer = NclWriter::create(output)?;
    
    let tx = writer.begin_transaction()?;
    
    while let Some(tensor) = reader.next_tensor()? {
        let canonical = normalizer.normalize(tensor)?;
        let annotated = mapper.map(canonical)?;
        let cells = planner.plan(annotated)?;
        
        for cell in cells {
            let quantized = quantizer.quantize(cell)?;
            let compressed = compressor.compress(quantized)?;
            let cell_id = hasher.hash(&compressed)?;
            writer.write_cell(tx, &cell_id, &compressed)?;
        }
    }
    
    writer.commit(tx)?;
    writer.finalize()?;
    
    Ok(())
}
```

---

## 3. Isolation Rules

### 3.1 Converter Isolation

Converters MUST be isolated from the runtime:

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

### 3.2 Critical Invariant

`nci-runtime` MUST NOT import any converter crate (NCI-P09, NCL-INV-015).

### 3.3 Dependency Enforcement

The following dependency edges are FORBIDDEN:
- `nci-runtime` → `nci-import-pytorch`
- `nci-runtime` → `nci-import-safetensors`
- `nci-runtime` → `nci-import-gguf`
- `nci-runtime` → `nci-converter`

---

## 4. Converter Interface

### 4.1 Converter Trait

```rust
trait Converter {
    fn detect_format(path: &Path) -> Result<FormatType>;
    fn convert(
        input: &Path,
        output: &Path,
        config: &ConversionConfig,
    ) -> Result<ConversionResult>;
}

struct ConversionConfig {
    target_quantization: Option<QuantizationConfig>,
    target_compression: Option<CompressionConfig>,
    cell_size_target: u64,
    cell_size_max: u64,
    enable_deduplication: bool,
    revision_message: Option<String>,
}

struct ConversionResult {
    model_id: Hash256,
    revision_id: Hash256,
    cell_count: u64,
    tensor_count: u64,
    conversion_time_ms: u64,
    peak_memory_bytes: u64,
}
```

### 4.2 Format Detection

```rust
fn detect_format(path: &Path) -> Result<FormatType> {
    let magic = read_magic(path)?;
    
    match magic {
        [0x50, 0x54, 0x00, 0x00] => Ok(FormatType::PyTorch),  // PT\x00\x00
        [0x89, 0x48, 0x44, 0x46] => Ok(FormatType::Safetensors),  // \x89HDF
        [0x47, 0x47, 0x55, 0x46] => Ok(FormatType::Gguf),  // GGUF
        _ => Err(NCIError::Import),
    }
}
```

---

## 5. Error Handling

| Condition | Error Code | Action |
|---|---|---|
| Unknown format | `NCI-E-IMPORT` | Reject file |
| Format parsing error | `NCI-E-IMPORT` | Reject file |
| Tensor dimension overflow | `NCI-E-IMPORT` | Reject tensor |
| Memory limit exceeded | `NCI-E-STORE` | Abort conversion |
| NCL write error | `NCI-E-STORE` | Abort conversion |
| Cell planning error | `NCI-E-FORMAT` | Abort conversion |
| Conversion timeout | `NCI-E-BUDGET` | Abort conversion |

---

## 6. Validation

1. Converter MUST be isolated from runtime.
2. Conversion MUST be streaming.
3. Memory usage MUST stay within budget.
4. Output NCL file MUST be valid.
5. Conversion result MUST be deterministic for identical input and config.

---

## 7. Conformance Requirements

1. Converter MUST support at least one of: PyTorch, Safetensors, GGUF.
2. Converter MUST be streaming (no full model residency).
3. Converter MUST be isolated from runtime.
4. Converter MUST produce valid NCL files.
5. Converter MUST report conversion statistics.
6. Two independently implemented converters MUST produce interoperable NCL files.
