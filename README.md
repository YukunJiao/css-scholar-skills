# Computational Social Science SKILL Library

> A curated set of reusable knowledge cards for AI-assisted computational social science research — covering R pipeline engineering, word embedding analysis, academic manuscript writing, and research ideation.

Structured after [Orchestra Research AI-Research-SKILLs](https://github.com/Orchestra-Research/AI-Research-SKILLs).

Each SKILL is a self-contained document that an AI assistant (or human collaborator) can read at the start of a session to immediately perform a specialised task — without needing the full project history re-explained.

---

## Skills

| # | Skill | Summary |
|---|-------|---------|
| 01 | [r-targets-pipeline](r-targets-pipeline/SKILL.md) | Build reproducible R pipelines with `{targets}` for large text corpora — chunked memory management, package configuration, incremental execution |
| 02 | [embedding-analysis](embedding-analysis/SKILL.md) | ALC word embedding regression with `conText` — GloVe training, anchor word selection, cross-group/diachronic semantic change, results interpretation |
| 03 | [thesis-writing](thesis-writing/SKILL.md) | Academic manuscript workflow with Quarto + targets — document structure, theoretical framing for computational discourse analysis, citation management |
| 04 | [research-ideation](research-ideation/SKILL.md) | Research question design and theory-method alignment for computational discourse analysis — operationalisation, contribution framing, extension strategies |

---

## Quick Reference

| Problem | Where to look |
|---------|--------------|
| `tar_make()` stuck / OOM | [r-targets-pipeline § Memory Management](r-targets-pipeline/SKILL.md#memory-management) |
| Worker: `could not find function X` | [r-targets-pipeline § Package Configuration](r-targets-pipeline/SKILL.md#package-configuration) |
| Selecting and validating anchor words | [embedding-analysis § Anchor Word Selection](embedding-analysis/SKILL.md#anchor-word-selection) |
| Choosing between boundary work and cultural classification | [thesis-writing § Theoretical Framing](thesis-writing/SKILL.md#theoretical-framing-key-decision) |
| Theory-method alignment check | [research-ideation § Theory-Method Alignment](research-ideation/SKILL.md#theory-method-alignment) |
| Framing contributions for different journals | [research-ideation § Contribution Framing](research-ideation/SKILL.md#contribution-framing) |

---

## Who This Is For

- **Computational sociologists** building reproducible pipelines for large digitised corpora
- **Digital historians** applying word embedding analysis to historical text
- **CSS researchers** writing reproducible manuscripts with Quarto and R
- **AI assistants** needing structured domain knowledge to assist with the above tasks efficiently

---

## File Structure

```
skills/
├── README.md
├── CONTRIBUTING.md
├── LICENSE
├── .gitignore
├── .github/
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.md
│       └── new_skill.md
│
├── r-targets-pipeline/
│   ├── SKILL.md
│   └── references/
│       └── memory-management.md      ← OOM diagnosis, gc() patterns, Arrow lazy loading
│
├── embedding-analysis/
│   ├── SKILL.md
│   └── references/
│       └── anchor-selection.md       ← frequency checks, OCR validation, exclusion rationale
│
├── thesis-writing/
│   ├── SKILL.md
│   └── references/
│       └── theoretical-framing.md    ← boundary work vs. cultural classification decision guide
│
└── research-ideation/
    ├── SKILL.md
    └── references/
        └── extension-directions.md   ← temporal, comparative, vocabulary, cross-validation
```

---

## How to Use with Claude Code

Load a SKILL at the start of a session:

```
Please read skills/r-targets-pipeline/SKILL.md and its references, then help me debug my tar_make() run.
```

Or reference a specific section:

```
My worker is crashing with OOM — check skills/r-targets-pipeline/references/memory-management.md and suggest a fix.
```

SKILLs are designed to be **loadable independently**: a new Claude session that reads only one SKILL has everything it needs to perform that task, without requiring the full project history.

---

## Dependencies

### R packages
```r
# Pipeline
targets, tarchetypes, qs, arrow

# NLP
quanteda, quanteda.textstats, text2vec, conText

# Data wrangling
dplyr, purrr, stringr, lubridate, tidytext

# Visualisation
ggplot2, ggrepel, patchwork

# Manuscript
quarto, knitr
```

---

## Related Resources

| Resource | Link |
|----------|------|
| conText R package | https://github.com/prodriguezsosa/conText |
| Rodriguez et al. (2023) — conText APSR paper | https://doi.org/10.1017/S0003055422001228 |
| {targets} R package book | https://books.ropensci.org/targets/ |
| AmericanStories dataset | https://huggingface.co/datasets/dell-research-lab/AmericanStories |
| Orchestra Research AI-Research-SKILLs | https://github.com/Orchestra-Research/AI-Research-SKILLs |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) — contributions, corrections, and new SKILLs are welcome.

## License

[MIT](LICENSE)
