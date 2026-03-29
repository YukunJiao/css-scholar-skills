---
name: embedding-analysis
description: Measure semantic change and group differences in text corpora using ALC word embedding regression (conText) in R — covering GloVe training, anchor selection, formula specification, and results interpretation.
version: 1.0.0
author: css-skills
tags: [conText, ALC, GloVe, word-embeddings, semantic-change, R, quanteda, text2vec, NNS, diachronic, computational-text-analysis]
dependencies: [conText, quanteda, text2vec, dplyr, ggplot2, ggrepel]
---

# ALC Word Embedding Regression with `conText`

## When to Use

| Goal | Method |
|------|--------|
| Measure semantic change across time periods | ALC regression with time variable as factor |
| Characterise the semantic neighbourhood of a word | Bootstrap NNS (`nns()`) |
| Estimate how group/covariate membership shifts word meaning | `conText()` regression — covariate coefficients |
| Compare meaning across discourse communities | Include community as factor in formula |
| Align embeddings across sub-corpora | Train GloVe jointly on all sub-corpora — no Procrustes needed |

**Do NOT** train separate GloVe models per period and compare vectors directly — the resulting spaces are not aligned and cosine similarities are not comparable across models.

---

## Core Pipeline

```r
# 1. Train a single GloVe model on the entire corpus
fcm        <- build_fcm(tokens, window = 6, min_count = 10)
embeddings <- train_glove(fcm, dims = 100, iter = 10)

# 2. Compute the ALC transformation matrix
transform  <- conText::compute_transform(fcm, embeddings, weighting = "log")

# 3. Build a DFM for candidate vocabulary (used in NNS)
dfm        <- quanteda::dfm(tokens) |> quanteda::dfm_trim(min_termfreq = 5)

# 4. For each target word — extract context windows
toks_ctx   <- conText::get_context(
  x = tokens, target = "TARGET_WORD",
  window = 6, valuetype = "fixed",
  hard_cut = FALSE, verbose = FALSE
)

# 5. Bootstrap nearest-neighbour analysis
nns <- conText::nns(
  x                = toks_ctx,
  pre_trained      = embeddings,
  transform        = TRUE,
  transform_matrix = transform,
  candidates       = dfm,
  N                = 15,
  bootstrap        = TRUE,
  num_bootstraps   = 100,
  confidence_level = 0.95,
  verbose          = FALSE
)

# 6. ALC regression — covariate effects on word meaning
model <- conText::conText(
  formula          = . ~ group + time_period + covariate,
  data             = toks_ctx,
  pre_trained      = embeddings,
  transform        = TRUE,
  transform_matrix = transform,
  bootstrap        = FALSE,   # set TRUE for final paper run
  permute          = FALSE,   # set TRUE for final paper run
  num_bootstraps   = 100,
  verbose          = FALSE
)
```

---

## GloVe Training

```r
train_glove <- function(fcm, dims = 100, iter = 10) {
  glove   <- text2vec::GlobalVectors$new(
    rank          = dims,
    x_max         = 10,
    learning_rate = 0.05
  )
  wv_main <- glove$fit_transform(fcm, n_iter = iter, convergence_tol = 1e-3)
  wv_main + t(glove$components)   # convention: sum word + context vectors
}
```

### Hyperparameter guidance

| Parameter | Typical value | Notes |
|-----------|--------------|-------|
| `dims` | 100–300 | 100 sufficient for most social science corpora; 300 for large/diverse corpora |
| `window` | 6 | Larger windows → more topical associations; smaller → more syntactic |
| `min_count` | 5–20 | Remove OCR noise and hapax legomena; tune based on corpus size |
| `iter` | 10–20 | Monitor convergence; stop when loss plateaus |
| `x_max` | 10 | Standard; rarely needs changing |

### Why joint training?

Training GloVe on all time periods / sub-corpora jointly ensures all ALC context embeddings project into the **same vector space**. Cross-period cosine comparisons are then directly interpretable as semantic distance — no alignment procedure required. Separate per-period models produce non-comparable spaces.

---

## Formula Specification

```r
# General template
formula = . ~ time_period + group + control_1 + control_2

# Example: diachronic analysis with discourse community variation
formula = . ~ decade + press_type + is_front_page + article_length_bin

# Example: cross-national comparison controlling for document length
formula = . ~ country + year + length_bin
```

### Covariate types

