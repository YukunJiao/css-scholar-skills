# Social Bias Measurement with Word Embeddings — Detailed Reference

> Primary source: Mendelsohn, Tsvetkov & Jurafsky (2020), "A Framework for the Computational Linguistic Analysis of Dehumanization." *Frontiers in Artificial Intelligence* 3:55.
> Secondary: Ohamadike et al. (2026) on race dimensions; Boholm et al. (2026) on dogwhistle semantics.

## Overview

Word embeddings capture aggregate patterns of language use, which means they absorb and can reveal social biases encoded in text corpora. This document covers how to operationalise:

1. Dehumanisation and negative group evaluation
2. Semantic proximity to stigmatising vs. legitimising vocabulary
3. Label variation (synonyms that carry different social meanings)
4. Temporal change in how groups are represented

These techniques apply whenever the research question concerns how a social group is *linguistically positioned* in discourse — not what individuals believe, but what the aggregate textual record reveals.

---

## The Dehumanisation Framework (Mendelsohn et al. 2020)

Social psychologists identify several psychological components of dehumanisation. Each maps to a measurable linguistic signal:

| Psychological component | Linguistic operationalisation | Embedding technique |
|------------------------|------------------------------|---------------------|
| **Negative evaluation** | Co-occurrence with low-valence words | Embedding neighbour valence (NRC VAD lexicon) |
| **Denial of agency** | Passive constructions; object not subject | Dependency parsing + connotation frames |
| **Moral disgust** | Proximity to contamination/disgust terms | NNS with disgust seed words |
| **Animalistic dehumanisation** | Co-occurrence with vermin/animal terms | NNS with animal seed words |
| **Mechanistic dehumanisation** | Co-occurrence with cold/object/machine terms | NNS with mechanistic seed words |

### Approach A: Embedding Neighbour Valence

Train word embeddings on the corpus. For each target group label, retrieve the *k* nearest semantic neighbours. Compute the average valence score of those neighbours using a valence lexicon (NRC VAD, ANEW, or similar).

```r
library(text2vec)
library(quanteda)

# Step 1: retrieve nearest neighbours for each group label
get_nns_valence <- function(target_word, word_vectors, valence_lexicon, k = 30) {
  if (!target_word %in% rownames(word_vectors)) return(NA)

  # Cosine similarities
  target_vec <- word_vectors[target_word, , drop = FALSE]
  sims <- text2vec::sim2(word_vectors, target_vec, method = "cosine")[, 1]

  # Top-k neighbours (excluding self)
  top_k <- sort(sims, decreasing = TRUE)[-1][1:k]
  neighbours <- names(top_k)

  # Match to valence lexicon
  matched <- valence_lexicon[valence_lexicon$word %in% neighbours, ]
  mean(matched$valence, na.rm = TRUE)
}

# Compare valence across group labels
labels    <- c("gay", "homosexual", "queer")
valences  <- sapply(labels, get_nns_valence,
                    word_vectors = wv,
                    valence_lexicon = nrc_vad,
                    k = 30)
```

### Approach B: Proximity to Seed Word Dictionaries

Define seed word lists for each bias dimension. Compute cosine similarity between the target group's embedding and the centroid of each seed list.

```r
# Define seed words for each dimension
seeds <- list(
  animalistic   = c("vermin", "beast", "animal", "creature", "swarm"),
  mechanistic   = c("robot", "machine", "object", "tool", "commodity"),
  moral_disgust = c("filth", "disease", "corrupt", "contaminate", "dirty"),
  legitimising  = c("citizen", "rights", "dignity", "community", "person")
)

# Compute centroid-to-target similarity
proximity_to_seeds <- function(target_word, seed_list, word_vectors) {
  in_vocab <- intersect(seed_list, rownames(word_vectors))
  if (length(in_vocab) == 0 || !target_word %in% rownames(word_vectors)) return(NA)

  centroid <- colMeans(word_vectors[in_vocab, ])
  text2vec::sim2(
    word_vectors[target_word, , drop = FALSE],
    matrix(centroid, nrow = 1),
    method = "cosine"
  )[1, 1]
}

# For each label × dimension combination
results <- expand.grid(label = labels, dimension = names(seeds), stringsAsFactors = FALSE)
results$similarity <- mapply(
  function(label, dim) proximity_to_seeds(label, seeds[[dim]], wv),
  results$label, results$dimension
)
```

---

## Label Variation: Synonyms with Different Social Meanings

Denotationally equivalent labels (e.g., *gay* vs. *homosexual*, *undocumented* vs. *illegal*) can occupy different semantic neighbourhoods, revealing distinct social meanings attached to each term. This is a powerful application of embedding analysis for discourse research.

### Design

1. Identify 2–5 labels for the same referent that have contested or shifting social meaning
2. Compute NNS for each label separately
3. Compute pairwise cosine distance between the labels' context vectors
4. Visualise with heatmap or biplot

