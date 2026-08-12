# NCI Routing Specification

**Version:** 1.0.0
**Status:** Normative
**Authority:** NCI Engineering Contract v1.0.0
**References:** `00-NCI-ENGINEERING-CONTRACT.md` §14; `runtime/NCI-RUNTIME-SPEC.md`

---

## Purpose

This document specifies the NCI Routing Engine: routing algorithm, difficulty estimation, Cell selection, and the routing contract.

## Scope

Routing algorithm, difficulty estimation, Cell selection, compute budget integration, fallback policy, and routing state.

---

## 1. Routing Algorithm

### 1.1 Input

```
Input Query
    ├── context (previous activations)
    ├── difficulty estimation
    ├── hardware budget (max_cells, max_bytes)
    ├── latency budget (max_latency_us)
    ├── memory budget (max_bytes)
    └── energy budget (if applicable)
```

### 1.2 Output

```
Routing Output
    ├── selected_cells: [CellId, CellId, ...]
    ├── activation_order: [CellId, CellId, ...]
    ├── confidence_scores: [0.98, 0.94, 0.89, ...]
    └── fallback_policy: (recursive or early_exit)
```

### 1.3 Algorithm

```rust
fn route(
    &self,
    context: &ExecutionContext,
    budget: &ComputeBudget,
) -> Result<RouteDecision> {
    // 1. Estimate difficulty
    let difficulty = self.estimate_difficulty(context)?;
    
    // 2. Determine target cell count
    let target_cells = self.difficulty_to_cell_count(difficulty);
    
    // 3. Score all candidate cells
    let scored_cells = self.score_cells(context, &difficulty)?;
    
    // 4. Select top cells within budget
    let selected = self.select_within_budget(scored_cells, budget)?;
    
    // 5. Determine activation order
    let order = self.topological_sort(&selected)?;
    
    // 6. Assign confidence scores
    let scores = self.compute_confidence(&selected, context)?;
    
    // 7. Determine fallback policy
    let fallback = self.determine_fallback(&selected, budget)?;
    
    Ok(RouteDecision {
        selected_cells: selected,
        activation_order: order,
        confidence_scores: scores,
        fallback_policy: fallback,
    })
}
```

---

## 2. Difficulty Estimation

### 2.1 Estimation Algorithm

```rust
fn estimate_difficulty(&self, context: &ExecutionContext) -> Result<Difficulty> {
    let mut score = 0.0;
    
    // 1. Token-level complexity
    score += self.token_complexity(context);
    
    // 2. Context length
    score += self.context_length_factor(context);
    
    // 3. Historical accuracy
    score += self.historical_accuracy(context);
    
    // 4. Query type
    score += self.query_type_factor(context);
    
    // 5. Map to difficulty level
    let difficulty = match score {
        s if s < 0.2 => Difficulty::Easy,
        s if s < 0.5 => Difficulty::Medium,
        s if s < 0.8 => Difficulty::Hard,
        _ => Difficulty::VeryHard,
    };
    
    Ok(difficulty)
}
```

### 2.2 Difficulty Levels

| Difficulty | Typical Cell Count | Description |
|---|---|---|
| Easy | 5–10 | Simple queries, fact retrieval |
| Medium | 50–100 | Moderate reasoning, multi-step |
| Hard | 500–1000 | Complex reasoning, many steps |
| Very Hard | Recursive | Requires recursive execution |

### 2.3 Difficulty-to-Cell-Count Mapping

```rust
fn difficulty_to_cell_count(&self, difficulty: Difficulty) -> usize {
    match difficulty {
        Difficulty::Easy => 8,
        Difficulty::Medium => 75,
        Difficulty::Hard => 750,
        Difficulty::VeryHard => self.budget.max_cells,  // Use full budget
    }
}
```

---

## 3. Cell Scoring

### 3.1 Scoring Factors

