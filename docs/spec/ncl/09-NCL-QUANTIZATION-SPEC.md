# NCL Quantization Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §10.1–§10.2; `ncl/04-NCL-TENSOR-PAYLOAD-SPEC.md`

---

## Purpose

This document specifies NCL quantization: supported quantization formats, per-Cell quantization, scale/zero-point storage, group quantization, and dequantization rules.

## Scope

Quantization formats, quantization parameters, per-Cell and per-group quantization, scale/zero-point storage, dequantization algorithms, and quantization identity.

---

## 1. Supported Quantization Formats

### 1.1 Mandatory Formats

| Format | Bits | Range | Description |
|---|---|---|---|
| FP32 | 32 | ±3.4×10^38 | IEEE 754 binary32 |
| BF16 | 16 | ±3.4×10^38 | Brain float16 |
| FP16 | 16 | ±65504 | IEEE 754 binary16 |
| FP8_E4M3 | 8 | ±448 | FP8 E4M3 |
| FP8_E5M2 | 8 | ±57344 | FP8 E5M2 |
| INT8 | 8 | [-128, 127] | Signed 8-bit integer |
| INT4 | 4 | [-8, 7] | Signed 4-bit integer |
| INT3 | 3 | [-4, 3] | Signed 3-bit integer |

### 1.2 Data Type Encodings

```rust
enum DataType {
    Fp32 = 0x01,
    Bf16 = 0x02,
    Fp16 = 0x03,
    Fp8E4M3 = 0x04,
    Fp8E5M2 = 0x05,
    Int8 = 0x06,
    Int4 = 0x07,
    Int3 = 0x08,
}
```

---

## 2. Quantization Parameters

### 2.1 Per-Tensor Quantization

Single scale and zero-point for entire tensor:

```
fp32_value = (quantized_value - zero_point) × scale
```

Scale: FP32
Zero-point: INT8 (for INT8), INT4 (for INT4), INT3 (for INT3)

### 2.2 Per-Channel Quantization

Scale and zero-point per output channel:

```
fp32_value = (quantized_value - zero_point[channel]) × scale[channel]
```

Channel axis: typically dimension 0 for weight matrices.

### 2.3 Per-Group Quantization

Scale and zero-point per group of elements:

```
group_index = element_index / group_size
fp32_value = (quantized_value - zero_point[group_index]) × scale[group_index]
```

Default group size: 128 elements.

### 2.4 Group Size Constraints

| Constraint | Value |
|---|---|
| Minimum | 1 |
| Maximum | 4096 |
| Default | 128 |
| Must divide tensor dimension | Yes (for per-group) |

---

## 3. Scale and Zero-Point Storage

### 3.1 Storage as Separate Cells

Scale and zero-point are stored as separate Cells, referenced by hash:

```
Quantized Cell C001
    ├── scale Cell (hash_S)
    │   └── FP32 array
    └── zero_point Cell (hash_Z)
        └── INT8/INT4/INT3 array
```

### 3.2 TensorDescriptor References

```rust
struct TensorDescriptor {
    // ...
    scale_ref_hash: [u8; 32],      // BLAKE3-256 of scale Cell
    zero_point_ref_hash: [u8; 32], // BLAKE3-256 of zero_point Cell
}
```

Zero hash (32 bytes of 0x00) indicates no scale/zero-point (FP32 or unquantized).

### 3.3 Scale Cell Format

```rust
struct ScaleCell {
    quantization_scheme: u8,    // PerTensor, PerChannel, PerGroup
    dtype: u8,                  // FP32 or FP16
    count: u64,                 // Number of scale values
    data: Vec<f32>,             // Scale values
}
```

### 3.4 Zero-Point Cell Format

```rust
struct ZeroPointCell {
    quantization_scheme: u8,    // PerTensor, PerChannel, PerGroup
    dtype: u8,                  // INT8, INT4, or INT3
    count: u64,                 // Number of zero-point values
    data: Vec<i8>,              // Zero-point values (packed for INT4/INT3)
}
```

---

## 4. Per-Cell Quantization

### 4.1 Principle

Quantization is applied per-Cell, NOT globally:

```
Attention-critical Cell    → FP16
Normal Dense Cell          → INT8
Large MLP Cell             → INT4
Rare Expert Cell           → INT3
Embedding Cell             → INT4 or FP16
Routing Cell               → FP16 or INT8
```

### 4.2 Mixed Quantization

A model MAY use different quantization levels for different Cells. The runtime MUST support dequantization to FP32 for computation if required.

