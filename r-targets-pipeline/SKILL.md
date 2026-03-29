---
name: r-targets-pipeline
description: Build and run reproducible R research pipelines with {targets} for large text corpora — covering memory management, package configuration, and incremental execution.
version: 1.0.0
author: css-skills
tags: [R, targets, pipeline, memory-management, reproducibility, large-corpus, arrow, quanteda]
dependencies: [targets, tarchetypes, arrow, qs]
---

# R `{targets}` Pipeline for Large-Corpus Text Analysis

## When to Use

| Situation | Action |
|-----------|--------|
| Multi-step analysis with >5 interdependent stages | Use `{targets}` — tracks dependencies, skips unchanged targets |
| Corpus too large to load into RAM at once | Chunked processing — one file/batch at a time with `gc()` |
| Need to re-run only changed parts of pipeline | `tar_make()` is incremental by default |
| Debugging a single step | `tar_make(names = "target_name")` |
| Worker stuck / no progress for >10 min | OOM — see Memory Management section |
| Checking what will re-run | `tar_outdated()` or `tar_manifest()` |

**Do NOT use** a single script when: data > 1 GB, pipeline has >5 steps, or results must be reproducible across machines.

---

## Quick Start

### Minimal `_targets.R`

```r
library(targets)
library(tarchetypes)

tar_option_set(
  packages = c(
    # List EVERY package used inside any target function.
    # Workers are callr subprocesses and do NOT inherit the main session's
    # loaded packages — missing entries cause "could not find function" errors.
    "arrow", "dplyr", "purrr", "stringr",
    "quanteda", "quanteda.textstats",
    "ggplot2",
    "qs"
  ),
  format = "qs"   # 3–5× faster serialisation than default "rds"
)

tar_source()      # load all R/*.R function files into worker environments

list(
  tar_target(raw_data,   load_metadata(DATA_PATHS)),
  tar_target(data,       engineer_covariates(raw_data)),
  tar_target(tokens,     build_tokens(TEXT_FILES, data)),
  tar_target(embeddings, train_embeddings(tokens)),
  tar_target(results,    run_analysis(tokens, embeddings, data))
)
```

### Run

```r
targets::tar_make()                    # run all outdated targets
targets::tar_make(names = "results")   # run one specific target
targets::tar_read("embeddings")        # load a completed target object
targets::tar_outdated()                # list what will re-run
targets::tar_manifest()                # list all targets and their commands
targets::tar_visnetwork()              # visualise dependency graph
```

---

## Memory Management

### The Problem

Loading a multi-file corpus all at once causes memory exhaustion:
- macOS: memory compressor fills (check with `vm_stat`), worker enters uninterruptible sleep (`UNs`)
- Linux: OOM killer terminates the R worker process
- Symptom: `tar_make()` appears frozen with no output for >10 minutes

### Diagnosis

```bash
# macOS
vm_stat | grep -E "Pages free|Pages occupied by compressor"
# Danger: compressor > 1,500,000 pages (~6 GB)

ps aux | grep R   # look for UNs state in STAT column
```

### The Fix: Chunked Processing

Process one file/partition at a time and release memory between iterations:

```r
# BAD — loads everything at once
build_tokens_bad <- function(files, metadata) {
  all_text <- purrr::map_dfr(files, arrow::read_parquet)   # entire corpus in RAM
  quanteda::tokens(quanteda::corpus(all_text), ...)
}

# GOOD — one file at a time, release between iterations
build_tokens <- function(files, metadata) {
  sw <- quanteda::stopwords("en")   # compute stopword list once, outside loop

  toks_list <- purrr::map(files, function(path) {
    # 1. Read one file
    chunk <- arrow::open_dataset(path) |>
      dplyr::select(doc_id, text) |>
      dplyr::collect() |>
      dplyr::left_join(metadata, by = "doc_id")

    # 2. Build corpus and tokenise
    corp <- quanteda::corpus(chunk, text_field = "text", docid_field = "doc_id")
    rm(chunk); gc()   # ← release text immediately after corpus is built

    toks <- quanteda::tokens(corp,
        remove_punct = TRUE, remove_symbols = TRUE,
        remove_numbers = TRUE, remove_url = TRUE) |>
      quanteda::tokens_tolower() |>
      quanteda::tokens_remove(pattern = sw, padding = TRUE)
    rm(corp); gc()    # ← release corpus immediately after tokenising
    toks
  })

  do.call(c, toks_list)   # combine into single tokens object
}
```

**Peak RAM: ~1 corpus file at a time** instead of the full corpus.

