# Memory Management Reference

## Diagnosis

### macOS

```bash
vm_stat | grep -E "Pages free|Pages occupied by compressor"
```

| Reading | Meaning |
|---------|---------|
| `Pages free` < 50,000 (~200 MB) | System under pressure |
| `Pages occupied by compressor` > 1,500,000 (~6 GB compressed) | Active memory compression — I/O-bound slowdown |

```bash
# Check if R worker is sleeping uninterruptibly
ps aux | grep -E "\bR\b" | grep -v grep
# UNs in STAT column = uninterruptible sleep = waiting for paged-out memory
```

### Linux

```bash
free -h
cat /proc/meminfo | grep -E "MemAvailable|SwapFree"
# If MemAvailable < 500 MB, OOM risk is high
```

---

## Memory Budget Estimation

Estimate peak RAM before running:

| Object | Rough size formula |
|--------|--------------------|
| Raw text parquet (1 file) | ~0.5–2 GB depending on corpus size |
| quanteda corpus (text in RAM) | ≈ raw text × 1.5 |
| quanteda tokens (full vocab) | ≈ corpus × 0.5 |
| DFM (untrimmed) | up to several GB for large vocab |
| FCM (trimmed vocab, window=6) | ~200–800 MB |
| GloVe word vectors (100 dims, 100K words) | ~80 MB |

**Rule of thumb**: if `n_files × file_size_GB > available_RAM × 0.5`, use chunked processing.

---

## `rm()` + `gc()` Placement Rules

```r
# Pattern: release large intermediates as soon as they are no longer needed
process_one_file <- function(path, metadata) {
  # Step 1: load text
  chunk <- arrow::open_dataset(path) |> dplyr::collect()

  # Step 2: build corpus (corpus duplicates text in memory)
  corp <- quanteda::corpus(chunk, text_field = "text")
  rm(chunk); gc()           # ← release raw text immediately

  # Step 3: tokenise (tokens object replaces corpus)
  toks <- quanteda::tokens(corp, ...)
  rm(corp); gc()            # ← release corpus immediately

  toks
}
```

**Do NOT**:
- Call `gc()` inside tight inner loops (overhead exceeds benefit)
- Call `gc()` between `|>` pipe steps (objects not yet materialised)
- Rely on `gc()` alone without `rm()` — R's garbage collector needs zero references

---

## Arrow Lazy Column Selection

Use `open_dataset()` + `select()` + `collect()` instead of `read_parquet()` when only a subset of columns is needed. Arrow pushes the projection into the Parquet file reader — only selected columns are decompressed and loaded into R.

```r
# Loads only doc_id and text (~500 MB instead of ~1.5 GB for a full file)
df <- arrow::open_dataset(path) |>
  dplyr::select(doc_id, text) |>
  dplyr::collect()

# Compute a derived column (e.g., text length) without loading text at all
meta <- arrow::open_dataset(path) |>
  dplyr::mutate(text_len = nchar(text)) |>
  dplyr::select(doc_id, text_len) |>
  dplyr::collect()
# text column never enters R memory
```

---

## Vocabulary Trimming Strategy

Building an FCM on a raw vocabulary of 300K–600K tokens from a large corpus produces a matrix that is prohibitively large. Trim the vocabulary using term frequency before building the FCM:

```r
build_fcm_trimmed <- function(tokens, min_count = 10, window = 6) {
  # Build a DFM first — cheap, used only to count frequencies
  vocab <- quanteda::dfm(tokens) |>
    quanteda::dfm_trim(min_termfreq = min_count) |>
    quanteda::featnames()

  # Apply trimmed vocab to tokens (padding preserves window positions)
  toks_trim <- quanteda::tokens_select(tokens, pattern = vocab, padding = TRUE)

  # FCM on trimmed tokens: manageable memory
  quanteda::fcm(toks_trim, context = "window", window = window,
                count = "frequency", tri = FALSE)
}
```

| Vocabulary size | FCM memory (window=6) | Typical trim threshold |
|----------------|----------------------|----------------------|
| 500K (raw) | 5–15 GB | — |
| 100K (min_count=10) | 300–800 MB | Standard |
| 50K (min_count=50) | 100–300 MB | Sparse corpora |
