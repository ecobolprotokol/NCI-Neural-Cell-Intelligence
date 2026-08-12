# NCI Conformance Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §27

---

## Purpose

This document specifies NCI Conformance: conformance requirements, conformance testing, and the conformance certification process.

## Scope

Conformance requirements, conformance testing, test categories, test coverage, and conformance certification.

---

## 1. Conformance Requirements

### 1.1 NCI-1.0 Conformance

An implementation is NCI-1.0 conformant if it satisfies ALL of:

| # | Requirement | Specification Reference |
|---|---|---|
| 1 | Universal Cell representation | `03-NCI-CELL-SPEC.md` |
| 2 | Columnar storage throughout | `ncl/03-NCL-COLUMNAR-SPEC.md` |
| 3 | NCL reader (complete API) | `ncl/11-NCL-READER-PROTOCOL.md` |
| 4 | NCL writer (streaming, atomic) | `ncl/12-NCL-WRITER-PROTOCOL.md` |
| 5 | BLAKE3-256 identity | `03-NCI-CELL-SPEC.md` §5 |
| 6 | Cell payload integrity | `ncl/07-NCL-INTEGRITY-SPEC.md` |
| 7 | Cell resolver (metadata + payload) | `ncl/11-NCL-READER-PROTOCOL.md` |
| 8 | Selective Cell loading | `ncl/11-NCL-READER-PROTOCOL.md` §3 |
| 9 | Dynamic execution with routing | `runtime/NCI-ROUTING-SPEC.md` |
| 10 | Persistent memory | `runtime/NCI-MEMORY-SPEC.md` |
| 11 | Revision system (immutable, DAG) | `ncl/10-NCL-REVISION-SPEC.md` |
| 12 | Converter from ≥1 major format | `converter/NCI-CONVERTER-SPEC.md` |
| 13 | Corruption detection & quarantine | `ncl/07-NCL-INTEGRITY-SPEC.md` §7 |
| 14 | Atomic commit semantics | `ncl/12-NCL-WRITER-PROTOCOL.md` §4 |
| 15 | Comprehensive test suite (≥95% coverage) | This document §3 |

### 1.2 NCL-1.0 Conformance

An NCL implementation is NCL-1.0 conformant if it satisfies ALL of:

| # | Requirement | Specification Reference |
|---|---|---|
| 1 | Superblock format | `ncl/01-NCL-BINARY-SPEC.md` §3 |
| 2 | Manifest format | `ncl/01-NCL-BINARY-SPEC.md` §4 |
| 3 | Column encoding (all 8 types) | `ncl/03-NCL-COLUMNAR-SPEC.md` |
| 4 | Payload block format | `ncl/04-NCL-TENSOR-PAYLOAD-SPEC.md` §3 |
| 5 | Index (O(1) amortized) | `ncl/06-NCL-INDEX-SPEC.md` |
| 6 | Integrity verification | `ncl/07-NCL-INTEGRITY-SPEC.md` |
| 7 | Compression (None + Zstd) | `ncl/08-NCL-COMPRESSION-SPEC.md` |
| 8 | Quantization (all formats) | `ncl/09-NCL-QUANTIZATION-SPEC.md` |
| 9 | Revision DAG | `ncl/10-NCL-REVISION-SPEC.md` |
| 10 | Canonical serialization | `ncl/02-NCL-CANONICAL-SERIALIZATION.md` |

---

## 2. Conformance Testing

### 2.1 Test Categories

**Unit Tests (per crate):**
- Serialization/deserialization
- Column encoding/decoding
- Cell identity hashing
- Index operations
- Graph traversal
- Routing decisions
- Memory management

**Integration Tests:**
- Full pipeline: External → NCL → Runtime
- Multi-model: Different architectures, sizes, quantizations
- Selective loading: Verify only needed Cells loaded
- Corruption scenarios: Verify quarantine behavior
- Concurrent access: Readers + Writer
- Edge cases: Tiny devices (256MB RAM), Large devices (1TB+)

**Conformance Tests:**
- Schema validation
- Binary format compliance
- Error code mapping
- Recovery procedures
- Cross-implementation compatibility

### 2.2 Test Coverage Requirements

| Category | Minimum Coverage |
|---|---|
| Unit tests | ≥90% of critical paths |
| Integration tests | All major workflows |
| Conformance tests | 100% pass rate |
| Large-model validation | 1B → 1T parameter range |

---

## 3. Test Matrix

### 3.1 Unit Test Matrix

```
Component               Test Cases    Coverage Target
────────────────────────────────────────────────────
Cell creation/identity  ≥50           100%
Column encoding/decoding ≥100         100%
BLAKE3 hashing          ≥20           100%
Reader state machine    ≥30           100%
Writer transaction      ≥30           100%
Deduplication           ≥20           100%
Revision resolution     ≥30           100%
Index operations        ≥40           100%
Graph traversal         ≥30           100%
Routing decisions       ≥30           100%
Memory management       ≥30           100%
```

### 3.2 Integration Test Matrix

```
Pipeline                        Test Cases    Coverage Target
────────────────────────────────────────────────────────────
PyTorch → NCL → Runtime         ≥10           100%
Safetensors → NCL → Runtime     ≥10           100%
GGUF → NCL → Runtime            ≥10           100%
Selective loading                ≥20           100%
Corruption handling              ≥20           100%
Concurrent access                ≥10           100%
Edge device simulation           ≥10           100%
```

### 3.3 Large-Model Test Matrix

```
Model Size    Format          Quantization    Conversion Time
──────────────────────────────────────────────────────────────
1B            PyTorch         FP16            < 1 hour
7B            Safetensors     INT8            < 2 hours
13B           GGUF            Mixed           < 3 hours
70B           PyTorch         INT4            < 4 hours
100B          Safetensors     INT3            < 6 hours
1T            (simulated)     Mixed           < 12 hours
```

---

## 4. Cross-Implementation Testing

### 4.1 Interoperability Requirements

Two independently implemented NCI systems MUST:
1. Read NCL files written by the other
2. Produce identical CellIds for identical content
3. Produce identical canonical byte sequences
4. Support the same column encodings
5. Support the same quantization formats

### 4.2 Test Protocol

```
1. Implementation A writes NCL file
2. Implementation B reads NCL file
3. Verify all Cells are accessible
4. Verify all hashes match
5. Verify all payloads are identical
6. Implementation B writes NCL file
7. Implementation A reads NCL file
8. Verify all Cells are accessible
9. Verify all hashes match
10. Verify all payloads are identical
```

---

## 5. Conformance Certification

### 5.1 Certification Process

1. Implement all requirements in §1.1
2. Pass all tests in §2
3. Pass cross-implementation tests in §4
4. Submit conformance report
5. Review by NCI Engineering Steering Committee
6. Certification issued

### 5.2 Certification Levels

| Level | Requirements |
|---|---|
| NCI-1.0-Core | Requirements 1–8 |
| NCI-1.0-Full | All requirements |
| NCL-1.0 | All NCL requirements |

### 5.3 Certification Maintenance

Certification MUST be maintained:
- Re-test on specification changes
- Re-test on implementation changes
- Annual re-certification

---

## 6. Validation

1. All conformance requirements MUST be tested.
2. Test coverage MUST meet minimum thresholds.
3. Cross-implementation tests MUST pass.
4. Certification process MUST be followed.

---

## 7. Conformance Requirements

1. Implementations MUST pass all conformance tests.
2. Implementations MUST achieve minimum test coverage.
3. Implementations MUST pass cross-implementation tests.
4. Implementations MUST be certified before claiming conformance.