```r
# Comparing embedding neighbourhoods across labels for the same referent
label_comparison <- function(labels, word_vectors, k = 20) {
  nns_lists <- lapply(labels, function(w) {
    sims <- text2vec::sim2(word_vectors, word_vectors[w, , drop = FALSE],
                           method = "cosine")[, 1]
    names(sort(sims, decreasing = TRUE)[-1][1:k])
  })
  names(nns_lists) <- labels

  # Jaccard overlap between neighbour lists
  overlap_matrix <- outer(seq_along(labels), seq_along(labels),
    Vectorize(function(i, j) {
      length(intersect(nns_lists[[i]], nns_lists[[j]])) /
      length(union(nns_lists[[i]], nns_lists[[j]]))
    }))
  dimnames(overlap_matrix) <- list(labels, labels)
  overlap_matrix
}
```

---

## Diachronic Bias Tracking

Track how the valence / proximity profile of a group label changes over time. Uses year-specific embeddings initialised from a jointly-trained base model (see `SKILL.md § Training GloVe`).

```r
# For each time period, compute bias metrics
years <- sort(unique(docvars(tokens, "year")))

bias_over_time <- lapply(years, function(yr) {
  yr_tokens  <- tokens_subset(tokens, year == yr)
  yr_fcm     <- quanteda::fcm(yr_tokens, context = "window", window = 6)

  # Initialise GloVe from full-corpus vectors
  glove_yr   <- text2vec::GlobalVectors$new(rank = 100, x_max = 10)
  glove_yr$fit_transform(yr_fcm, n_iter = 5,
                         init = list(w_i = wv_main, w_j = wv_context))

  wv_yr <- glove_yr$get_word_vectors()

  list(
    year     = yr,
    valence  = get_nns_valence("target_group", wv_yr, nrc_vad),
    animalistic = proximity_to_seeds("target_group", seeds$animalistic, wv_yr),
    legitimising = proximity_to_seeds("target_group", seeds$legitimising, wv_yr)
  )
})

bias_df <- dplyr::bind_rows(bias_over_time)
```

---

## Othering and In-Group/Out-Group Framing

A related application: tracking how *in-group* vs. *out-group* semantic associations change over time, connecting to othering theory (Falkowska & Linde-Usiekniewicz 2025). The key methodological insight from Boholm et al. (2026) is that embedding-based semantic proximity can be calibrated against human annotation data to distinguish:

- **Explicit (out-group) meaning**: the publicly legible, mainstream semantic association
- **Implicit (in-group) meaning**: the coded or stigmatising semantic association, visible only to some communities

### Practical application

Define "explicit" and "implicit" seed word sets based on theoretical reasoning or survey data:

```r
seeds_ingroup  <- c("crime", "threat", "burden", "invasion")   # implicit derogatory meaning
seeds_outgroup <- c("community", "population", "resident", "arrival")  # neutral descriptive meaning

# Track how each target term shifts between in-group and out-group semantic space
for (term in target_terms) {
  explicit_prox <- proximity_to_seeds(term, seeds_outgroup, wv)
  implicit_prox <- proximity_to_seeds(term, seeds_ingroup, wv)
  cat(term, "→ explicit:", round(explicit_prox, 3),
             "| implicit:", round(implicit_prox, 3), "\n")
}
```

---

## Interpreting Results: What Embedding Bias Evidence Can and Cannot Show

| Valid inference | Invalid inference |
|----------------|-------------------|
| "Term X is semantically closer to [dimension] in corpus A than corpus B" | "Speakers of corpus A consciously hold [attitude]" |
| "The semantic neighbourhood of X shifted toward [cluster] between period T1 and T2" | "Authors intentionally promoted [ideology]" |
| "Label X is more proximate to derogatory vocabulary than label Y in this corpus" | "Label X *is* derogatory" (without human validation) |
| "The aggregate pattern of co-occurrence reflects [social meaning] encoded in discourse" | Individual-level beliefs or intentions |

**Always validate**: compute neighbour valence or seed proximity for a *neutral control word* in the same semantic domain. If the control shows the same pattern, the result may reflect domain characteristics rather than group-specific bias.

---

## Sources

- Mendelsohn, J., Tsvetkov, Y., & Jurafsky, D. (2020). A framework for the computational linguistic analysis of dehumanization. *Frontiers in Artificial Intelligence*, 3:55. https://doi.org/10.3389/frai.2020.00055
- Ohamadike, N., Durrheim, K., & Primus, M. (2026). Anchoring race: improving the construction of race dimensions in word embeddings. *Journal of Computational Social Science*, 9:20.
- Boholm, M., et al. (2026). Tracking dogwhistles online and through time using distributional semantics. *Journal of Computational Social Science*, 9:33.
- Falkowska, M., & Linde-Usiekniewicz, J. (2025). Othering and language: An introduction. In *Naming for Othering in a Diversified Europe*. Brill.
