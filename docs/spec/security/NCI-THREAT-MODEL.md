# NCI Threat Model

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §23; `security/NCI-SECURITY-SPEC.md`

---

## Purpose

This document specifies the NCI Threat Model: identified threats, attack vectors, mitigations, and security requirements for all NCI components.

## Scope

Threat categories, attack vectors, mitigations, security boundaries, and threat response.

---

## 1. Threat Categories

### 1.1 Malicious Input

| Threat | Description | Severity |
|---|---|---|
| Malicious checkpoint | PyTorch/Safetensors/GGUF with exploit | CRITICAL |
| Decompression bomb | Compressed payload expanding to exhaust memory | HIGH |
| Dimension overflow | Tensor dimensions causing integer overflow | HIGH |
| Path traversal | File path escaping sandbox | HIGH |

### 1.2 Data Corruption

| Threat | Description | Severity |
|---|---|---|
| Bit flip | Storage error corrupting data | MEDIUM |
| Partial write | Crash during write leaving inconsistent state | HIGH |
| Hash collision | Two different payloads producing same hash | LOW (BLAKE3) |

### 1.3 Denial of Service

| Threat | Description | Severity |
|---|---|---|
| Memory exhaustion | Loading too many Cells | HIGH |
| CPU exhaustion | Infinite routing recursion | MEDIUM |
| Storage exhaustion | Creating too many revisions | MEDIUM |

### 1.4 Privilege Escalation

| Threat | Description | Severity |
|---|---|---|
| Code execution via pickle | Arbitrary code in checkpoint | CRITICAL |
| Symlink attack | Symlink pointing outside sandbox | HIGH |

---

## 2. Attack Vectors

### 2.1 External Format Import

**Vector:** Malicious PyTorch checkpoint with pickle exploit

**Mitigation:**
- Use `weights_only=True`
- Use restricted unpickler
- Never eval pickle content
- Validate all tensor data

### 2.2 Decompression

**Vector:** Compressed block expanding to exhaust memory

**Mitigation:**
- Enforce maximum block size (16 MiB)
- Detect suspicious compression ratios (>1000:1)
- Limit total decompressed size per session

### 2.3 File Operations

**Vector:** Path traversal accessing files outside sandbox

**Mitigation:**
- Canonicalize all paths
- Check for `..` components
- Reject absolute paths in relative context
- Prevent symlink escapes

### 2.4 Integer Operations

**Vector:** Integer overflow in dimension calculations

**Mitigation:**
- Use checked arithmetic
- Validate all dimensions before use
- Reject zero dimensions
- Limit maximum dimensions

---

## 3. Security Boundaries

### 3.1 Trust Boundary

```
Untrusted Zone:
    ├── External format files (PyTorch, Safetensors, GGUF)
    ├── Network data (if any)
    └── User input

Trusted Zone:
    ├── NCI runtime
    ├── NCL reader
    ├── NCL writer
    ├── Cell execution
    └── Memory management
```

### 3.2 Boundary Rules

1. Data from untrusted zone MUST be validated before entering trusted zone
2. Trusted zone MUST NOT execute code from untrusted zone
3. Trusted zone MUST enforce resource limits
4. Trusted zone MUST detect and quarantine corruption

---

## 4. Mitigations

### 4.1 Input Validation

```rust
fn validate_input(data: &[u8], config: &ValidationConfig) -> Result<()> {
    // 1. Magic validation
    validate_magic(data)?;
    
    // 2. Version validation
    validate_version(data)?;
    
    // 3. Size validation
    validate_sizes(data, config)?;
    
    // 4. Dimension validation
    validate_dimensions(data)?;
    
    // 5. Hash validation
    validate_hashes(data)?;
    
    Ok(())
}
```

### 4.2 Resource Limits

```rust
struct ResourceLimits {
    max_memory: u64,
    max_cells: usize,
    max_depth: u32,
    max_file_size: u64,
    max_block_size: u64,
}
```

### 4.3 Sandboxing

For untrusted code execution (converters):
- Run in separate process
- No filesystem access outside working directory
- No network access
- Resource limits enforced

---

## 5. Threat Response

### 5.1 Detection

All threats MUST be detected via:
- Integrity verification
- Resource monitoring
- Input validation
- Anomaly detection

### 5.2 Response

| Threat | Response |
|---|---|
| Malicious checkpoint | Reject file, log event |
| Decompression bomb | Reject block, log event |
| Path traversal | Reject path, log event |
| Integer overflow | Reject data, log event |
| Corruption | Quarantine, log event |
| Memory exhaustion | Reject operation, log event |

### 5.3 Logging

All security events MUST be logged:
- Timestamp
- Threat type
- Attack vector
- Mitigation applied
- Resource affected

---

## 6. Security Audit Requirements

### 6.1 Audit Scope

All NCI implementations SHOULD undergo:
- Code review for security
- Fuzzing for input validation
- Penetration testing for importers
- Static analysis for memory safety

### 6.2 Compliance

Implementations SHOULD comply with:
- OWASP guidelines for input validation
- CWE/SANS Top 25 for common vulnerabilities

---

## 7. Validation

1. All threats in §1 MUST have mitigations in §4.
2. All attack vectors in §2 MUST be addressed.
3. Security boundaries in §3 MUST be enforced.
4. All security events MUST be logged.

---

## 8. Conformance Requirements

1. Importer MUST implement all mitigations in §4.1.
2. Resource limits MUST be enforced.
3. Security events MUST be logged.
4. Implementations SHOULD undergo security audit.
