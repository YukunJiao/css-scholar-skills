# Symbolic Boundary Measurement — Region vs. Direction

> Derived from thesis research design on incel discourse; generalises to any two-mechanism boundary framework.

## Core Distinction

Two types of symbolic boundary mechanism require different operationalisation strategies:

| Boundary type | Semantic structure | Primary metric | Method |
|--------------|-------------------|----------------|--------|
| **Moral exclusion** (e.g. dehumanisation) | Cluster / region | CMDist to centroid | Concept centroid + cosine proximity |
| **Cultural exclusion** (e.g. essentialism) | Axis / direction | Projection score | Contrastive direction + dot product |

The choice of metric follows from the geometric structure of the concept in embedding space, not from theoretical preference alone.

---

## Why Dehumanisation Is a Region, Not a Direction

Attempting to construct a `human ←→ non-human` axis fails because:

- The sub-components of dehumanisation (animalistic, objectifying, morally degrading) do **not** align along a single direction — they scatter across multiple semantic dimensions
- The positive pole ("human") is difficult to define as a clean contrastive anchor

**Consequence**: A direction built from `human – animal`, `person – object`, etc. produces noisy, low-PairDir offset vectors (see `anchor-quality-metrics.md`). The dimension will not replicate.

**Fix**: Treat dehumanisation as a **semantic region** — a cluster of related meanings — and measure proximity to its centroid rather than position along an axis.

---

## Operationalisation: Dehumanisation → Centroid + CMDist

### Step 1: Define the concept centroid

```r
dehumanisation_seeds <- list(
  animalistic   = c("animal", "beast", "vermin", "creature", "swarm"),
  objectifying  = c("object", "tool", "commodity", "thing", "property"),
  moral_disgust = c("trash", "worthless", "disgusting", "filth", "waste")
)

# Pool all seeds and compute a single centroid
all_seeds <- unlist(dehumanisation_seeds)
in_vocab  <- intersect(all_seeds, rownames(word_vectors))
dehumanisation_centroid <- colMeans(word_vectors[in_vocab, ])
```

### Step 2: Measure CMDist (cosine proximity to centroid)

```r
# For a target word (e.g. a group label in ALC context embeddings)
cmd_to_dehumanisation <- function(target_vec, centroid) {
  as.numeric(
    text2vec::sim2(
      matrix(target_vec, nrow = 1),
      matrix(centroid,   nrow = 1),
      method = "cosine"
    )
  )
}

# In a conText pipeline: use the ALC context embedding as target_vec
# Higher value = stronger alignment with dehumanising semantic space
```

### Interpretation

> How strongly does the discourse around this target align with the dehumanising semantic region?

A high CMDist score means the target's context embedding falls near the centroid of animalistic, objectifying, and morally degrading vocabulary — it does **not** claim the term is positioned on one specific axis.

---

## Operationalisation: Essentialism → Direction + Projection

### Conceptual structure

Essentialism forms a **contrastive semantic dimension**:

```
changeable / social  ←→  innate / fixed
```

- Left pole (anti-essentialist): effort, behavior, improvement, choice, learned
- Right pole (essentialist): genetics, born, bone, height, nature, hardwired

This is directional because the contrast is genuinely bipolar and each word pair captures the same distinction from a slightly different angle.

### Step 1: Build the semantic direction

```r
# Contrastive pairs: (social/changeable, innate/fixed)
anti_essentialist <- c("effort", "behavior", "improvement", "choice", "learned")
essentialist      <- c("genetics", "born", "bone", "height", "hardwired")

in_anti <- intersect(anti_essentialist, rownames(word_vectors))
in_ess  <- intersect(essentialist,      rownames(word_vectors))

# Centroid of each pole
pole_social  <- colMeans(word_vectors[in_anti, ])
pole_innate  <- colMeans(word_vectors[in_ess,  ])

# Semantic direction (not normalised yet)
essentialism_direction <- pole_innate - pole_social

# Normalise for projection
essentialism_direction <- essentialism_direction / sqrt(sum(essentialism_direction^2))
```