| Covariate | Type | Interpretation |
|-----------|------|----------------|
| Time period | Factor (reference = earliest) | Coefficients = direction of semantic shift relative to baseline |
| Group / community | Factor | Coefficients = direction of semantic difference from reference group |
| Document length | Factor (bins) | Controls for depth of treatment; usually strong predictor |
| Front page / prominence | Logical | Controls for editorial salience |

### Continuous vs. factor time

- Use `factor(year)` or `decade` when you expect **non-linear** or **discrete** change
- Use `year` (continuous) only when change is expected to be linear over time
- Factor encoding: coefficients for each period are independent contrasts against the reference

### Interpreting normed estimates

The `conText` output reports coefficients as **normed estimates** — the cosine similarity between the coefficient vector and the pre-trained embedding of the target word:

- **Positive** normed estimate: the semantic neighbourhood shifted *toward* the target word's own embedding (meaning became more like itself, or was reinforced)
- **Negative**: shifted away
- **Near zero**: covariate did not shift the meaning of this word
- **Compare across words**: asymmetric magnitudes reveal which vocabulary is more sensitive to a given covariate

---

## Anchor Word Selection

### Principles

1. **Sufficient frequency**: ≥100 context-window instances per group/period for reliable ALC estimates. Check with `quanteda::ntoken()` on the context subset.
2. **Conceptual validity**: the word must directly index the concept you are studying, not a proxy or peripheral indicator.
3. **Semantic coherence**: words in the same group should cluster together in the embedding space (verify with NNS).
4. **No confounds**: check for polysemy, proper noun overlap, and OCR errors (inspect with `quanteda::kwic()`).
5. **Theory-driven**: each word or group of words should correspond to a distinct theoretical construct.

### Frequency check

```r
# Check frequency per group/period before finalising word list
dfm_grouped <- quanteda::dfm(tokens) |>
  quanteda::dfm_group(groups = quanteda::docvars(tokens, "decade"))

# Extract frequencies for candidate words
freq_table <- quanteda::featfreq(dfm_grouped)[candidate_words, ]
# Rule: drop any word with < 100 occurrences in any group
freq_table[apply(freq_table, 1, min) < 100, ]
```

### OCR and polysemy checks

```r
# Inspect context windows for OCR artifacts or unintended senses
quanteda::kwic(tokens, "target_word", window = 5) |> head(30)
# Common problems: city/person names, OCR garbling, archaic spellings
```

### Exclusion decision table

| Issue | Example | Decision |
|-------|---------|----------|
| Proper noun overlap | "providence" = city name | Exclude or use longer phrase |
| Agent noun (role, not concept) | "physician" = social role | Exclude if testing knowledge boundary |
| Sparse in target period | n < 100 in any group | Exclude; unreliable estimates |
| OCR noise dominates context | garbled tokens in KWIC | Exclude |
| Near-synonym of another anchor | overlapping NNS | Keep only one; reduces redundancy |

---

## Results Interpretation

### NNS plots

```r
# Standard NNS figure template
nns_data |>
  ggplot(aes(x = reorder(feature, estimate), y = estimate,
             ymin = lower.ci, ymax = upper.ci, colour = group)) +
  geom_pointrange() +
  coord_flip() +
  facet_wrap(~ target_word, scales = "free_y") +
  labs(x = NULL, y = "Cosine similarity", colour = NULL)
```

- **Non-overlapping CIs across groups**: stable semantic neighbourhood → no change
- **Overlapping CIs**: neighbourhood shifted between groups/periods
- **Unexpected neighbours**: flag OCR issues or polysemy

### Covariate heatmap

```r
# Heatmap of normed estimates across all target words and covariates
coef_all |>
  ggplot(aes(x = coefficient, y = target, fill = normed_estimate)) +
  geom_tile() +
  scale_fill_gradient2(low = "blue", mid = "white", high = "red", midpoint = 0) +
  labs(fill = "Normed\nestimate")
```

Reading the heatmap:
- **Rows**: target words — compare to see which concepts are most sensitive to each covariate
- **Columns**: covariates — compare to see which factors most strongly shift word meaning
- **Magnitude**: normed estimate range ±1; values > 0.3 are substantively large

### bootstrap / permute for final run

```r
# Development (fast — no p-values)
bootstrap = FALSE, permute = FALSE

# Final paper run (slow — produces CIs and permutation p-values)
bootstrap = TRUE, permute = TRUE, num_bootstraps = 100
```

Set these in your `run_context()` wrapper function before the final `tar_make()`.
