# NCI Security Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §23

---

## Purpose

This document specifies NCI Security: importer safety, codec validation, path traversal prevention, and the security requirements for all NCI components.

## Scope

Importer safety, codec validation, path traversal prevention, memory safety, integer overflow protection, and security best practices.

---

## 1. Importer Safety

### 1.1 External Format Restrictions

Importer from external format MUST:

- NEVER execute code from checkpoint (no pickle eval, no torch.load without restrictions)
- Perform bounds checking on all dimensions
- Limit memory allocation per tensor
- Prevent path traversal attacks
- Validate tensor dimensions (no negative, no overflow)
- Validate integer operations (detect overflow/underflow)
- Verify all payload against declared sizes
- Reject malformed NCL with clear error

### 1.2 PyTorch Importer Restrictions

```rust
fn load_pytorch_safe(path: &Path) -> Result<PytorchCheckpoint> {
    // 1. Validate path (no traversal)
    validate_path(path)?;
    
    // 2. Use restricted loading
    torch.load(
        path,
        weights_only=True,
        torch_function_enabled=False,
    )
}
```

### 1.3 Pickle Restrictions

For pickle-based formats:

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

## 2. Codec Validation

### 2.1 Codec Registry

Only codecs in official codec registry allowed:
- 0x00: None (mandatory)
- 0x01: Zstandard (mandatory)

### 2.2 Unknown Codecs

Custom/unknown codecs MUST be rejected:

```rust
fn validate_codec(codec: u8) -> Result<()> {
    match codec {
        0x00 | 0x01 => Ok(()),
        _ => Err(NCIError::Unsafe),
    }
}
```

### 2.3 Decompression Limits

Decompression MUST have size limits:

```rust
fn validate_decompression(
    compressed_size: u64,
    uncompressed_size: u64,
) -> Result<()> {
    // Maximum block size
    if uncompressed_size > 16 * 1024 * 1024 {
        return Err(NCIError::Unsafe);
    }
    
    // Maximum compression ratio (decompression bomb detection)
    if compressed_size > 0 && uncompressed_size / compressed_size > 1000 {
        return Err(NCIError::Unsafe);
    }
    
    Ok(())
}
```

---

## 3. Path Traversal Prevention

### 3.1 Path Validation

```rust
fn validate_path(path: &Path) -> Result<()> {
    // 1. Canonicalize path
    let canonical = path.canonicalize()?;
    
    // 2. Check for traversal
    if canonical.to_string_lossy().contains("..") {
        return Err(NCIError::Unsafe);
    }
    
    // 3. Check for absolute path in relative context
    if path.is_absolute() && !is_expected_absolute(path) {
        return Err(NCIError::Unsafe);
    }
    
    Ok(())
}
```

### 3.2 Symlink Prevention

```rust
fn validate_no_symlinks(path: &Path) -> Result<()> {
    let metadata = std::fs::symlink_metadata(path)?;
    if metadata.file_type().is_symlink() {
        return Err(NCIError::Unsafe);
    }
    Ok(())
}
```

---

## 4. Memory Safety

### 4.1 Allocation Limits

```rust
fn validate_allocation(size: u64, limit: u64) -> Result<()> {
    if size > limit {
        return Err(NCIError::Store);
    }
    Ok(())
}
```

### 4.2 Buffer Overflow Prevention

All buffer operations MUST use bounds checking:

```rust
fn read_bytes(data: &[u8], offset: u64, size: u64) -> Result<&[u8]> {
    let end = offset.checked_add(size)
        .ok_or(NCIError::Format)?;
    
    if end > data.len() as u64 {
        return Err(NCIError::Format);
    }
    
    Ok(&data[offset as usize..end as usize])
}
```

---

## 5. Integer Overflow Protection

### 5.1 Safe Arithmetic

```rust
fn safe_add(a: u64, b: u64) -> Result<u64> {
    a.checked_add(b).ok_or(NCIError::Format)
}

fn safe_mul(a: u64, b: u64) -> Result<u64> {
    a.checked_mul(b).ok_or(NCIError::Format)
}

fn safe_shl(a: u64, b: u32) -> Result<u64> {
    a.checked_shl(b).ok_or(NCIError::Format)
}
```

### 5.2 Dimension Validation

```rust
fn validate_dimensions(dims: &[u64]) -> Result<()> {
    let mut total = 1u64;
    for &dim in dims {
        if dim == 0 {
            return Err(NCIError::Format);
        }
        total = total.checked_mul(dim)
            .ok_or(NCIError::Format)?;
    }
    Ok(())
}
```

---

## 6. Integrity Verification

### 6.1 Hash Verification

All hashes MUST be verified before data release:

```rust
fn verify_hash(data: &[u8], expected: &[u8; 32]) -> Result<()> {
    let computed = blake3_256(data);
    if computed != *expected {
        return Err(NCIError::Corrupt);
    }
    Ok(())
}
```

### 6.2 Checksum Verification

All checksums MUST be verified:

```rust
fn verify_checksum(data: &[u8], expected: u32) -> Result<()> {
    let computed = crc32(data);
    if computed != expected {
        return Err(NCIError::Corrupt);
    }
    Ok(())
}
```

---

## 7. Security Audit

### 7.1 Audit Requirements

All NCI implementations SHOULD undergo security audit:
- Importer safety
- Codec validation
- Path traversal prevention
- Memory safety
- Integer overflow protection

### 7.2 Vulnerability Reporting

Vulnerabilities SHOULD be reported to NCI Engineering Steering Committee.

---

## 8. Validation

1. Importer MUST use safe loading.
2. Codecs MUST be validated.
3. Paths MUST be validated.
4. Memory allocations MUST be bounded.
5. Integer operations MUST be checked.

---

## 9. Error Handling

| Condition | Error Code |
|---|---|
| Unsafe class in pickle | `NCI-E-UNSAFE` |
| Unknown codec | `NCI-E-UNSAFE` |
| Path traversal | `NCI-E-UNSAFE` |
| Decompression bomb | `NCI-E-UNSAFE` |
| Integer overflow | `NCI-E-FORMAT` |
| Memory limit exceeded | `NCI-E-STORE` |

---

## 10. Conformance Requirements

1. Importer MUST use safe loading.
2. Codecs MUST be validated.
3. Paths MUST be validated against traversal.
4. Memory allocations MUST be bounded.
5. Integer operations MUST use checked arithmetic.
6. All hashes and checksums MUST be verified.