### Step 2: Validate with PairDir + ACS

```r
# Build offset vectors from each pair
offsets <- mapply(function(pos, neg) {
  if (pos %in% rownames(word_vectors) && neg %in% rownames(word_vectors))
    word_vectors[pos, ] - word_vectors[neg, ]
  else NA
}, essentialist, anti_essentialist, SIMPLIFY = FALSE)

# Remove pairs not in vocabulary
offsets <- Filter(Negate(is.na), offsets)
offsets_mat <- do.call(rbind, offsets)

# PairDir and ACS from anchor-quality-metrics.md
pairdir <- compute_pairdir(offsets_mat)  # should be > 0.3
acs     <- compute_acs(offsets_mat)      # should be > 0.4
```

If PairDir < 0.3, the word pairs are not capturing a coherent single axis — revise the anchor list before proceeding.

### Step 3: Project targets onto the axis

```r
# Projection of a target ALC embedding onto the essentialism axis
# Positive = more essentialist; negative = more social/constructionist
project_onto_essentialism <- function(target_vec, direction) {
  as.numeric(target_vec %*% direction)
}

# In a conText pipeline: apply to the ALC coefficient vectors
# Compare projections across groups or time periods
```

### Interpretation

> Where on the social-constructionist ←→ biological-determinist axis does the discourse around this group fall?

Positive projection = group framed as inherently fixed; negative = framed as capable of change.

---

## Diagnostic: Choosing Between CMDist and Projection

Before deciding on a method, check whether the concept is structurally directional or clustered:

```r
# Check 1: Do sub-components form a single direction?
# Compute offset vectors for the sub-component pairs and run PairDir
# PairDir > 0.3 → consider direction; < 0.3 → use centroid

# Check 2: Is there a theoretically coherent opposite pole?
# If yes → direction; if no → centroid
# "Human" does not cleanly oppose a dehumanisation cluster
# "Changeable" clearly opposes "innate"

# Check 3: NNS inspection
# Plot nearest neighbours of seed words from each sub-component
# If sub-components cluster together → region
# If sub-components spread along one axis → direction
```

---

## Two-Mechanism Research Design Template

For studies analysing two boundary types simultaneously:

```
symbolic boundary
└── moral exclusion    → dehumanisation → centroid + CMDist
└── cultural exclusion → essentialism   → direction + projection
```

Each mechanism requires its own operationalisation, anchor validation, and interpretation frame. Do not use a single metric (e.g., NNS cosine similarity) for both — the structural difference in the semantic representation is theoretically meaningful, not a technical detail.

### Method statement template

> Cultural exclusion is operationalised as a semantic direction contrasting fixed, innate traits with changeable, socially constructed attributes; position along this axis is measured by projecting ALC context embeddings onto the direction vector. Moral exclusion is operationalised as semantic proximity to a dehumanisation concept centroid, reflecting a clustered rather than directional semantic structure; degree of proximity is measured by cosine similarity (CMDist) between context embeddings and the centroid of animalistic, objectifying, and morally degrading seed words.

---

## Sources

- Lamont, M., & Molnár, V. (2002). The study of boundaries in the social sciences. *Annual Review of Sociology*, 28, 167–195.
- Mendelsohn, J., Tsvetkov, Y., & Jurafsky, D. (2020). A framework for the computational linguistic analysis of dehumanization. *Frontiers in Artificial Intelligence*, 3:55.
- Ohamadike, N., Durrheim, K., & Primus, M. (2026). Anchoring race: improving the construction of race dimensions in word embeddings. *Journal of Computational Social Science*, 9:20.
- See also: `anchor-quality-metrics.md` (PairDir, ACS), `social-bias-framework.md` (dehumanisation measurement), `anchor-selection.md` (seed word validation)