### 4.3 Quantization Selection

Writer SHOULD select quantization based on:
- Cell importance (critical cells get higher precision)
- Cell size (larger cells benefit more from quantization)
- Target hardware (some hardware has INT4/INT3 support)
- Accuracy requirements (quantization-sensitive layers get higher precision)

---

## 5. Dequantization

### 5.1 Dequantization Formula

```
fp32_value = (quantized_value - zero_point) × scale
```

### 5.2 Type-Specific Dequantization

**INT8 → FP32:**
```
fp32_value = (int8_value as f32 - zero_point as f32) × scale
```

**INT4 → FP32:**
```
fp32_value = (int4_value as f32 - zero_point as f32) × scale
```

**INT3 → FP32:**
```
fp32_value = (int3_value as f32 - zero_point as f32) × scale
```

**FP8 → FP32:**
```
fp32_value = fp8_to_fp32(fp8_value)  // Direct bit conversion
```

**FP16 → FP32:**
```
fp32_value = fp16_to_fp32(fp16_value)  // Direct bit conversion
```

**BF16 → FP32:**
```
fp32_value = bf16_to_fp32(bf16_value)  // Direct bit conversion
```

### 5.3 Group Dequantization

For per-group quantization:

```
fn dequantize_group(
    quantized: &[i8],  // or i4, i3
    scale: &[f32],
    zero_point: &[i8],
    group_size: usize,
) -> Vec<f32> {
    let mut result = Vec::with_capacity(quantized.len());
    
    for (i, &q) in quantized.iter().enumerate() {
        let group_idx = i / group_size;
        let fp32_val = (q as f32 - zero_point[group_idx] as f32) * scale[group_idx];
        result.push(fp32_val);
    }
    
    result
}
```

---

## 6. Quantization Identity

### 6.1 Identity Computation

Quantized Cell identity is computed from the DEQUANTIZED (FP32) canonical payload:

```
canonical_payload = dequantize_to_fp32(quantized_payload)
cell_id = blake3_256(canonical_cell_header || canonical_cell_metadata || canonical_payload)
```

### 6.2 Rationale

This ensures:
- Identical logical content has identical identity regardless of quantization level
- A Cell quantized at INT8 and the same Cell quantized at INT4 have different representations but the SAME logical Cell identity
- This is NOT the case — see §6.3 for clarification

### 6.3 Correction

Actually, per Contract §9.3, a Cell MAY have multiple Representations. Each Representation has its own identity. The Cell's PRIMARY identity is computed from the PRIMARY representation's canonical payload.

For quantized representations:
- FP32 canonical payload → Cell identity
- INT8 canonical payload → same Cell identity (if PRIMARY is FP32)
- If INT8 is PRIMARY → Cell identity computed from INT8 canonical payload

The key invariant: identity is computed from the PRIMARY representation's canonical payload.

---

## 7. INT4 and INT3 Packing

### 7.1 INT4 Packing

Two INT4 values per byte:
- Low nibble (bits 0-3): element at even index
- High nibble (bits 4-7): element at odd index
- If element count is odd, last byte has unused high nibble (zero-padded)

### 7.2 INT3 Packing

Three INT3 values per 9 bits, packed LSB first:
- Bits 0-2: element 0
- Bits 3-5: element 1
- Bits 6-8: element 2
- Padding to byte boundary as needed

---

## 8. Validation

1. Scale and zero-point Cells MUST be valid and verifiable.
2. Group size MUST divide tensor dimension.
3. Scale count MUST match expected count for quantization scheme.
4. Zero-point count MUST match expected count.
5. Quantized values MUST be within type range.

---

## 9. Error Handling

| Condition | Error Code |
|---|---|
| Invalid quantization scheme | `NCI-E-FORMAT` |
| Group size does not divide dimension | `NCI-E-FORMAT` |
| Scale/zero-point hash not found | `NCI-E-NOTFOUND` |
| Scale/zero-point data corrupt | `NCI-E-CORRUPT` |
| Quantized value out of range | `NCI-E-CORRUPT` |
| Unsupported quantization format | `NCI-E-REPRESENTATION` |

---

## 10. Conformance Requirements

1. All implementations MUST support FP32, FP16, BF16, INT8, INT4, INT3.
2. Per-Cell quantization MUST be supported.
3. Dequantization to FP32 MUST be supported for all formats.
4. Scale and zero-point MUST be stored as separate Cells.
5. Quantization identity MUST follow §6.
6. INT4 and INT3 packing MUST follow §7.
