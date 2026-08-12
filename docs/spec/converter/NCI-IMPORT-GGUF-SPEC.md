# NCI Import GGUF Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §18.3–§18.4; `converter/NCI-CONVERTER-SPEC.md`

---

## Purpose

This document specifies the GGUF Importer: parsing of GGUF format, tensor extraction, metadata handling, quantization mapping, and conversion to NCL.

## Scope

GGUF format detection, header parsing, tensor extraction, quantization mapping, metadata handling, and streaming conversion.

---

## 1. Format Detection

### 1.1 GGUF Magic

| Format | Magic Bytes | Description |
|---|---|---|
| GGUF v1 | `47 47 55 46 01 00 00 00` | GGUF version 1 |
| GGUF v2 | `47 47 55 46 02 00 00 00` | GGUF version 2 |
| GGUF v3 | `47 47 55 46 03 00 00 00` | GGUF version 3 |

### 1.2 Detection Algorithm

```rust
fn detect_gguf(path: &Path) -> Result<bool> {
    let magic = read_magic(path, 4)?;
    Ok(magic == [0x47, 0x47, 0x55, 0x46])
}
```

---

## 2. Header Parsing

### 2.1 GGUF Header Structure

```
[4 bytes: magic "GGUF"]
[4 bytes: version (little-endian u32)]
[8 bytes: tensor_count (little-endian u64)]
[8 bytes: metadata_kv_count (little-endian u64)]
[metadata_kv × metadata_kv_count]
[tensor_info × tensor_count]
[alignment (padding to alignment boundary)]
```

### 2.2 Metadata Key-Value

```
[key_length (u64)] [key (UTF-8)] [value_type (u8)] [value (type-specific)]
```

### 2.3 Tensor Info

```
[name_length (u64)] [name (UTF-8)] [n_dimensions (u64)] [dimensions (u64 × n)] [type (u8)] [offset (u64)]
```

---

## 3. Tensor Extraction

### 3.1 Quantization Type Mapping

| GGUF Type | GGUF Name | NCI DataType |
|---|---|---|
| 0 | F32 | FP32 |
| 1 | F16 | FP16 |
| 2 | Q4_0 | INT4 (custom quantization) |
| 3 | Q4_1 | INT4 (custom quantization) |
| 8 | Q8_0 | INT8 |
| 9 | Q8_1 | INT8 |
| 10 | Q2_K | INT4 (custom quantization) |
| 11 | Q3_K_S | INT3 (custom quantization) |
| 12 | Q3_K_M | INT3 (custom quantization) |
| 13 | Q3_K_L | INT3 (custom quantization) |
| 14 | Q4_K_S | INT4 (custom quantization) |
| 15 | Q4_K_M | INT4 (custom quantization) |
| 16 | Q5_K_S | INT4 (custom quantization) |
| 17 | Q5_K_M | INT4 (custom quantization) |
| 18 | Q6_K | INT4 (custom quantization) |

### 3.2 Extraction Process

```rust
fn extract_gguf_tensors(path: &Path, header: &GgufHeader) -> Result<Vec<ExtractedTensor>> {
    let mut tensors = Vec::new();
    
    for tensor_info in &header.tensor_infos {
        let data = read_bytes_at(path, tensor_info.offset, tensor_info.data_size())?;
        
        // GGUF quantized tensors need special handling
        let (dtype, data) = match tensor_info.ggml_type {
            0 => (DataType::Fp32, data),
            1 => (DataType::Fp16, data),
            8 | 9 => (DataType::Int8, data),
            _ => {
                // Custom GGUF quantization: dequantize to FP32
                let fp32_data = gguf_dequantize(&data, tensor_info.ggml_type)?;
                (DataType::Fp32, fp32_data)
            }
        };
        
        tensors.push(ExtractedTensor {
            name: tensor_info.name.clone(),
            dtype,
            shape: tensor_info.dimensions.clone(),
            data,
        });
    }
    
    Ok(tensors)
}
```

### 3.3 GGUF Dequantization

For GGUF-specific quantization formats (Q4_0, Q4_1, Q2_K, etc.):

```rust
fn gguf_dequantize(data: &[u8], ggml_type: u32) -> Result<Vec<u8>> {
    match ggml_type {
        2 => dequantize_q4_0(data),
        3 => dequantize_q4_1(data),
        10 => dequantize_q2_k(data),
        11 => dequantize_q3_ks(data),
        // ... etc
        _ => Err(NCIError::Representation),
    }
}
```

---

## 4. Semantic Mapping

### 4.1 GGUF Key Mapping

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
| `blk.{i}.attn_norm.bias` | Normalization |
| `token_embd.weight` | Embedding |
| `output.weight` | Weight |

### 4.2 Metadata Extraction

GGUF metadata is preserved as NCI Metadata Cell:

```rust
fn extract_gguf_metadata(header: &GgufHeader) -> AttributeSet {
    let mut attrs = AttributeSet::new();
    
    for (key, value) in &header.metadata {
        attrs.insert(key, value);
    }
    
    attrs
}
```

---

## 5. Streaming Conversion

### 5.1 Implementation

```rust
fn convert_gguf_streaming(
    input: &Path,
    output: &Path,
    config: &ConversionConfig,
) -> Result<ConversionResult> {
    let header = parse_gguf_header(input)?;
    let tensors = extract_gguf_tensors(input, &header)?;
    let metadata = extract_gguf_metadata(&header);
    
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
        attributes: metadata,
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

1. GGUF magic MUST be valid.
2. Header MUST be parseable.
3. Tensor offsets MUST be within file bounds.
4. Quantization types MUST be supported or dequantizable.
5. Shapes MUST be valid.
6. Output NCL MUST be valid.

---

## 7. Error Handling

| Condition | Error Code |
|---|---|
| Invalid magic | `NCI-E-FORMAT` |
| Unsupported GGUF version | `NCI-E-SCHEMA` |
| Tensor offset out of bounds | `NCI-E-IMPORT` |
| Unsupported quantization type | `NCI-E-REPRESENTATION` |
| Dequantization failure | `NCI-E-CORRUPT` |
| File not found | `NCI-E-STORE` |

---

## 8. Conformance Requirements

1. GGUF importer MUST support GGUF v1, v2, v3.
2. Importer MUST dequantize GGUF-specific formats to FP32.
3. Importer MUST map tensor names to CellKinds.
4. Importer MUST be streaming.
5. Importer MUST preserve GGUF metadata.
