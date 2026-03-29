# Anchor Word Selection — Detailed Reference

## Theoretical Grounding

Anchor words (target words) operationalise the theoretical constructs you are studying. Poor anchor selection is the most common source of invalid conclusions in embedding analysis. The selection process has three stages:

1. **Theoretical** — derive candidate words from the conceptual framework
2. **Empirical** — verify frequency and coverage in the corpus
3. **Validation** — inspect context windows for confounds; verify NNS structure matches theory

---

## Stage 1: Theory → Candidates

For each theoretical construct, brainstorm 5–10 candidate words. Then reduce to the 2–4 most representative based on stages 2 and 3.

### Example mapping (knowledge/epistemic boundary studies)

| Construct | Candidate words |
|-----------|----------------|
| Scientific knowledge | science, philosophy, reason, knowledge, learning |
| Empirical practice | experiment, observation, evidence, discovery |
| Applied knowledge | invention, technology, medicine, chemistry |
| Religious knowledge | religion, faith, scripture, revelation |
| Condemned/illegitimate belief | superstition, fanaticism, enthusiasm |

### Example mapping (political discourse studies)

| Construct | Candidate words |
|-----------|----------------|
| Freedom | liberty, freedom, rights, independence |
| Authority | order, authority, government, power |
| Civic identity | citizen, nation, republic, democracy |
| Threat | danger, enemy, tyranny, oppression |

---

## Stage 2: Frequency Validation

```r
# Step 1: Build DFM grouped by your covariate of interest
dfm_by_group <- quanteda::dfm(tokens) |>
  quanteda::dfm_group(groups = quanteda::docvars(tokens, "time_period"))

# Step 2: Extract frequency table for candidates
candidate_words <- c("science", "philosophy", "religion", "superstition")
freq_matrix <- quanteda::featfreq(dfm_by_group)[candidate_words, ]

# Step 3: Flag words below threshold
min_freq <- 100   # minimum instances per group for reliable ALC
below_threshold <- freq_matrix[apply(freq_matrix, 1, min) < min_freq, ]
cat("Words below frequency threshold:\n")
print(below_threshold)
```

If a word fails the frequency threshold in any group, either:
- Exclude it from that group's analysis
- Merge adjacent groups (e.g., two decades into one)
- Replace with a higher-frequency synonym

---

## Stage 3: Context Window Validation

```r
library(quanteda)

# Check for polysemy / proper noun overlap
kwic(tokens, "providence", window = 5) |> head(40)
# ↑ if output shows city names, newspaper titles, etc. → exclude

# Check for OCR garbling
kwic(tokens, "chymistry", window = 3) |> head(20)
# ↑ archaic spelling — decide whether to fold into "chemistry" via tokens_replace()

# Check context coherence for a theoretical group
# All words in a group should have overlapping NNS
group_words <- c("science", "philosophy", "reason")
for (w in group_words) {
  ctx <- conText::get_context(tokens, w, window = 6)
  nns_result <- conText::nns(ctx, pre_trained = embeddings,
                              transform = TRUE, transform_matrix = transform,
                              candidates = dfm, N = 10, bootstrap = FALSE)
  cat(w, "→", paste(nns_result$feature[1:5], collapse = ", "), "\n")
}
# Expect substantial overlap in top neighbours within a theoretical group
```

---

## Handling Archaic / OCR Variants

Historical corpora often contain spelling variants. Use `tokens_replace()` to normalise before analysis:

```r
tokens <- quanteda::tokens_replace(
  tokens,
  pattern     = c("chymistry", "chymist", "chymical"),
  replacement = c("chemistry",  "chemist",  "chemical"),
  valuetype   = "fixed"
)
```

Apply before building the FCM so the unified form gets its full frequency weight.

---

## Common Pitfalls

| Pitfall | Consequence | Prevention |
|---------|------------|------------|
| Including agent nouns (e.g., "scientist", "minister") | Captures social role, not epistemic concept | Prefer abstract nouns; inspect NNS |
| City / person name homonyms | NNS polluted by geographic/biographical context | Check KWIC; exclude or disambiguate |
| Synonym redundancy within a group | Artificially similar NNS; no additional information | Keep only 1–2 representatives per construct |
| Including a word very rare in one group | ALC estimate unreliable; wide CIs | Frequency gate: min 100 per group |
| Words whose meaning IS the research outcome | Circular reasoning | Ensure anchors are proxies, not direct measures |
