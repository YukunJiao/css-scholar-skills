---
name: thesis-writing
description: Write and render reproducible academic manuscripts in Quarto integrated with a {targets} pipeline; covers document structure, theoretical framing for computational discourse analysis, and citation management.
version: 1.0.0
author: css-skills
tags: [Quarto, targets, manuscript, academic-writing, theoretical-framing, computational-sociology, citation, reproducible-research]
dependencies: [quarto, tarchetypes, knitr]
---

# Academic Manuscript Writing — Quarto + `{targets}`

## Quarto + targets Integration

### Project layout

```
project/
├── _targets.R
├── R/
├── data/
├── figures/                    ← saved PNGs from visualise targets
└── manuscript/
    ├── _targets.yaml           ← store pointer (required)
    ├── manuscript.qmd
    └── references.bib
```

### `_targets.yaml` (inside `manuscript/`)

```yaml
store: ../_targets
```

This tells `tarchetypes::tar_quarto()` — and all `tar_read()` calls inside the Qmd — to look at the project-root `_targets/` store, not a local one.

### Manuscript target in `_targets.R`

```r
tarchetypes::tar_quarto(
  manuscript,
  path  = "manuscript/manuscript.qmd",
  quiet = FALSE
)
```

Place this as the **last** target in `list(...)`. It will re-render only when upstream targets change.

### Declare dependencies inside the Qmd

`tarchetypes` statically detects `tar_read()` calls to build the dependency graph:

```r
# setup chunk (echo: false)
library(targets)
library(dplyr)
library(ggplot2)

data     <- tar_read(data)
results  <- tar_read(results)
nns_all  <- tar_read(nns_all)
coef_all <- tar_read(coef_all)
```

### Inline figures from saved PNGs

```r
#| label: fig-results
#| fig-cap: "Main results figure. Caption here."
#| out-width: "100%"
knitr::include_graphics("../figures/results.png")
```

The PNG is produced by a dedicated visualisation target that saves to `figures/`. The Qmd reads the file, not the plot object, to avoid holding large ggplot objects in the targets store.

---

## Document Structure for Computational Discourse Analysis

```
Abstract           — key findings first, method second
Introduction       — gap → site → three contributions (bullet list)
Conceptual Framework
  § [Primary theory]               ← what your evidence directly measures
  § [Original contribution]        ← your extension / two-dimensional model
  § Discourse-Level Operationalisation   ← how method connects to theory
Data and Methods
  § Corpus Description
  § [Unit classification, e.g. press type / speaker type]
  § Embedding Method (GloVe + ALC regression)
  § Target Vocabulary (N words, M groups, exclusion rationale)
Results
  § Semantic Neighbourhoods (NNS figures)
  § Covariate Effects (coefficient heatmap)
  § [Optional: article-level aggregate index, trend figure]
Discussion
  § [Finding 1 in relation to theory]
  § [Finding 2: conceptual framework illustrated]
Conclusion
References
```

---

## Theoretical Framing: Key Decision

### Matching evidence type to theory

| Evidence type | Best-fit theory | Why |
|---------------|----------------|-----|
| Aggregate semantic position (NNS, ALC coefficients) | **Cultural classification** (Durkheim, Zerubavel, Lamont & Molnár) | ALC captures the classification structure inscribed in collective discourse — not individual acts |
| Specific rhetorical strategies in individual texts | **Boundary work** (Gieryn) | Boundary work requires active agents with identifiable interests |
| Shared conceptual distinctions across a community | **Symbolic boundaries** (Lamont & Molnár 2002) | Broad enough to fit discourse-level evidence |
| Topic distributions with covariate effects | **Framing theory** | Frames are topic-covariate associations |
| Word frequency change over time | **Conceptual history / Begriffsgeschichte** | Change in word use = change in concept |

### Common mismatch to avoid

> "The results show that newspapers engaged in *boundary work* by positioning X near Y."

**Problem**: boundary work requires intentional demarcation by identifiable agents. An aggregate NNS pattern across millions of articles cannot be attributed to a strategic rhetorical act.

**Fix**: reframe as *cultural classification* or *symbolic boundary inscription* — the aggregate co-occurrence structure reflects how the discourse community collectively positioned these concepts, regardless of individual intent.

### Theory hierarchy for discourse analysis

```
Most appropriate for aggregate semantic evidence:
  Durkheim (1912) → classification as social form
  Zerubavel (1991) → "mental boundaries" — shared cognitive distinctions
  Lamont & Molnár (2002) → symbolic boundaries in discourse

Can be used as background/motivation:
  Gieryn (1983, 1999) → boundary work — explains why agents produced the discourse
  Bourdieu → field theory — explains why different communities differ
```

---

## Writing the Conceptual Framework Section

### Structure

```
Paragraph 1: State the two (or more) theoretical traditions you draw on.
Paragraph 2: Explain why your evidence fits the primary tradition.
Paragraph 3: Explain how the secondary tradition provides context or motivation.
Paragraph 4 (if original contribution): State the novel framework element.
Paragraph 5: Operationalise — connect theory to method explicitly.
```

### Operationalisation paragraph template

> In this paper, I operationalise [Dimension 1] at the level of public discourse through [Method A] and [Dimension 2] through [Method B]. The unit of analysis is [discourse unit], not individual cognition. [Method A] captures [what the method literally measures], which I treat as a proxy for [theoretical construct] under the assumption that [linking assumption].

---

## Citation Management

### Working with Zotero + Quarto

1. Maintain all references in Zotero
2. Export collection as BibTeX → `manuscript/references.bib`
3. Reference in Qmd front matter: `bibliography: references.bib`
4. Use Zotero's "Better BibTeX" plugin for stable, human-readable keys

### Key format: `authorYEARkeyword`

```
lamont2002study         Lamont & Molnár (2002) ARS
gieryn1983boundary      Gieryn (1983) ASR
zerubavel1991fine       Zerubavel (1991) The Fine Line
rodriguez2023context    Rodriguez et al. (2023) APSR — conText package
rodriguez2022embeddings Rodriguez & Spirling (2022) JoPol
hamilton2016diachronic  Hamilton et al. (2016) — diachronic word vectors
```

### Global search/replace for key changes

If a citation key changes (e.g., after Zotero re-export):

```bash
# In terminal, from manuscript/ directory
grep -r "old_key" .
sed -i '' 's/@old_key/@new_key/g' manuscript.qmd
```

Or use RStudio's Find in Files with regex.

---

## Common Writing Pitfalls

| Pitfall | Fix |
|---------|-----|
| Claiming aggregate NNS results show "boundary work" (active demarcation) | Reframe as symbolic boundary inscription or cultural classification |
| Treating cosine similarity as cultural distance | Note: proximity = discourse co-occurrence, not equivalence in the social world |
| Writing Results before pipeline completes | Write interpretive scaffold with hedged language; verify specific claims against actual figures |
| Year as continuous covariate when change is expected to be non-linear | Use `factor(decade)` with earliest as reference |
| Manuscript doesn't re-render when upstream targets change | Ensure `_targets.yaml` points to correct store; use `tar_read()` not `readRDS()` |
| Citations in manuscript not in bib | Quarto silently renders missing cites as `[?]`; run `quarto render` and check warnings |