```rust
fn score_cells(
    &self,
    context: &ExecutionContext,
    difficulty: &Difficulty,
) -> Result<Vec<(CellId, f64)>> {
    let mut scored = Vec::new();
    
    for cell_id in self.cell_index.all_cells() {
        let cell = self.reader.resolve_cell(cell_id)?;
        
        let mut score = 0.0;
        
        // 1. Relevance to query
        score += self.relevance_score(&cell, context);
        
        // 2. Importance score (from attributes)
        score += self.importance_score(&cell);
        
        // 3. Access frequency
        score += self.frequency_score(&cell);
        
        // 4. Dependency coverage
        score += self.dependency_score(&cell, context);
        
        // 5. Hardware fit
        score += self.hardware_fit_score(&cell);
        
        scored.push((*cell_id, score));
    }
    
    // Sort by score descending
    scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    
    Ok(scored)
}
```

### 3.2 Relevance Score

```rust
fn relevance_score(&self, cell: &Cell, context: &ExecutionContext) -> f64 {
    // Based on cell's semantic similarity to query
    // Uses cell attributes and routing table
    let query_embedding = context.query_embedding();
    let cell_embedding = cell.attributes.get("nci.embedding");
    
    cosine_similarity(query_embedding, cell_embedding)
}
```

---

## 4. Budget Integration

### 4.1 Selection Within Budget

```rust
fn select_within_budget(
    &self,
    scored_cells: Vec<(CellId, f64)>,
    budget: &ComputeBudget,
) -> Result<Vec<CellId>> {
    let mut selected = Vec::new();
    let mut total_bytes = 0u64;
    let mut total_cells = 0usize;
    
    for (cell_id, score) in scored_cells {
        let cell_size = self.estimate_cell_size(&cell_id);
        
        if total_cells >= budget.max_cells {
            break;
        }
        if total_bytes + cell_size > budget.max_bytes {
            break;
        }
        
        selected.push(cell_id);
        total_cells += 1;
        total_bytes += cell_size;
    }
    
    Ok(selected)
}
```

---

## 5. Fallback Policy

### 5.1 Recursive Fallback

When initial routing is insufficient:
1. Increase difficulty level
2. Re-route with expanded budget
3. If still insufficient, try again (up to max_depth)

### 5.2 Early-Exit Fallback

When budget is exhausted:
1. Return best-effort result
2. Include confidence metric
3. Log budget exhaustion event

### 5.3 Fallback Selection

```rust
fn determine_fallback(
    &self,
    selected: &[CellId],
    budget: &ComputeBudget,
) -> Result<FallbackPolicy> {
    let avg_confidence = self.average_confidence(selected);
    
    if avg_confidence > 0.8 && budget.remaining_depth() > 0 {
        Ok(FallbackPolicy::Recursive)
    } else {
        Ok(FallbackPolicy::EarlyExit)
    }
}
```

---

## 6. Routing State

### 6.1 State Structure

```rust
struct RoutingState {
    difficulty_history: Vec<Difficulty>,
    confidence_history: Vec<f64>,
    cell_access_counts: HashMap<CellId, u64>,
    routing_table: RoutingTable,
}
```

### 6.2 Routing Table

```rust
struct RoutingTable {
    entries: Vec<RoutingEntry>,
}

struct RoutingEntry {
    query_pattern: QueryPattern,
    target_cells: Vec<CellId>,
    scores: Vec<f64>,
}
```

---

## 7. Validation

1. Difficulty estimation MUST be deterministic.
2. Cell scoring MUST be deterministic.
3. Budget MUST be enforced.
4. Fallback policy MUST be explicitly chosen.
5. Routing state MUST be updated after each route.

---

## 8. Error Handling

| Condition | Error Code |
|---|---|
| No cells available | `NCI-E-ROUTING` |
| Budget exhausted | `NCI-E-BUDGET` |
| Routing table empty | `NCI-E-ROUTING` |
| Difficulty estimation failure | `NCI-E-ROUTING` |

---

## 9. Conformance Requirements

1. Router MUST estimate difficulty.
2. Router MUST select cells within budget.
3. Router MUST support recursive and early-exit fallback.
4. Router MUST update routing state.
5. Router MUST be deterministic for identical inputs.
