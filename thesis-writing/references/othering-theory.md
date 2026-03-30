# Othering Theory — Detailed Reference

> Primary source: Falkowska, M. & Linde-Usiekniewicz, J. (2025). "Othering and Language: An Introduction." In *Naming for Othering in a Diversified Europe*. Brill. https://doi.org/10.1163/9789004743229_002
> Related: Boholm et al. (2026) on dogwhistles; Mendelsohn et al. (2020) on dehumanisation.

## Core Concept: Othering

**Othering** is the discursive process by which social actors construct and maintain an "us"–"them" distinction that positions certain groups as outside the in-group's norms, values, or humanity. Key characteristics:

- It operates through *language* — nomination strategies, referential practices, metaphors, euphemisms, and semantic framing
- It is **not limited to explicit slurs** — subtle linguistic choices (labels, grammatical constructions, semantic proximity) are equally important
- It is **aggregate and structural**, not individual — the cumulative pattern across many texts constitutes the discourse, regardless of any single author's intent
- It produces real social effects: legitimation of exclusion, marginalisation, and in extreme cases, violence

This makes it a natural fit for computational text analysis, which captures aggregate patterns rather than individual intent.

---

## Relation to Symbolic Boundaries Framework

| Framework | Focus | Unit of analysis | Method fit |
|-----------|-------|-----------------|------------|
| **Othering** (Wodak, Falkowska) | Linguistic construction of in-group/out-group | Discourse corpus | NNS, seed proximity, diachronic embedding change |
| **Symbolic boundaries** (Lamont & Molnár) | Conceptual distinctions used to categorise persons/practices | Discourse field | ALC regression, NNS clusters |
| **Boundary work** (Gieryn) | Strategic rhetorical demarcation by identifiable agents | Individual texts/actors | Qualitative, or corpus only with caution |
| **Cultural classification** (Durkheim, Zerubavel) | Socially shared cognitive distinctions | Aggregate corpus | ALC regression, co-occurrence structure |

**Othering is a process-level concept** that complements the structural-level concept of symbolic boundaries. A computational analysis of aggregate co-occurrence patterns captures the *output* of othering processes — the sedimented semantic structure — not the acts themselves.

Framing sentence for a methods section:

> "We treat the aggregate semantic proximity of group labels to legitimising vs. stigmatising vocabulary as a trace of discourse-level othering — the cumulative result of nomination and framing practices across the corpus. We do not claim to observe individual intent or strategic acts of exclusion."

---

## Linguistic Strategies of Othering

Falkowska & Linde-Usiekniewicz identify several linguistic manifestations relevant to computational operationalisation:

| Strategy | Description | Computational signal |
|----------|-------------|---------------------|
| **Nomination** | Choice of label for the out-group (e.g., *alien* vs. *migrant* vs. *refugee*) | Label-level embedding comparison |
| **Negative predication** | Attribution of negative traits or actions | Embedding neighbour valence; connotation frames |
| **Metaphorisation** | Animalistic, mechanical, or threat-based metaphors | NNS proximity to seed word dictionaries |
| **Euphemism / dogwhistling** | Coded language with hidden derogatory meaning understood by in-group | Diachronic embedding shift; in-group vs. out-group seed proximity |
| **Generalisation / stereotyping** | Treating individual as representative of the group | Frequency of bare group labels without individual modifiers |
| **Presupposition** | Embedding negative assumptions in syntactically neutral sentences | Dependency parsing + connotation frames |

---

## Us/Them Dichotomy as a Semantic Axis

For embedding-based analysis, the in-group/out-group distinction can be operationalised as a semantic axis:

```r
# In-group pole (positive community associations)
ingroup_seeds <- c("citizen", "us", "our", "community", "people", "neighbour",
                   "rights", "dignity", "belong")

# Out-group pole (othering/exclusion associations)
outgroup_seeds <- c("alien", "foreign", "threat", "burden", "invasion",
                    "crime", "infiltrate", "replace", "expel")

# Compute where a target term sits on this axis
compute_inout_position <- function(target, word_vectors, ingroup_seeds, outgroup_seeds) {
  in_vecs  <- word_vectors[intersect(ingroup_seeds,  rownames(word_vectors)), ]
  out_vecs <- word_vectors[intersect(outgroup_seeds, rownames(word_vectors)), ]

  in_centroid  <- colMeans(in_vecs)
  out_centroid <- colMeans(out_vecs)

  target_vec <- word_vectors[target, ]
  sim_in  <- as.numeric(text2vec::sim2(matrix(target_vec, nrow=1),
                                       matrix(in_centroid,  nrow=1),
                                       method = "cosine"))
  sim_out <- as.numeric(text2vec::sim2(matrix(target_vec, nrow=1),
                                       matrix(out_centroid, nrow=1),
                                       method = "cosine"))

  # Positive = more in-group; negative = more out-group
  sim_in - sim_out
}
```

---

## Dogwhistle Semantics: Implicit Meaning Tracking

Boholm et al. (2026) extend this to *dogwhistles* — terms with a publicly legitimate (out-group) meaning and a coded derogatory (in-group) meaning. Their method:

1. Collect candidate dogwhistle terms from political corpora
2. Define **out-group seeds** (explicit, neutral meaning) and **in-group seeds** (coded, derogatory meaning) using survey data or theory
3. Track the *ratio* of in-group to out-group proximity over time using diachronic embeddings
4. Rising ratio = term becoming more coded/dogwhistle-like; falling ratio = mainstream detection of coded meaning

```r
# Track dogwhistle dynamics over time
dogwhistle_score <- function(term, wv, in_seeds, out_seeds) {
  prox_in  <- proximity_to_seeds(term, in_seeds,  wv)
  prox_out <- proximity_to_seeds(term, out_seeds, wv)
  prox_in / (prox_in + prox_out)   # 0 = purely out-group; 1 = purely in-group
}
```

---

## Choosing This Framework

Use othering theory / in-group/out-group framing when:

- Your research question concerns **how a group is positioned** relative to dominant discourse norms
- You have **multiple labels** for the same referent (e.g., different ethnic or demographic terms) and want to compare their social meanings
- Your analysis spans multiple **discourse communities** (e.g., different newspaper types) where the same group may be positioned differently
- You want to connect computational findings to a sociology of exclusion or critical discourse analysis tradition

**Do not use** othering theory when:
- Your question is about the internal structure of a semantic field (not group positioning)
- You cannot distinguish in-group from out-group discourse communities in your corpus
- You claim to observe *strategic intent* rather than *aggregate semantic patterns*

---

## Sources

- Falkowska, M., & Linde-Usiekniewicz, J. (2025). Othering and language: An introduction. In *Naming for Othering in a Diversified Europe*. Brill. https://doi.org/10.1163/9789004743229_002
- Wodak, R. (2008). Introduction: Discourse studies — important concepts and terms. In R. Wodak & M. Krzyzanowski (Eds.), *Qualitative Discourse Analysis in the Social Sciences*.
- Boholm, M., et al. (2026). Tracking dogwhistles online and through time using distributional semantics. *Journal of Computational Social Science*, 9:33.
- `theoretical-framing.md` in this directory — decision guide for boundary work vs. cultural classification vs. symbolic boundaries
- `social-bias-framework.md` in `embedding-analysis/references/` — computational operationalisation of dehumanisation and othering
