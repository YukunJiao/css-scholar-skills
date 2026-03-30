# Ideology Scaling and Trajectory Analysis with Embeddings — Detailed Reference

> Primary source: Yantseva, V. (2025). "Climate change denial and ideology in Swedish online media: measuring ideology change using a computational approach." *Journal of Computational Social Science*, 8:14. https://doi.org/10.1007/s42001-024-00343-x
> Related: Boholm et al. (2026) on diachronic embedding validation.

## Overview

Ideology scaling with text embeddings treats discourse *positions* as points in a vector space and tracks their movement over time. Rather than classifying texts into fixed categories, this approach:

1. Embeds each actor/document in a shared semantic space
2. Reduces to 2D for visualisation and trajectory tracking
3. Identifies *clusters* of similar positions
4. Tracks *trajectories* — how positions move across time windows

This is an alternative to ALC regression when the goal is positional mapping rather than covariate-specific semantic shift.

---

## When to Use Ideology Scaling vs. ALC Regression

| Dimension | Ideology scaling (Yantseva) | ALC regression (conText) |
|-----------|---------------------------|--------------------------|
| **Goal** | Map where actors/groups sit in semantic space | Estimate effect of covariate on a target word's meaning |
| **Unit of analysis** | Actor/group (aggregated texts) | Context window around a target word |
| **Hypothesis** | Where do groups cluster? Do they move? | How does X's meaning differ by group or period? |
| **Output** | 2D positional map + trajectories | Regression coefficients + NNS |
| **Best for** | Polarisation, radicalisation, ideological divergence | Semantic change of specific concepts across groups |

Use both when feasible — ALC regression for specific conceptual questions; ideology scaling for corpus-level positional overview.

---

## Method: Embedding-Based Ideology Scaling

### Step 1: Choose an embedding model

**Pre-trained sentence embeddings** (e.g., LASER, SBERT) are preferred when:
- The corpus is in a non-English language
- Cross-time alignment is required without re-alignment procedures
- The corpus is too small to train reliable custom embeddings

**Custom word/document embeddings** (GloVe, word2vec) are preferred when:
- The corpus has distinctive vocabulary (historical, technical, domain-specific)
- Semantic change in *specific* vocabulary is the focus
- You need interpretable neighbouring terms

```r
# Option A: Pre-trained sentence embeddings (LASER via Python)
# Use reticulate to call Python from R
library(reticulate)
laser <- import("laserembeddings")
encoder <- laser$Laser()

# Embed each document
docs_embedded <- encoder$embed_sentences(texts, lang = "en")

# Option B: Custom GloVe word vectors, aggregate to document level
# Average word vectors weighted by TF-IDF
embed_documents <- function(texts, word_vectors, tfidf_weights) {
  purrr::map_dfr(texts, function(text) {
    words <- unlist(quanteda::tokens(quanteda::corpus(text)))
    in_vocab <- intersect(words, rownames(word_vectors))
    if (length(in_vocab) == 0) return(rep(0, ncol(word_vectors)))
    weights <- tfidf_weights[in_vocab]
    colSums(word_vectors[in_vocab, ] * weights, na.rm = TRUE) / sum(weights, na.rm = TRUE)
  })
}
```

### Step 2: Aggregate to actor/group level per time period

```r
# Aggregate document embeddings to group-year mean
group_embeddings <- docs_df |>
  dplyr::group_by(group_id, year) |>
  dplyr::summarise(
    dplyr::across(starts_with("dim_"), mean, na.rm = TRUE),
    n_docs = dplyr::n(),
    .groups = "drop"
  ) |>
  dplyr::filter(n_docs >= 10)   # minimum document threshold per group-year
```

### Step 3: Reduce dimensionality

```r
library(Rtsne)

# t-SNE: non-linear, good for visualising clusters
# Note: t-SNE is stochastic — set seed and run multiple times
set.seed(42)
tsne_result <- Rtsne::Rtsne(
  as.matrix(group_embeddings[, starts_with("dim_")]),
  dims        = 2,
  perplexity  = 30,    # typical range 5–50; larger = more global structure
  max_iter    = 1000,
  verbose     = FALSE
)

group_embeddings$x <- tsne_result$Y[, 1]
group_embeddings$y <- tsne_result$Y[, 2]

# PCA: linear, preserves global structure, reproducible — often better for trajectory analysis
pca_result <- prcomp(as.matrix(group_embeddings[, starts_with("dim_")]),
                     center = TRUE, scale. = FALSE)
group_embeddings$pc1 <- pca_result$x[, 1]
group_embeddings$pc2 <- pca_result$x[, 2]
```

**Note on t-SNE vs PCA for trajectory analysis**: t-SNE distorts distances, so trajectories across time can appear to cross or reverse due to the non-linear mapping. For *temporal* trajectories, PCA is safer; use t-SNE only for static cluster visualisation within a single time period.

### Step 4: Cluster groups in ideological space

