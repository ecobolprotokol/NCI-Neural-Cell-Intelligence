# NCL Compression Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §10.3–§10.4; `ncl/01-NCL-BINARY-SPEC.md`

---

## Purpose

This document specifies NCL compression: supported codecs, compression parameters, streaming decompression, decompression bomb protection, and codec negotiation.

## Scope

Supported codecs, codec registry, compression parameters, streaming decompression, size limits, codec validation, and decompression bomb protection.

---

## 1. Supported Codecs

### 1.1 Mandatory Codecs

| Codec ID | Name | Status | Streaming |
|---|---|---|---|
| 0x00 | None | Mandatory | N/A |
| 0x01 | Zstandard (zstd) | Mandatory | Yes |

### 1.2 Future Codecs

| Codec ID | Name | Status |
|---|---|---|
| 0x02–0xFE | Reserved | Future specification amendment |
| 0xFF | Implementation-defined | Non-standard, MUST NOT be used in portable NCL |

### 1.3 Codec Selection

Writer SHOULD select codec based on:
- Compression ratio achieved
- Decompression speed required
- Data characteristics (tensor data compresses differently from metadata)

---

## 2. Zstandard Parameters

### 2.1 Default Parameters

| Parameter | Default | Range | Description |
|---|---|---|---|
| compression_level | 3 | 1–22 | Compression level |
| window_log | 0 (auto) | 0–31 | Maximum back-reference distance |
| hash_log | 0 (auto) | 0–27 | Hash table size |
| chain_log | 0 (auto) | 0–27 | Chain table size |
| min_match | 0 (auto) | 3–128 | Minimum match length |
| target_length | 0 (auto) | 1–128 | Target match length |
| compression_strategy | 0 (auto) | 0–4 | Compression strategy |

### 2.2 Content-Defined Chunking

Zstandard in NCL MUST use content-defined chunking:

```
block_size = 1 MiB (default)
chunk_size = 64 KiB (default)
```

Content-defined chunking ensures:
- Resumable decompression
- Parallel decompression
- Corruption isolation within chunks

### 2.3 Frame Format

Each compressed block uses a standard Zstandard frame:

```
[frame_header][compressed_data][checksum]
```

Frame header contains:
- FrameContentSize (if known)
- Window_Descriptor
- Dictionary_ID (0 for no dictionary)

---

## 3. Streaming Decompression

### 3.1 Requirement

Reader MUST support streaming decompression without loading entire compressed or decompressed data into memory simultaneously.

### 3.2 Streaming Process

```
input_stream: compressed bytes
output_buffer: fixed-size buffer (e.g., 64 KiB)

loop:
    chunk = input_stream.read(chunk_size)
    if chunk.is_empty():
        break
    
    decompressed = zstd.decompress(chunk)
    output_buffer.write(decompressed)
    
    if output_buffer.is_full():
        process(output_buffer)
        output_buffer.clear()
```

### 3.3 Memory Bounds

Maximum memory during decompression:
- Input buffer: chunk_size (64 KiB default)
- Output buffer: chunk_size (64 KiB default)
- Zstd internal state: ~256 KiB
- Total: < 1 MiB per decompression stream

---

## 4. Decompression Bomb Protection

### 4.1 Size Limits

| Limit | Value | Enforcement |
|---|---|---|
| Maximum uncompressed block size | 16 MiB | Writer MUST enforce |
| Maximum compression ratio | 1000:1 | Reader SHOULD flag |
| Maximum total decompressed size per file | No limit | N/A |

### 4.2 Detection

```
fn check_decompression_bomb(
    compressed_size: u64,
    uncompressed_size: u64,
) -> Result<()> {
    if uncompressed_size > 16 * 1024 * 1024 {
        return Err(NCIError::Unsafe);  // Block too large
    }
    
    if compressed_size > 0 && uncompressed_size / compressed_size > 1000 {
        return Err(NCIError::Unsafe);  // Suspicious ratio
    }
    
    Ok(())
}
```

### 4.3 Response

On decompression bomb detection:
1. Return `NCI-E-UNSAFE`
2. Do NOT decompress
3. Log event with forensic details
4. Quarantine the block

---

## 5. Codec Validation

### 5.1 Writer Validation

Writer MUST validate:
1. Codec is in registry (0x00 or 0x01 for v1.x)
2. Compression parameters are within valid range
3. Compressed size < uncompressed_size (except for incompressible data)
4. Block checksum is correct

### 5.2 Reader Validation

Reader MUST validate:
1. Codec byte is recognized
2. Compressed data is parseable
3. Decompressed size matches uncompressed_size
4. Decompressed data is valid
5. Decompression does not exceed memory limits

---

## 6. Incompressible Data

### 6.1 Detection

Data is considered incompressible if:
- Compressed size ≥ uncompressed_size (after first attempt)
- Data is random or encrypted

### 6.2 Handling

For incompressible data:
- Set codec = 0x00 (None)
- Set compressed_size = uncompressed_size
- Store data uncompressed

---

## 7. Codec Registry

### 7.1 Registry Format

The codec registry is a simple table:

```
codec_id: u8
name: String (UTF-8)
version: String (UTF-8)
```

### 7.2 Registry Updates

New codecs MAY be added via specification amendment. The registry is:
- Part of the NCL specification
- Versioned alongside NCL
- Readable by all conformant readers

---

## 8. Validation

1. Codec MUST be in registry.
2. Compressed data MUST be parseable.
3. Decompressed size MUST match declared size.
4. Decompression bomb protection MUST be enforced.
5. Streaming decompression MUST NOT exceed memory bounds.

---

## 9. Error Handling

| Condition | Error Code |
|---|---|
| Unknown codec | `NCI-E-FORMAT` |
| Compressed data corrupt | `NCI-E-CORRUPT` |
| Decompressed size mismatch | `NCI-E-CORRUPT` |
| Decompression bomb detected | `NCI-E-UNSAFE` |
| Decompression OOM | `NCI-E-STORE` |
| Checksum mismatch | `NCI-E-CORRUPT` |

---

## 10. Conformance Requirements

1. All implementations MUST support codec 0x00 (None) and 0x01 (Zstandard).
2. Reader MUST support streaming decompression.
3. Decompression bomb protection MUST be implemented.
4. Codec validation MUST be performed before decompression.
5. Incompressible data MUST be stored uncompressed.
6. Zstandard parameters MUST be within specification-defined ranges.
