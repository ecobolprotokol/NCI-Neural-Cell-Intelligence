# NCI Backend ABI Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §17.3; `runtime/NCI-RUNTIME-SPEC.md`

---

## Purpose

This document specifies the NCI Backend ABI: the hardware abstraction interface for CPU, GPU, and NPU backends, including the interface contract and backend selection rules.

## Scope

Backend trait, backend capabilities, backend selection, hardware representation, and backend lifecycle.

---

## 1. Backend Trait

### 1.1 Core Interface

```rust
trait Backend: Send + Sync {
    fn name(&self) -> &str;
    fn capabilities(&self) -> BackendCapabilities;
    fn execute(&self, cell: &Cell, input: &Tensor) -> Result<Tensor>;
    fn supports_dtype(&self, dtype: DataType) -> bool;
    fn memory_available(&self) -> u64;
    fn compute_capability(&self) -> u64;
    fn bandwidth(&self) -> u64;
}
```

### 1.2 Backend Capabilities

```rust
struct BackendCapabilities {
    supported_dtypes: Vec<DataType>,
    max_tensor_size: u64,
    max_batch_size: usize,
    vector_width: usize,
    memory_type: MemoryType,
    compute_type: ComputeType,
}

enum MemoryType {
    SystemRam,
    GpuVram,
    NpuMemory,
}

enum ComputeType {
    Scalar,
    Simd,
    Parallel,
    TensorCore,
}
```

---

## 2. Backend Implementations

### 2.1 CPU Backend

```rust
struct CpuBackend {
    core_count: usize,
    vector_extension: VectorExtension,
    cache_sizes: CacheSizes,
}

impl Backend for CpuBackend {
    fn name(&self) -> &str { "cpu" }
    
    fn capabilities(&self) -> BackendCapabilities {
        BackendCapabilities {
            supported_dtypes: vec![
                DataType::Fp32,
                DataType::Fp16,
                DataType::Bf16,
                DataType::Int8,
                DataType::Int4,
                DataType::Int3,
            ],
            max_tensor_size: u64::MAX,
            max_batch_size: 1024,
            vector_width: self.vector_extension.width(),
            memory_type: MemoryType::SystemRam,
            compute_type: ComputeType::Simd,
        }
    }
    
    fn execute(&self, cell: &Cell, input: &Tensor) -> Result<Tensor> {
        // Execute using CPU kernels
        cpu_kernels::execute(cell, input)
    }
}
```

### 2.2 GPU Backend

```rust
struct GpuBackend {
    device_id: u32,
    vram_size: u64,
    compute_capability: (u32, u32),
    tensor_cores: bool,
}

impl Backend for GpuBackend {
    fn name(&self) -> &str { "cuda" }
    
    fn capabilities(&self) -> BackendCapabilities {
        BackendCapabilities {
            supported_dtypes: vec![
                DataType::Fp32,
                DataType::Fp16,
                DataType::Bf16,
                DataType::Fp8E4M3,
                DataType::Fp8E5M2,
                DataType::Int8,
                DataType::Int4,
            ],
            max_tensor_size: self.vram_size,
            max_batch_size: 4096,
            vector_width: 32,  // warp size
            memory_type: MemoryType::GpuVram,
            compute_type: if self.tensor_cores {
                ComputeType::TensorCore
            } else {
                ComputeType::Parallel
            },
        }
    }
    
    fn execute(&self, cell: &Cell, input: &Tensor) -> Result<Tensor> {
        // Transfer data to GPU
        let gpu_input = self.transfer_to_gpu(input)?;
        
        // Execute using CUDA kernels
        let gpu_output = cuda_kernels::execute(cell, &gpu_input)?;
        
        // Transfer result back
        let output = self.transfer_from_gpu(&gpu_output)?;
        
        Ok(output)
    }
}
```

### 2.3 NPU Backend

```rust
struct NpuBackend {
    device_id: u32,
    npu_memory: u64,
    supported_ops: Vec<OpType>,
}

impl Backend for NpuBackend {
    fn name(&self) -> &str { "npu" }
    
    fn capabilities(&self) -> BackendCapabilities {
        BackendCapabilities {
            supported_dtypes: vec![
                DataType::Int8,
                DataType::Int4,
                DataType::Int3,
            ],
            max_tensor_size: self.npu_memory,
            max_batch_size: 256,
            vector_width: 64,
            memory_type: MemoryType::NpuMemory,
            compute_type: ComputeType::TensorCore,
        }
    }
    
    fn execute(&self, cell: &Cell, input: &Tensor) -> Result<Tensor> {
        // Transfer data to NPU
        let npu_input = self.transfer_to_npu(input)?;
        
        // Execute using NPU kernels
        let npu_output = npu_kernels::execute(cell, &npu_input)?;
        
        // Transfer result back
        let output = self.transfer_from_npu(&npu_output)?;
        
        Ok(output)
    }
}
```

---

## 3. Backend Selection

### 3.1 Selection Algorithm

```rust
fn select_backend(
    cell: &Cell,
    backends: &[Box<dyn Backend>],
    policy: &ExecutionPolicy,
) -> Result<Box<dyn Backend>> {
    // 1. Check policy preference
    if let Some(preferred) = policy.preferred_backend {
        if let Some(backend) = backends.iter().find(|b| b.name() == preferred) {
            if backend.supports_dtype(cell.representation.dtype) {
                return Ok(backend.clone());
            }
        }
    }
    
    // 2. Check hardware capability
    for backend in backends {
        if backend.supports_dtype(cell.representation.dtype) {
            if backend.memory_available() >= cell.size() {
                return Ok(backend.clone());
            }
        }
    }
    
    // 3. Fallback to CPU
    backends.iter()
        .find(|b| b.name() == "cpu")
        .cloned()
        .ok_or(NCIError::Hardware)
}
```

---

## 4. Data Transfer

### 4.1 CPU → GPU

```rust
fn transfer_to_gpu(&self, tensor: &Tensor) -> Result<GpuTensor> {
    // 1. Allocate GPU memory
    let gpu_ptr = cuda_malloc(tensor.size())?;
    
    // 2. Copy data
    cuda_memcpy_host_to_device(gpu_ptr, tensor.ptr, tensor.size())?;
    
    Ok(GpuTensor::new(gpu_ptr, tensor.shape))
}
```

### 4.2 GPU → CPU

```rust
fn transfer_from_gpu(&self, gpu_tensor: &GpuTensor) -> Result<Tensor> {
    // 1. Allocate CPU memory
    let cpu_ptr = malloc(gpu_tensor.size())?;
    
    // 2. Copy data
    cuda_memcpy_device_to_host(cpu_ptr, gpu_tensor.ptr, gpu_tensor.size())?;
    
    Ok(Tensor::new(cpu_ptr, gpu_tensor.shape))
}
```

---

## 5. Validation

1. Backend MUST implement Backend trait.
2. Backend MUST report capabilities correctly.
3. Backend MUST support declared dtypes.
4. Backend MUST handle data transfer.
5. Backend MUST be thread-safe.

---

## 6. Error Handling

| Condition | Error Code |
|---|---|
| Backend unavailable | `NCI-E-HARDWARE` |
| Unsupported dtype | `NCI-E-REPRESENTATION` |
| Memory allocation failure | `NCI-E-STORE` |
| Data transfer failure | `NCI-E-STORE` |
| Execution failure | `NCI-E-HARDWARE` |

---

## 7. Conformance Requirements

1. All backends MUST implement Backend trait.
2. Backend selection MUST be deterministic.
3. Data transfer MUST be explicit.
4. Backends MUST be thread-safe.
5. Backends MUST report capabilities correctly.