```r
library(cluster)

# K-means (requires pre-specifying k)
set.seed(42)
km <- kmeans(as.matrix(group_embeddings[, c("pc1", "pc2")]),
             centers = 5, nstart = 25)
group_embeddings$cluster <- as.factor(km$cluster)

# Silhouette analysis to choose k
sil_widths <- sapply(2:10, function(k) {
  km_k <- kmeans(as.matrix(group_embeddings[, c("pc1", "pc2")]),
                 centers = k, nstart = 25)
  mean(cluster::silhouette(km_k$cluster,
                           dist(as.matrix(group_embeddings[, c("pc1", "pc2")])))[, 3])
})
best_k <- which.max(sil_widths) + 1
```

### Step 5: Trajectory analysis

Track each group's movement between consecutive time periods:

```r
library(ggplot2)
library(ggrepel)

# Compute displacement vectors (trajectory)
trajectories <- group_embeddings |>
  dplyr::arrange(group_id, year) |>
  dplyr::group_by(group_id) |>
  dplyr::mutate(
    dx = pc1 - dplyr::lag(pc1),
    dy = pc2 - dplyr::lag(pc2),
    displacement = sqrt(dx^2 + dy^2)
  ) |>
  dplyr::filter(!is.na(dx))

# Visualise
ggplot(group_embeddings, aes(x = pc1, y = pc2, colour = cluster)) +
  geom_point(aes(size = n_docs), alpha = 0.6) +
  geom_segment(data = trajectories,
               aes(xend = pc1, yend = pc2,
                   x = pc1 - dx, y = pc2 - dy),
               arrow = arrow(length = unit(0.2, "cm")),
               alpha = 0.4) +
  facet_wrap(~ year) +
  labs(x = "PC1", y = "PC2", colour = "Cluster",
       title = "Ideological trajectories by year")
```

---

## Interpreting the Ideological Space

Unlike word association analyses, the axes of an ideology-scaled embedding space are not labelled *a priori*. Interpretation requires:

1. **Identify the extreme ends of each axis** — which groups anchor each pole?
2. **Retrieve top vocabulary** for each pole using NNS or high-TF-IDF terms from texts at each extreme
3. **Name the dimension** based on the substantive contrast the vocabulary reveals

```r
# Find most distinctive vocabulary for each cluster
top_vocab_by_cluster <- function(docs_df, cluster_assignments, word_vectors, k = 20) {
  lapply(unique(cluster_assignments), function(cl) {
    cluster_docs <- docs_df[cluster_assignments == cl, "text"]
    toks <- quanteda::tokens(quanteda::corpus(cluster_docs))
    dfm_cl <- quanteda::dfm(toks)
    # Top terms by frequency within cluster
    top_features <- quanteda.textstats::textstat_frequency(dfm_cl, n = k)
    top_features$feature
  })
}
```

Yantseva (2025) notes that dimension labels should be *post hoc* and grounded in the vocabulary that anchors each pole — imposing a label before inspecting the vocabulary is a validity threat.

---

## Validating Ideological Positions

Cross-validate the computed positions against external indicators:

| External indicator | Validation test |
|-------------------|-----------------|
| Known party affiliation or editorial stance | Do groups with known ideology cluster as expected? |
| Fact-checker labels (denial vs. mainstream) | Do labelled groups occupy predicted regions of the space? |
| Survey data on group beliefs | Correlate group position on PC1/PC2 with survey-measured attitude scales |
| Human coding of text samples | Do coders assign texts to the same cluster as the algorithm? |

```r
# Example: correlate PC1 position with external rating
model_valid <- lm(pc1 ~ external_rating, data = group_embeddings)
summary(model_valid)
```

---

## Key Methodological Cautions

| Issue | Risk | Mitigation |
|-------|------|-----------|
| Pre-trained embeddings impose external semantic structure | Corpus-specific meaning may be distorted | Use custom embeddings if corpus is large enough |
| t-SNE stochasticity | Trajectories can appear spurious | Use PCA for temporal analysis; t-SNE only for static cluster plots |
| Aggregation masks within-group heterogeneity | Group-level position hides dissent | Report within-group variance alongside mean position |
| Axes have no inherent meaning | Dimension labels are interpretive | Validate labels with vocabulary analysis and external indicators |
| Small n per group-year | Noisy mean embedding | Set minimum document threshold (≥ 10 per cell) |

---

## Sources

- Yantseva, V. (2025). Climate change denial and ideology in Swedish online media. *Journal of Computational Social Science*, 8:14. https://doi.org/10.1007/s42001-024-00343-x
- Boholm, M., et al. (2026). Tracking dogwhistles online and through time using distributional semantics. *Journal of Computational Social Science*, 9:33. https://doi.org/10.1007/s42001-026-00464-5
- `SKILL.md` in this directory — ALC regression (conText) as a complement to ideology scaling
- `extension-directions.md` in `research-ideation/references/` — comparative corpus design principles
