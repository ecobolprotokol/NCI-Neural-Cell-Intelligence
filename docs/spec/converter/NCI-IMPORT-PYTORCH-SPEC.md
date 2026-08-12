# NCI Import PyTorch Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §18.3–§18.4, §23.2; `converter/NCI-CONVERTER-SPEC.md`

---

## Purpose

This document specifies the PyTorch Importer: safe loading of PyTorch checkpoints, tensor extraction, semantic mapping, and conversion to NCL.

## Scope

PyTorch format detection, safe loading (weights_only=True), tensor extraction, semantic mapping, streaming conversion, and security restrictions.

---

## 1. Format Detection

### 1.1 PyTorch Checkpoint Magic

| Format | Magic Bytes | Description |
|---|---|---|
| ZIP-based | `PK\x03\x04` | ZIP archive (PyTorch < 2.0) |
| Safe | `PT\x00\x00` | PyTorch 2.0+ safe format |
| Pickle | `80 04 95` | Pickle protocol 4 |

### 1.2 Detection Algorithm

```rust
fn detect_pytorch_format(path: &Path) -> Result<PytorchFormat> {
    let magic = read_magic(path, 4)?;
    
    match magic {
        [0x50, 0x4B, 0x03, 0x04] => Ok(PytorchFormat::Zip),
        [0x50, 0x54, 0x00, 0x00] => Ok(PytorchFormat::Safe),
        [0x80, 0x04, 0x95, _] => Ok(PytorchFormat::Pickle),
        _ => Err(NCIError::Import),
    }
}
```

---

## 2. Safe Loading

### 2.1 Security Requirement

PyTorch import MUST use restricted loading:

```
FORBIDDEN:
torch.load(checkpoint_path)  // arbitrary code execution risk

REQUIRED:
torch.load(
    checkpoint_path,
    weights_only=True,  // PyTorch 2.0+ feature
    torch_function_enabled=False,
)
```

### 2.2 Implementation

```rust
fn load_pytorch_safe(path: &Path) -> Result<PytorchCheckpoint> {
    // 1. Validate file exists and is readable
    validate_path(path)?;
    
    // 2. Read magic bytes to determine format
    let format = detect_pytorch_format(path)?;
    
    match format {
        PytorchFormat::Safe => {
            // Use safe loading
            load_safe_format(path)
        }
        PytorchFormat::Zip => {
            // ZIP format: extract tensors only, no pickle eval
            load_zip_safe(path)
        }
        PytorchFormat::Pickle => {
            // Pickle format: use restricted unpickler
            load_pickle_safe(path)
        }
    }
}
```

### 2.3 Restricted Unpickler

For pickle-based checkpoints:

```rust
struct RestrictedUnpickler {
    allowed_classes: HashSet<String>,
}

impl RestrictedUnpickler {
    fn new() -> Self {
        let mut allowed = HashSet::new();
        // Allow only tensor-related classes
        allowed.insert("torch.Tensor".to_string());
        allowed.insert("torch.nn.parameter.Parameter".to_string());
        allowed.insert("collections.OrderedDict".to_string());
        allowed.insert("builtins.dict".to_string());
        allowed.insert("builtins.tuple".to_string());
        allowed.insert("builtins.list".to_string());
        
        Self { allowed_classes: allowed }
    }
    
    fn unpickle(&self, data: &[u8]) -> Result<Value> {
        // Use custom unpickler that only allows safe classes
        // Reject any class not in allowed_classes
        // Return NCI-E-UNSAFE for disallowed classes
    }
}
```

---

## 3. Tensor Extraction

### 3.1 State Dict Extraction

```rust
fn extract_state_dict(checkpoint: &PytorchCheckpoint) -> Result<HashMap<String, Tensor>> {
    let mut state_dict = HashMap::new();
    
    for (key, value) in checkpoint.items() {
        match value {
            PytorchValue::Tensor(tensor) => {
                state_dict.insert(key.clone(), tensor);
            }
            PytorchValue::OrderedDict(dict) => {
                // Recursively extract
                let nested = extract_state_dict_from_dict(dict)?;
                for (k, v) in nested {
                    state_dict.insert(format!("{}.{}", key, k), v);
                }
            }
            _ => {
                // Skip non-tensor values
                continue;
            }
        }
    }
    
    Ok(state_dict)
}
```

### 3.2 Tensor Normalization

All tensors are normalized to canonical format:

```rust
fn normalize_tensor(tensor: &PytorchTensor) -> Result<CanonicalTensor> {
    // 1. Extract dtype
    let dtype = match tensor.dtype() {
        PytorchDtype::Float32 => DataType::Fp32,
        PytorchDtype::Float16 => DataType::Fp16,
        PytorchDtype::BFloat16 => DataType::Bf16,
        PytorchDtype::Int8 => DataType::Int8,
        PytorchDtype::Int4 => DataType::Int4,
        _ => return Err(NCIError::Import),
    };
    
    // 2. Extract shape
    let shape = tensor.shape();
    
    // 3. Extract data (contiguous)
    let data = tensor.contiguous().data_ptr();
    
    // 4. Validate
    validate_tensor(&data, &shape, &dtype)?;
    
    Ok(CanonicalTensor {
        dtype,
        shape,
        data,
    })
}
```

---

## 4. Semantic Mapping

### 4.1 PyTorch Pattern Matching

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
| `*.weight` (default) | Weight |
| `*.bias` (default) | Weight |

### 4.2 Fallback

For unrecognized patterns:
- Assign CellKind::Weight
- Log warning with tensor name
- Preserve original name in attributes

---

## 5. Streaming Conversion

### 5.1 Implementation

```rust
fn convert_pytorch_streaming(
    input: &Path,
    output: &Path,
    config: &ConversionConfig,
) -> Result<ConversionResult> {
    let checkpoint = open_pytorch(input)?;
    let state_dict = extract_state_dict(checkpoint)?;
    
    let mut normalizer = Normalizer::new()?;
    let mut mapper = SemanticMapper::new(config)?;
    let mut planner = CellPlanner::new(config)?;
    let mut hasher = Hasher::new()?;
    let mut writer = NclWriter::create(output)?;
    let tx = writer.begin_transaction()?;
    
    for (key, tensor) in state_dict {
        let canonical = normalizer.normalize(tensor)?;
        let semantic_hint = mapper.map_key(&key)?;
        let annotated = AnnotatedTensor::new(canonical, semantic_hint);
        let cells = planner.plan(annotated)?;
        
        for cell in cells {
            let cell_id = hasher.hash(&cell)?;
            writer.write_cell(tx, &cell_id, &cell)?;
        }
    }
    
    writer.commit(tx)?;
    writer.finalize()
}
```

---

## 6. Validation

1. Loading MUST use `weights_only=True`.
2. Only tensor-related classes MUST be unpickled.
3. Tensors MUST be validated before conversion.
4. Output NCL MUST be valid.
5. Conversion MUST be streaming.

---

## 7. Error Handling

| Condition | Error Code |
|---|---|
| Unsafe class in pickle | `NCI-E-UNSAFE` |
| Tensor dimension overflow | `NCI-E-IMPORT` |
| Unsupported dtype | `NCI-E-REPRESENTATION` |
| File not found | `NCI-E-STORE` |
| Conversion failure | `NCI-E-IMPORT` |

---

## 8. Conformance Requirements

1. PyTorch importer MUST use safe loading.
2. Importer MUST be streaming.
3. Importer MUST be isolated from runtime.
4. Importer MUST handle ZIP, Safe, and Pickle formats.
5. Importer MUST map tensor names to CellKinds.
