# NCL Representation Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §9.3; `03-NCI-CELL-SPEC.md`; `ncl/04-NCL-TENSOR-PAYLOAD-SPEC.md`

---

## Purpose

This document specifies NCL Representation: how a single logical Cell can have multiple physical representations (FP16, INT8, INT4, etc.), how representations are stored, indexed, and selected at runtime.

## Scope

Representation identity, representation storage, representation index, representation selection, multi-representation Cells, and the relationship between Cell identity and Representation identity.

---

## 1. Representation Model

### 1.1 Multi-Representation Cell

A single logical Cell MAY have multiple Representations:

```
Logical Cell C001
    ├── Representation R1 (FP32) — identity: hash_R1
    ├── Representation R2 (FP16) — identity: hash_R2
    ├── Representation R3 (INT8) — identity: hash_R3
    └── Representation R4 (INT4) — identity: hash_R4
```

### 1.2 Identity Invariants

1. **Logical Cell identity** remains constant across all Representations.
2. **Representation identity** is the hash of the canonical representation-specific payload.
3. Logical Cell identity ≠ Representation identity (unless there is exactly one representation).
4. The Cell's primary identity (CellId) is computed from the canonical uncompressed payload of the Cell's PRIMARY representation.

### 1.3 Primary vs Alternate Representations

- Each Cell MUST have exactly one PRIMARY representation (used for identity computation).
- Additional representations are ALTERNATE (used for hardware-specific optimization).
- The PRIMARY representation is the first representation written.

---

## 2. Representation Storage

### 2.1 Representation Region Structure

```
┌──────────────────────────────────────┐
│ RepresentationRegionHeader (68 bytes) │
├──────────────────────────────────────┤
│ RepresentationEntry[0]                │
│ RepresentationEntry[1]                │
│ ...                                  │
│ RepresentationEntry[N-1]              │
├──────────────────────────────────────┤
│ Representation Data[0]                │
│ Representation Data[1]                │
│ ...                                  │
└──────────────────────────────────────┘
```

### 2.2 Representation Region Header

```rust
#[repr(C, packed)]
struct RepresentationRegionHeader {
    region_type: u16,           // offset 0, size 2 — MUST be 0x0006
    entry_count: u32,           // offset 2, size 4
    total_data_size: u64,       // offset 6, size 8
    region_hash: [u8; 32],      // offset 14, size 32
    reserved: [u8; 14],        // offset 46, size 14
}
// Total: 60 bytes
```

### 2.3 Representation Entry

```rust
#[repr(C, packed)]
struct RepresentationEntry {
    cell_id: [u8; 32],          // offset 0, size 32
    representation_id: u32,     // offset 32, size 4
    dtype: u8,                  // offset 36, size 1
    is_primary: u8,             // offset 37, size 1
    data_offset: u64,           // offset 38, size 8
    data_size: u64,             // offset 46, size 8
    checksum: u32,              // offset 54, size 4
    representation_hash: [u8; 32], // offset 58, size 32
}
// Total: 90 bytes per entry
```

---

## 3. Representation Identity

### 3.1 Formation

```
RepresentationId = BLAKE3-256(
    cell_id
    || representation_id
    || canonical_representation_payload
)
```

### 3.2 Canonical Representation Payload

The canonical payload for a representation includes:

1. Tensor descriptor (dtype, shape, layout)
2. Quantization parameters (scale, zero-point references)
3. Raw tensor data (canonical NaN handling)

### 3.3 Invariants

- Identical tensor data + identical dtype → identical Representation identity
- Different dtype → different Representation identity
- Representation identity is independent of physical storage location

---

## 4. Representation Selection

### 4.1 Selection Criteria

Runtime selects representation based on:

1. **Hardware capability** — CPU may prefer INT8, GPU may prefer FP16
2. **Available memory** — Smaller representations for memory-constrained devices
3. **Compute budget** — Higher precision for accuracy-critical paths
4. **User policy** — Explicit representation preference

### 4.2 Selection Algorithm

```rust
fn select_representation(
    representations: &[RepresentationEntry],
    hardware: &HardwareInfo,
    budget: &ComputeBudget,
    policy: &ExecutionPolicy,
) -> Option<RepresentationEntry> {
    // 1. Filter by hardware capability
    let compatible = representations.iter()
        .filter(|r| hardware.supports_dtype(r.dtype))
        .collect::<Vec<_>>();
    
    // 2. Filter by memory budget
    let affordable = compatible.iter()
        .filter(|r| r.data_size <= budget.remaining_memory)
        .collect::<Vec<_>>();
    
    // 3. Score by preference
    let scored = affordable.iter()
        .map(|r| (r, score_representation(r, hardware, policy)))
        .collect::<Vec<_>>();
    
    // 4. Select highest score
    scored.into_iter()
        .max_by_key(|(_, score)| score)
        .map(|(r, _)| r.clone())
}
```

### 4.3 Fallback

If no representation is compatible:
1. Try dequantizing to FP32
2. If still incompatible → `NCI-E-REPRESENTATION`

---

## 5. Dequantization

### 5.1 Dequantization Process

```
quantized_value → (quantized_value - zero_point) × scale → fp32_value
```

### 5.2 Supported Dequantization Paths

| Source | Target | Supported |
|---|---|---|
| INT4 → FP32 | Yes | Mandatory |
| INT3 → FP32 | Yes | Mandatory |
| INT8 → FP32 | Yes | Mandatory |
| FP8 → FP32 | Yes | Mandatory |
| FP16 → FP32 | Yes | Mandatory |
| BF16 → FP32 | Yes | Mandatory |
| INT4 → FP16 | Yes | Optional |
| INT8 → FP16 | Yes | Optional |
| FP8 → FP16 | Yes | Optional |

### 5.3 Dequantization Quality

Dequantization is an approximation. The runtime SHOULD use the original quantized representation for computation when possible, dequantizing only when the compute kernel requires FP32.

---

## 6. Representation Migration

### 6.1 Runtime-Generated Representations

The runtime MAY generate new representations at load time:

```
Original: INT8 representation
    ↓
Runtime: Dequantize to FP16 for GPU execution
    ↓
New Representation: FP16 (temporary, in-memory only)
```

Temporary representations are NOT committed to NCL. They exist only in memory.

### 6.2 Persisted Representations

If a new representation should persist:
1. Compute Representation identity
2. Add to Representation Region
3. Update Cell's representation list
4. Commit via atomic transaction (new revision)

---

## 7. Validation

1. Each Cell MUST have exactly one PRIMARY representation.
2. Representation entries MUST be sorted by (cell_id, representation_id).
3. Representation data MUST be independently verifiable.
4. dtype MUST be consistent between TensorDescriptor and RepresentationEntry.
5. data_size MUST match actual data on disk.

---

## 8. Error Handling

| Condition | Error Code |
|---|---|
| No compatible representation | `NCI-E-REPRESENTATION` |
| Multiple primary representations | `NCI-E-FORMAT` |
| No primary representation | `NCI-E-FORMAT` |
| Representation checksum mismatch | `NCI-E-CORRUPT` |
| Representation hash mismatch | `NCI-E-CORRUPT` |
| dtype mismatch | `NCI-E-FORMAT` |

---

## 9. Conformance Requirements

1. All implementations MUST support FP32, FP16, BF16, INT8, INT4, INT3 representations.
2. Each Cell MUST have exactly one PRIMARY representation.
3. Representation selection MUST be deterministic for identical inputs.
4. Dequantization to FP32 MUST be supported for all quantized types.
5. Temporary representations MUST NOT be committed to NCL without explicit transaction.
