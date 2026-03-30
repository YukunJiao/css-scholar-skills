# Anchor Quality Metrics — Detailed Reference

> Based on: Ohamadike, Durrheim & Primus (2026), "Anchoring race: improving the construction of race dimensions in word embeddings." *Journal of Computational Social Science* 9:20.

## Why Anchor Quality Matters

Choosing anchor words (or word pairs) intuitively is the standard practice but introduces silent failures: anchors that *seem* theoretically appropriate may not form a coherent semantic dimension in the embedding space. Poor-quality anchors produce:

- Offset vectors pointing in inconsistent directions → noisy regression coefficients
- A dimension that captures multiple conflated semantic distinctions (e.g., colour vs. race; profession vs. gender)
- Results that do not replicate across embedding models or corpora

The two metrics below — PairDir and ACS — diagnose these problems before analysis.

---

## Metric 1: PairDir (Pairwise Direction)

**What it measures**: Whether the offset vectors for each anchor pair point in the same direction — i.e., whether all pairs capture the *same* semantic contrast.

**Formula**:

```
PairDir = (1 / n(n-1)) * Σᵢ≠ⱼ cos(dᵢ, dⱼ)

where:
  n   = number of word pairs
  dᵢ  = wᵢ⁺ − wᵢ⁻   (offset vector for pair i)
  dⱼ  = wⱼ⁺ − wⱼ⁻   (offset vector for pair j)
  cos = cosine similarity
```

**Interpretation**:

| PairDir value | Meaning |
|---------------|---------|
| Near 1.0 | All pairs point in the same direction → high internal consistency |
| Near 0.0 | Pairs point in random directions → dimension is incoherent |
| Negative | Pairs are actively opposed → anchors capture different contrasts |

**Rule of thumb**: PairDir > 0.3 is acceptable; > 0.5 is strong.

**Implementation in R**:

```r
# Given a matrix of offset vectors (rows = pairs, cols = embedding dims)
compute_pairdir <- function(offsets) {
  # Normalise each row
  norms    <- sqrt(rowSums(offsets^2))
  offsets  <- offsets / norms
  # Pairwise cosine similarities
  n        <- nrow(offsets)
  cos_sim  <- tcrossprod(offsets)   # n × n similarity matrix
  # Average of off-diagonal elements
  mean(cos_sim[lower.tri(cos_sim)])
}

# Example: anchor pairs for "science" vs "religion" dimension
science_words  <- c("science", "experiment", "observation")
religion_words <- c("religion", "faith", "scripture")

offsets <- mapply(function(pos, neg) {
  embeddings[pos, ] - embeddings[neg, ]
}, science_words, religion_words)

pairdir <- compute_pairdir(t(offsets))
```

---

## Metric 2: ACS (Axis Coherence Score)

**What it measures**: Whether the offset vectors are organised along a *single dominant direction* — i.e., whether they define a one-dimensional axis or scatter across multiple directions.

**Computation**: Apply PCA to the offset matrix. ACS = variance explained by PC1.

```
ACS = var_PC1 / Σᵢ var_PCᵢ
```

**Interpretation**:

| ACS value | Meaning |
|-----------|---------|
| > 0.6 | A single semantic axis dominates → clean dimension |
| 0.3–0.6 | Moderate coherence — check for outlier pairs |
| < 0.3 | Dimension is multi-directional → split into sub-dimensions or revise anchors |

**Implementation in R**:

```r
compute_acs <- function(offsets) {
  pca <- prcomp(offsets, center = TRUE, scale. = FALSE)
  var_explained <- pca$sdev^2 / sum(pca$sdev^2)
  var_explained[1]   # proportion explained by PC1
}

acs <- compute_acs(t(offsets))
```

**When ACS is low**: visualise the PCA biplot to identify which pairs deviate from the main direction. Either exclude them or treat them as capturing a different sub-dimension.

---

## PairDir vs ACS: Complementarity

| Scenario | PairDir | ACS | Diagnosis |
|----------|---------|-----|-----------|
| All pairs parallel, tight | High | High | Ideal — coherent, reliable axis |
| Pairs mostly parallel, one outlier | High | Medium | Remove outlier pair |
| Pairs scattered, no pattern | Low | Low | Wrong anchors; revise on theoretical grounds |
| Two clusters of pairs in opposite directions | Near 0 | Low | Dimension captures two contrasts — split or reframe |

Both metrics should be computed and reported in the appendix/supplementary material. If they disagree, investigate the outlier pairs with `kwic()` before deciding.

---

## Generalising Beyond Anchor *Pairs*

When using single anchor words rather than contrastive pairs (as in `conText`), the same logic applies:

- Replace "offset vector dᵢ" with the embedding vector for anchor word *i*
- PairDir becomes the average cosine similarity among all anchor embeddings
- High PairDir → anchors cluster in the same semantic region → the group is coherent

```r
# Check cluster coherence for a list of anchor words
compute_anchor_coherence <- function(anchors, embeddings) {
  vecs  <- embeddings[anchors, , drop = FALSE]
  norms <- sqrt(rowSums(vecs^2))
  vecs  <- vecs / norms
  cos   <- tcrossprod(vecs)
  mean(cos[lower.tri(cos)])
}

# Example
anchors_epistemic <- c("reason", "evidence", "observation", "experiment")
coherence <- compute_anchor_coherence(anchors_epistemic, word_vectors)
# If < 0.2, the group is too semantically diffuse
```

---

## Integration Into the Anchor Selection Workflow

These metrics extend the three-stage process in `anchor-selection.md`:

**After Stage 2 (frequency validation)**, add:

> **Stage 2b — Metric validation**
> 1. Compute PairDir (for contrastive dimensions) or inter-anchor cosine coherence (for single-anchor groups)
> 2. Compute ACS
> 3. Exclude anchors with PairDir contribution < mean − 1 SD
> 4. Report both metrics in supplementary material

**Decision rule**:

```
If PairDir > 0.3 AND ACS > 0.4:
  → proceed with anchor set
Elif PairDir 0.1–0.3:
  → remove worst-contributing pair, re-test
Else:
  → revise theoretical operationalisation
```

---

## Sources and Further Reading

- Ohamadike, N., Durrheim, K., & Primus, M. (2026). Anchoring race: improving the construction of race dimensions in word embeddings. *Journal of Computational Social Science*, 9:20. https://doi.org/10.1007/s42001-025-00449-w
- Boutyline, A., & Johnston, C. (cited in Ohamadike et al.) — original PairDir development
- `anchor-selection.md` in this directory — full three-stage anchor selection workflow