Apply the same `rm() + gc()` pattern to any function that reads article text in a loop.

### Vocabulary Trimming Before FCM

Building a feature co-occurrence matrix (FCM) on a raw 500K-word vocabulary produces a huge sparse matrix. Trim first:

```r
build_fcm <- function(tokens, window = 6, min_count = 10) {
  # Trim vocabulary: raw ~500K → trimmed ~80–120K
  # FCM memory: several GB → a few hundred MB
  vocab      <- quanteda::dfm(tokens) |>
    quanteda::dfm_trim(min_termfreq = min_count) |>
    quanteda::featnames()
  toks_trim  <- quanteda::tokens_select(tokens, pattern = vocab, padding = TRUE)
  quanteda::fcm(toks_trim, context = "window", window = window,
                count = "frequency", tri = FALSE)
}
```

### Arrow Lazy Evaluation

Avoid loading columns you do not need:

```r
# BAD: materialises full parquet file
df <- arrow::read_parquet(path)

# GOOD: pushes column selection and filtering into Arrow before collect()
df <- arrow::open_dataset(path) |>
  dplyr::select(doc_id, text) |>
  dplyr::filter(year >= 1800) |>
  dplyr::collect()

# Computing text length without loading text into R
lengths <- arrow::open_dataset(path) |>
  dplyr::mutate(text_len = nchar(text)) |>
  dplyr::select(doc_id, text_len) |>
  dplyr::collect()
```

---

## Package Configuration

### The `tar_option_set(packages = ...)` Rule

Worker processes are callr subprocesses — they start fresh and do **not** inherit packages loaded in the main R session. Every package called inside any target function must be declared:

```r
# WRONG — causes "could not find function 'geom_text_repel'" in worker
library(ggrepel)   # loaded in main session only
tar_target(plot, make_plot(data))

# CORRECT
tar_option_set(packages = c("ggplot2", "ggrepel", "patchwork"))
```

**Rule**: if a function from package `foo` is called anywhere inside a target (directly or via `foo::bar()`), add `"foo"` to `packages`.

### `format = "qs"`

Always set `format = "qs"` in `tar_option_set`. Requires the `qs` package. Serialises large objects (tokens, sparse matrices, data frames) 3–5× faster than the default `"rds"` format.

```r
tar_option_set(
  packages = c(...),
  format   = "qs"
)
```

---

## `tar_map` for Repeated Analysis Across Groups

When the same set of targets must run for multiple inputs (e.g., a list of keywords, demographic groups, or time windows), use `tar_map` to avoid repetition:

```r
KEYWORDS <- c("democracy", "liberty", "constitution")

tar_map(
  values = list(keyword = KEYWORDS),
  tar_target(context_windows, get_context(tokens, keyword)),
  tar_target(nns,             run_nns(context_windows, embeddings)),
  tar_target(model,           run_regression(context_windows, embeddings, transform, data))
)
```

This generates `context_windows_democracy`, `nns_democracy`, `model_democracy`, etc. — 9 targets from 3 lines.

Combine results downstream:

```r
tar_combine(
  nns_all,
  tar_select_targets(nns_.*),
  command = dplyr::bind_rows(!!!.x)
)
```

---

## Quarto Manuscript Integration

```r
# In _targets.R — add manuscript as the final target
tarchetypes::tar_quarto(
  manuscript,
  path  = "manuscript/manuscript.qmd",
  quiet = FALSE
)
```

```yaml
# manuscript/_targets.yaml — redirect tar_read() to project-root store
store: ../_targets
```

```r
# In manuscript.qmd setup chunk — declare upstream dependencies
library(targets)
results <- tar_read(results)     # tarchetypes detects these as dependencies
```

---

## Common Issues

| Error | Cause | Fix |
|-------|-------|-----|
| Worker frozen, no output for >10 min | OOM / memory compression | Switch to chunked processing; check `vm_stat` |
| `could not find function X` in worker | Package not in `tar_option_set(packages)` | Add missing package |
| `object 'foo' not found` in worker | `tar_source()` missing | Add `tar_source()` before `list(...)` in `_targets.R` |
| Target re-runs every time | Non-deterministic hash (timestamp, random seed) | Set `set.seed()`; or `tar_cue(mode = "never")` for stable intermediates |
| `qs` serialisation error | `qs` not installed | `install.packages("qs")` |
| Slow `fcm()` / OOM | Full vocabulary, no trim | Apply `dfm_trim(min_termfreq = N)` before `fcm()` |
