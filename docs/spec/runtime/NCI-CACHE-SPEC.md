# NCI Cache Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §16.2; `runtime/NCI-RUNTIME-SPEC.md`

---

## Purpose

This document specifies the NCI Cache: cache addressing, eviction policy, cache hierarchy, and cache coherence.

## Scope

Cache addressing, LRU eviction, cache levels, cache warming, cache invalidation, and cache metrics.

---

## 1. Cache Addressing

### 1.1 Cache Key

Cache keys MUST be:

```
(CellId, RepresentationId)
```

NOT file offset, NOT block index, NOT hardware-specific layout.

### 1.2 Key Structure

```rust
struct CacheKey {
    cell_id: CellId,          // 32 bytes
    representation_id: RepresentationId,  // 4 bytes
}

impl CacheKey {
    fn hash(&self) -> u64 {
        blake3_256_64(&self.cell_id.0, &self.representation_id.0)
    }
}
```

---

## 2. Cache Hierarchy

### 2.1 Levels

```
L1 Cache (per-backend, fast, small)
    ↓
L2 Cache (shared, medium)
    ↓
L3 Cache (system, large, slow)
    ↓
NCL Storage (disk, slowest)
```

### 2.2 Level Sizes

| Level | Default Size | Eviction |
|---|---|---|
| L1 | 256 MiB | LRU |
| L2 | 2 GiB | LRU |
| L3 | 16 GiB | LRU |
| NCL Storage | Unlimited | N/A |

### 2.3 Level Selection

```rust
fn get(&self, key: &CacheKey) -> Option<CachedCell> {
    // Check L1
    if let Some(cell) = self.l1.get(key) {
        self.l1.hit();
        return Some(cell);
    }
    self.l1.miss();
    
    // Check L2
    if let Some(cell) = self.l2.get(key) {
        self.l2.hit();
        self.l1.insert(key, cell.clone());
        return Some(cell);
    }
    self.l2.miss();
    
    // Check L3
    if let Some(cell) = self.l3.get(key) {
        self.l3.hit();
        self.l2.insert(key, cell.clone());
        self.l1.insert(key, cell.clone());
        return Some(cell);
    }
    self.l3.miss();
    
    None
}
```

---

## 3. Eviction Policy

### 3.1 LRU (Least Recently Used)

```rust
struct LruCache {
    capacity: u64,
    entries: HashMap<CacheKey, CacheEntry>,
    access_order: LinkedList<CacheKey>,
}

struct CacheEntry {
    data: CachedCell,
    access_count: u64,
    last_access: u64,
}

impl LruCache {
    fn get(&mut self, key: &CacheKey) -> Option<CachedCell> {
        if let Some(entry) = self.entries.get_mut(key) {
            entry.last_access = now();
            entry.access_count += 1;
            self.access_order.retain(|k| k != key);
            self.access_order.push_back(key.clone());
            return Some(entry.data.clone());
        }
        None
    }
    
    fn insert(&mut self, key: CacheKey, data: CachedCell) {
        if self.current_size() >= self.capacity {
            self.evict();
        }
        
        self.entries.insert(key.clone(), CacheEntry {
            data,
            access_count: 1,
            last_access: now(),
        });
        self.access_order.push_back(key);
    }
    
    fn evict(&mut self) {
        if let Some(oldest) = self.access_order.pop_front() {
            self.entries.remove(&oldest);
        }
    }
}
```

---

## 4. Cache Operations

### 4.1 Insert

```rust
fn insert(&mut self, key: CacheKey, cell: CachedCell) {
    self.stats.inserts += 1;
    
    // Check if already cached
    if self.entries.contains_key(&key) {
        self.stats.duplicates += 1;
        return;
    }
    
    // Evict if necessary
    while self.current_size() + cell.size() > self.capacity {
        self.evict();
    }
    
    // Insert
    self.entries.insert(key, CacheEntry::new(cell));
    self.access_order.push_back(key);
}
```

### 4.2 Get

```rust
fn get(&mut self, key: &CacheKey) -> Option<CachedCell> {
    self.stats.accesses += 1;
    
    if let Some(entry) = self.entries.get_mut(key) {
        self.stats.hits += 1;
        entry.last_access = now();
        entry.access_count += 1;
        return Some(entry.data.clone());
    }
    
    self.stats.misses += 1;
    None
}
```

### 4.3 Invalidate

```rust
fn invalidate(&mut self, key: &CacheKey) {
    self.entries.remove(key);
    self.access_order.retain(|k| k != key);
    self.stats.invalidations += 1;
}
```

### 4.4 Clear

```rust
fn clear(&mut self) {
    self.entries.clear();
    self.access_order.clear();
    self.stats.clears += 1;
}
```

---

## 5. Cache Warming

### 5.1 Prefetch-Based Warming

```rust
fn warm_cache(&mut self, cell_ids: &[CellId]) {
    for cell_id in cell_ids {
        let cell = self.reader.read_cell(cell_id)?;
        let key = CacheKey::new(*cell_id, cell.representation);
        self.insert(key, CachedCell::new(cell));
    }
}
```

### 5.2 Dependency-Based Warming

```rust
fn warm_from_dependencies(&mut self, current_cell: &CellId) {
    let deps = self.reader.get_dependencies(current_cell);
    self.warm_cache(&deps);
}
```

---

## 6. Cache Invalidation

### 6.1 Invalidation Triggers

Cache MUST be invalidated when:
- Reader is closed
- File is modified (detected via metadata)
- Explicit `clear()` call
- Memory pressure (system signal)

### 6.2 Selective Invalidation

```rust
fn invalidate_by_cell(&mut self, cell_id: &CellId) {
    let keys: Vec<_> = self.entries.keys()
        .filter(|k| k.cell_id == *cell_id)
        .cloned()
        .collect();
    
    for key in keys {
        self.invalidate(&key);
    }
}
```

---

## 7. Cache Metrics

### 7.1 Metrics Structure

```rust
struct CacheMetrics {
    hits: u64,
    misses: u64,
    inserts: u64,
    evictions: u64,
    invalidations: u64,
    duplicates: u64,
    clears: u64,
    total_size: u64,
    entry_count: usize,
}

impl CacheMetrics {
    fn hit_rate(&self) -> f64 {
        if self.hits + self.misses == 0 {
            return 0.0;
        }
        self.hits as f64 / (self.hits + self.misses) as f64
    }
}
```

### 7.2 Metrics Exposure

Cache metrics MUST be exposed via observability:
- `nci_cache_hit_rate` (Histogram)
- `nci_cache_size` (Gauge)
- `nci_cache_evictions` (Counter)

---

## 8. Validation

1. Cache key MUST be (CellId, RepresentationId).
2. Eviction MUST be LRU.
3. Cache MUST be invalidated on file change.
4. Metrics MUST be accurate.
5. Cache size MUST respect configured limits.

---

## 9. Error Handling

| Condition | Error Code |
|---|---|
| Cache full | N/A (evict) |
| Cache corrupted | `NCI-E-CORRUPT` |
| File changed | Invalidation |

---

## 10. Conformance Requirements

1. Cache key MUST be (CellId, RepresentationId).
2. Cache MUST implement LRU eviction.
3. Cache MUST support multi-level hierarchy.
4. Cache MUST be invalidated on file change.
5. Cache MUST expose metrics.
6. Cache MUST be thread-safe.
