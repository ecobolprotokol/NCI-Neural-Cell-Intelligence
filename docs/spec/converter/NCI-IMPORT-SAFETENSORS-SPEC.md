# NCI Import Safetensors Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §18.3–§18.4; `converter/NCI-CONVERTER-SPEC.md`

---

## Purpose

This document specifies the Safetensors Importer: parsing of Safetensors format, tensor extraction, semantic mapping, and conversion to NCL.

## Scope

Safetensors format detection, header parsing, tensor extraction, metadata handling, semantic mapping, and streaming conversion.

---

## 1. Format Detection

### 1.1 Safetensors Magic

Safetensors files use HDF5-like magic:

| Format | Magic Bytes | Description |
|---|---|---|
| Safetensors | `89 48 44 46` | \x89HDF (HDF5-compatible header) |

### 1.2 Detection Algorithm

```rust
fn detect_safetensors(path: &Path) -> Result<bool> {
    let magic = read_magic(path, 4)?;
    Ok(magic == [0x89, 0x48, 0x44, 0x46])
}
```

---

## 2. Header Parsing

### 2.1 Safetensors Structure

```
[8 bytes: header length (little-endian u64)]
[header length bytes: JSON header]
[remaining bytes: tensor data]
```

### 2.2 JSON Header Format

```json
{
    "tensor_name": {
        "dtype": "F32",
        "shape": [4096, 4096],
        "data_offsets": [0, 67108864]
    },
    "__metadata__": {
        "format": "pt"
    }
}
```

### 2.3 Supported dtypes

| Safetensors dtype | NCI DataType |
|---|---|
| F32 | FP32 |
| F16 | FP16 |
| BF16 | BF16 |
| I32 | INT32 (not in NCI, convert to FP32) |
| I16 | INT16 (not in NCI, convert to FP32) |
| I8 | INT8 |
| U8 | UINT8 (not in NCI, convert to INT8) |
| BOOL | BOOL |

---

## 3. Tensor Extraction

### 3.1 Extraction Process

```rust
fn extract_tensors(path: &Path) -> Result<Vec<ExtractedTensor>> {
    let file = File::open(path)?;
    let header_len = read_u64(&file)?;
    let header_bytes = read_bytes(&file, header_len as usize)?;
    let header: HashMap<String, TensorInfo> = serde_json::from_slice(&header_bytes)?;
    
    let mut tensors = Vec::new();
    
    for (name, info) in &header {
        if name == "__metadata__" {
            continue;
        }
        
        let data_offset = 8 + header_len + info.data_offsets[0];
        let data_size = info.data_offsets[1] - info.data_offsets[0];
        
        let data = read_bytes_at(path, data_offset, data_size as usize)?;
        
        tensors.push(ExtractedTensor {
            name: name.clone(),
            dtype: parse_dtype(&info.dtype)?,
            shape: info.shape.clone(),
            data,
        });
    }
    
    Ok(tensors)
}
```

### 3.2 Metadata Extraction

```rust
fn extract_metadata(path: &Path) -> Result<HashMap<String, String>> {
    let header = parse_header(path)?;
    
    if let Some(metadata) = header.get("__metadata__") {
        Ok(metadata.clone())
    } else {
        Ok(HashMap::new())
    }
}
```

---

## 4. Semantic Mapping

### 4.1 Safetensors Naming

Safetensors uses PyTorch tensor naming conventions:

| Pattern | CellKind |
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

### 4.2 Fallback

For unrecognized patterns: CellKind::Weight.

---

## 5. Streaming Conversion

### 5.1 Implementation

```rust
fn convert_safetensors_streaming(
    input: &Path,
    output: &Path,
    config: &ConversionConfig,
) -> Result<ConversionResult> {
    let tensors = extract_tensors(input)?;
    let metadata = extract_metadata(input)?;
    
    let mut normalizer = Normalizer::new()?;
    let mut mapper = SemanticMapper::new(config)?;
    let mut planner = CellPlanner::new(config)?;
    let mut hasher = Hasher::new()?;
    let mut writer = NclWriter::create(output)?;
    let tx = writer.begin_transaction()?;
    
    for tensor in tensors {
        let canonical = normalizer.normalize(tensor)?;
        let semantic_hint = mapper.map_key(&canonical.name)?;
        let annotated = AnnotatedTensor::new(canonical, semantic_hint);
        let cells = planner.plan(annotated)?;
        
        for cell in cells {
            let cell_id = hasher.hash(&cell)?;
            writer.write_cell(tx, &cell_id, &cell)?;
        }
    }
    
    // Write metadata as Metadata Cell
    let metadata_cell = CellPlan {
        kind: CellKind::Metadata,
        attributes: AttributeSet::from_map(&metadata),
        ..Default::default()
    };
    let metadata_id = hasher.hash(&metadata_cell)?;
    writer.write_cell(tx, &metadata_id, &metadata_cell)?;
    
    writer.commit(tx)?;
    writer.finalize()
}
```

---

## 6. Validation

1. Header MUST be valid JSON.
2. Tensor offsets MUST be within file bounds.
3. Dtypes MUST be supported.
4. Shapes MUST be valid (non-negative, no overflow).
5. Output NCL MUST be valid.

---

## 7. Error Handling

| Condition | Error Code |
|---|---|
| Invalid header | `NCI-E-FORMAT` |
| Tensor offset out of bounds | `NCI-E-IMPORT` |
| Unsupported dtype | `NCI-E-REPRESENTATION` |
| Invalid shape | `NCI-E-FORMAT` |
| File not found | `NCI-E-STORE` |

---

## 8. Conformance Requirements

1. Safetensors importer MUST parse JSON header correctly.
2. Importer MUST extract all tensors.
3. Importer MUST map tensor names to CellKinds.
4. Importer MUST be streaming.
5. Importer MUST preserve metadata.
