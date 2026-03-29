# Extension Directions — Detailed Reference

## Temporal Extension

### Design principles

When extending a diachronic analysis to new time periods:

1. **Re-train GloVe jointly** on the extended corpus — do not concatenate embeddings from separate models
2. **Add the new period as a factor level**, not a separate analysis — maintain comparability through the shared embedding space
3. **Hypothesise directionality** before running: what do you expect to happen in the extended period and why?
4. **Check frequency thresholds** for all anchor words in the new period — some words that were common in the original period may become rare (or vice versa)

### Identifying period boundaries

Rather than using arbitrary calendar decades, consider:
- **Institutional markers**: founding of professional associations, new legislation, major publications
- **Technological markers**: new media technologies, transportation changes
- **Political markers**: elections, wars, constitutional moments
- **Conceptual markers**: coinage of key terms, publication of foundational texts

```r
# Test whether a specific year is a structural break
# Fit ALC regression with year as continuous + interaction terms
# Large coefficient change around a year = candidate breakpoint
model_linear <- conText(formula = . ~ year, ...)
model_break  <- conText(formula = . ~ year + I(year > 1840) + year:I(year > 1840), ...)
```

---

## Comparative Corpus Design

### Within-language comparison

Compare discourse communities that produce text in the same language but differ along theoretically relevant dimensions:

| Dimension | Examples |
|-----------|---------|
| Geography | Regional newspapers, state legislatures, city vs. rural |
| Institutional type | Commercial, political, religious, civic publications |
| Class / audience | Elite vs. popular press |
| Political alignment | Partisan organs across the spectrum |

**Design requirement**: the comparison makes sense only if the corpora are comparable in text type and time period. Mixing genres (newspaper articles vs. scientific papers vs. sermons) conflates genre effects with community effects.

### Cross-language comparison

Comparing semantic structure across languages requires additional care:

1. Train separate GloVe models per language (joint training is not applicable)
2. Align the spaces using Procrustes rotation or anchor-based alignment
3. Alternatively: use multilingual pre-trained embeddings as a common space
4. Interpret differences cautiously — linguistic structure (not just social meaning) affects neighbourhoods

---

## Vocabulary Extension

### When to add words

Add words to the target vocabulary when:
- A new theoretical construct needs operationalisation
- You want to test whether a finding generalises to adjacent vocabulary
- Reviewers request robustness checks with alternative operationalisations

### When NOT to add words

Do not add words:
- That fail the frequency threshold (< 100 occurrences per group)
- That are near-synonyms of existing anchors without theoretical justification (redundancy inflates apparent confidence)
- That are confounded by polysemy or OCR noise (see `anchor-selection.md`)

### Robustness via synonym substitution

```r
# Test: does the finding hold if we replace "science" with "natural philosophy"?
# If results are similar → robust operationalisation
# If results differ → the two words capture different constructs; report both
words_to_compare <- c("science", "natural philosophy", "philosophy")
results_list <- purrr::map(words_to_compare, function(w) {
  ctx <- conText::get_context(tokens, w, window = 6)
  conText::conText(formula = . ~ decade + press_type, data = ctx, ...)
})
```

---

## Cross-Validation with Other Methods

### With topic models (STM)

Structural topic models (STM) provide an independent measure of thematic variation by covariate. Compare:
- Do topics align with semantic clusters from NNS?
- Do topic prevalence differences by group match coefficient directions from conText?
- Convergent findings → stronger evidence; divergent → investigate why

```r
library(stm)
stm_model <- stm(dfm, K = 20,
                 prevalence = ~ decade + press_type,
                 data = quanteda::docvars(tokens))

# Compare topic words with NNS neighbours
# Topic 5 top words: "experiment", "discovery", "observation"
# → matches NNS cluster for "science" → convergent validity
```

### With supervised classifier

Train a classifier to label documents as science-dominant vs. religion-dominant using a small labelled set. Compare:
- Does classifier output correlate with net science orientation index?
- Does classifier performance vary by press type in the direction predicted by conText?

### With manual annotation

For key claims, validate a random sample (n = 200–500 context windows) with human coders:
- Do human coders assign the same semantic register as the NNS suggests?
- Inter-rater reliability as a quality check on the embedding operationalisation
