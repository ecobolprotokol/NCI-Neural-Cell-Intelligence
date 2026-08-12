# NCL Compatibility Matrix

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §32; `conformance/NCI-CONFORMANCE-SPEC.md`

---

## Purpose

This document specifies the NCL Compatibility Matrix: version compatibility, cross-implementation interoperability, and the compatibility guarantees for NCL implementations.

## Scope

Version compatibility rules, cross-implementation interoperability, feature support matrix, and compatibility guarantees.

---

## 1. Version Compatibility

### 1.1 Semantic Versioning

NCL uses semantic versioning: `MAJOR.MINOR.PATCH`

- **MAJOR:** Incompatible binary format changes
- **MINOR:** Backward-compatible extensions
- **PATCH:** Clarifications, bug fixes, non-normative

### 1.2 Compatibility Rules

| Reader Version | Writer Version | Compatibility |
|---|---|---|
| Same major, same minor | Same major, same minor | FULL |
| Same major, higher minor | Same major, lower minor | FORWARD |
| Same major, lower minor | Same major, higher minor | BACKWARD (with feature flags) |
| Different major | Any | INCOMPATIBLE |

### 1.3 Feature Flags

Minor version additions are gated by feature flags in Superblock:

```rust
struct FeatureFlags {
    has_compression: bool,      // bit 0
    has_deduplication: bool,    // bit 1
    has_quantization: bool,     // bit 2
    has_revisions: bool,        // bit 3
    has_routing: bool,          // bit 4
    has_memory: bool,           // bit 5
    has_graph: bool,            // bit 6
    streaming_ok: bool,         // bit 7
}
```

Reader MUST check feature flags before using features.

---

## 2. Cross-Implementation Interoperability

### 2.1 Interoperability Requirements

Two NCL-1.0 conformant implementations MUST:
1. Read NCL files written by the other
2. Produce identical CellIds for identical content
3. Produce identical canonical byte sequences
4. Support all mandatory column encodings
5. Support all mandatory quantization formats
6. Support all mandatory compression codecs

### 2.2 Interoperability Test Protocol

```
1. Impl A writes NCL file with:
   - 100 cells
   - Mixed CellKinds
   - Multiple representations
   - Deduplication
   - Compression
   - Revisions

2. Impl B reads NCL file:
   - Verify all cells accessible
   - Verify all hashes match
   - Verify all payloads identical
   - Verify revision chain intact

3. Impl B writes NCL file with:
   - Modified cells
   - New revision

4. Impl A reads NCL file:
   - Verify all cells accessible
   - Verify all hashes match
   - Verify revision chain intact
```

---

## 3. Feature Support Matrix

### 3.1 Mandatory Features

| Feature | NCI-1.0-Core | NCI-1.0-Full |
|---|---|---|
| Universal Cell | ✓ | ✓ |
| Columnar Storage | ✓ | ✓ |
| BLAKE3-256 Identity | ✓ | ✓ |
| NCL Reader | ✓ | ✓ |
| NCL Writer | ✓ | ✓ |
| Selective Loading | ✓ | ✓ |
| Atomic Commit | ✓ | ✓ |
| Corruption Detection | ✓ | ✓ |
| Deduplication | ✓ | ✓ |
| Revisions | ✓ | ✓ |
| Routing | - | ✓ |
| Persistent Memory | - | ✓ |
| Converter (≥1 format) | ✓ | ✓ |
| Test Suite (≥95%) | ✓ | ✓ |

### 3.2 Column Encoding Support

| Encoding | Required |
|---|---|
| RAW | ✓ |
| RLE | ✓ |
| DELTA | ✓ |
| DELTA_RLE | ✓ |
| BITPACK | ✓ |
| VARINT | ✓ |
| DICT | ✓ |
| SPARSE | ✓ |

### 3.3 Quantization Format Support

| Format | Required |
|---|---|
| FP32 | ✓ |
| BF16 | ✓ |
| FP16 | ✓ |
| FP8 (E4M3, E5M2) | ✓ |
| INT8 | ✓ |
| INT4 | ✓ |
| INT3 | ✓ |

### 3.4 Compression Codec Support

| Codec | Required |
|---|---|
| None | ✓ |
| Zstandard | ✓ |

---

## 4. Hardware Compatibility

### 4.1 CPU Backend

| Architecture | Support Level |
|---|---|
| x86_64 | FULL |
| ARM64 | FULL |
| RISC-V | BASIC |
| WASM | BASIC |

### 4.2 GPU Backend

| Architecture | Support Level |
|---|---|
| NVIDIA CUDA | FULL |
| Apple Metal | FULL |
| Vulkan | FULL |
| OpenCL | BASIC |

### 4.3 NPU Backend

| Architecture | Support Level |
|---|---|
| Qualcomm Hexagon | BASIC |
| Apple Neural Engine | BASIC |
| Google TPU | BASIC |

---

## 5. Format Compatibility

### 5.1 NCL ↔ External Formats

| External Format | NCL Support | Notes |
|---|---|---|
| PyTorch Checkpoint | ✓ | Via converter |
| Safetensors | ✓ | Via converter |
| GGUF | ✓ | Via converter |
| ONNX | - | Not in v1.0 |
| TensorFlow | - | Not in v1.0 |

### 5.2 Backward Compatibility

NCL v1.0 files MUST be readable by:
- NCL v1.0 readers
- NCL v1.x readers (x > 0)
- NCL v2.0 readers (if major version bump)

### 5.3 Forward Compatibility

NCL v1.x readers (x > 0) MUST handle:
- Unknown feature flags gracefully
- Unknown CellKinds opaquely
- Unknown attributes preserved
- Unknown representations → `NCI-E-REPRESENTATION`

---

## 6. Migration

### 6.1 Version Upgrade Path

```
NCL 1.0 → NCL 1.1:
  - Backward compatible
  - New features optional
  - No migration required

NCL 1.x → NCL 2.0:
  - Incompatible format
  - Migration tool required
  - Full model conversion
```

### 6.2 Migration Tool

```rust
fn migrate_v1_to_v2(input: &Path, output: &Path) -> Result<()> {
    // 1. Read v1 file
    let reader = NclReader::open(input)?;
    
    // 2. Create v2 writer
    let mut writer = NclWriter::create_v2(output)?;
    
    // 3. Copy all cells
    for cell_id in reader.all_cells() {
        let cell = reader.read_cell(&cell_id)?;
        writer.write_cell(cell)?;
    }
    
    // 4. Commit
    writer.finalize()?;
    
    Ok(())
}
```

---

## 7. Validation

1. Version compatibility rules MUST be followed.
2. Cross-implementation tests MUST pass.
3. Feature flags MUST be respected.
4. Migration tools MUST be provided for major version upgrades.

---

## 8. Conformance Requirements

1. Implementations MUST declare version compatibility.
2. Implementations MUST respect feature flags.
3. Implementations MUST support forward compatibility for unknown features.
4. Implementations MUST provide migration tools for major version upgrades.
5. Cross-implementation interoperability MUST be tested.
