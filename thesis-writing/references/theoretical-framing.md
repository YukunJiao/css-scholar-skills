# Theoretical Framing — Detailed Reference

## Boundary Concepts Typology

### Symbolic Boundary vs. Social Boundary

These two terms are often used interchangeably but are analytically distinct:

| Concept | Definition | Level |
|---------|-----------|-------|
| **Symbolic boundary** | Cultural criteria used by actors to categorise social interactions in order to make judgements about group membership (Vila-Henninger 2015) | Cognitive / discursive |
| **Social boundary** | Differential access to resources, opportunities, or recognition that results from symbolic distinctions being institutionally enacted | Structural / material |

**Key relation**: Symbolic boundaries are the raw material from which social boundaries are made (Lamont & Molnár 2002). Discourse analysis typically captures symbolic boundaries; social boundaries require institutional or outcome data.

**Implication for corpus analysis**: A finding that a group is semantically proximate to dehumanising vocabulary is evidence of a *symbolic* boundary. Claims about resulting exclusion from resources require additional evidence.

---

### Boundary Work

**Definition**: The decisions and rhetorical acts through which actors determine group membership — who is included, who is excluded, and on what criteria (Gieryn 1983; Vila-Henninger 2015).

**Evidence required**:
- Identifiable agents producing specific texts
- Rhetorical strategies: contrast, exclusion, expansion, protection of autonomy
- Traceable interests: who benefits from drawing the boundary this way?

**Not a fit for**: aggregate NNS patterns, corpus-level co-occurrence — these cannot be attributed to strategic decisions by identifiable actors.

---

### Boundary Strength

**Definition**: The degree of commitment that an actor has to a given classificatory judgement (Vila-Henninger 2015).

**In discourse**: a strong boundary = the classificatory distinction is sharp, consistent, and resistant to ambiguity in the text. A weak boundary = the distinction is blurred, contested, or varies across sub-groups.

**Computational proxies**:

| Proxy | What it captures | How to measure |
|-------|-----------------|----------------|
| Semantic distance between in-group and out-group centroids | Sharpness of the categorical distinction in embedding space | Cosine distance between group-label ALC embeddings |
| Variance in ALC coefficient across sub-corpora | Consistency of the distinction across communities | SD of normed estimates across press types / regions |
| CMDist magnitude for exclusion seeds | How strongly the target term is embedded in dehumanising/stigmatising semantic region | Cosine similarity to concept centroid |
| PairDir of exclusion/inclusion offset vectors | Internal coherence of the classificatory dimension | PairDir score (see `anchor-quality-metrics.md`) |

**Interpretation rule**: a large, consistent CMDist or projection score across sub-groups indicates a *strong* symbolic boundary; a small or highly variable score indicates a *weak* one.

---

### Boundary Change

**Definition**: Changes over time in the forms and strength of classificatory distinctions through which symbolic boundaries are constructed (Vila-Henninger 2015).

Two analytically separable dimensions of change:

| Dimension | What changes | Computational measure |
|-----------|-------------|----------------------|
| **Form** | Which semantic dimension organises the distinction (e.g. moral → biological) | Shift in NNS cluster composition; change in direction of ALC coefficient vectors |
| **Strength** | How sharp or committed the distinction is | Change in magnitude of CMDist / projection score across time periods |

**Design guidance**: Use a time-factor ALC regression to estimate both simultaneously. Coefficient *direction* captures form change; coefficient *magnitude* (normed estimate) captures strength change.

```r
# Time-factor model for boundary change
model <- conText::conText(
  formula          = . ~ time_period,
  data             = toks_ctx,
  pre_trained      = embeddings,
  transform        = TRUE,
  transform_matrix = transform,
  bootstrap        = TRUE,
  num_bootstraps   = 100
)
# Direction of each period's coefficient = form of boundary at that time
# Normed estimate magnitude = strength of boundary at that time
# Change in normed estimate across periods = boundary change
```

---

### Operationalising Symbolic Boundaries via Exclusion/Inclusion Patterns

Symbolic boundaries are operationalised through patterns of **exclusion and inclusion** — the semantic associations that capture how actors enact classificatory distinctions in practice (Vila-Henninger 2015).

**In embedding analysis this means**:

- **Inclusion**: semantic proximity of a group label to legitimising, valued vocabulary (dignity, rights, community, reason)
- **Exclusion**: semantic proximity of a group label to stigmatising, devalued, or dehumanising vocabulary (threat, filth, animal, object)
- **Boundary** = the contrast between the two proximity profiles

```r
# Operationalise symbolic boundary as inclusion–exclusion contrast
inclusion_seeds <- c("dignity", "rights", "community", "person", "citizen")
exclusion_seeds <- c("threat", "burden", "filth", "animal", "object")

boundary_score <- function(target, wv, incl_seeds, excl_seeds) {
  prox_incl <- proximity_to_seeds(target, incl_seeds, wv)
  prox_excl <- proximity_to_seeds(target, excl_seeds, wv)
  # Positive = more included; negative = more excluded
  prox_incl - prox_excl
}
```

The **magnitude** of this score operationalises boundary strength; its **change over time** operationalises boundary change.

---

## Choosing Between Boundary Work and Cultural Classification

These two frameworks are often confused because they address similar phenomena. The choice depends on **what kind of evidence you have**.

### Boundary Work (Gieryn 1983, 1999)

**Core claim**: Scientists (and other actors) strategically deploy rhetorical resources to demarcate science from non-science, for purposes of professional authority, resource allocation, and credibility.

**Evidence required**:
- Identifiable agents (scientists, editors, preachers) producing specific texts
- Rhetorical strategies: contrast, exclusion, expansion, protection of autonomy
- Traceable interests: who benefits from drawing the boundary this way?

**Not a fit for**: aggregate NNS patterns, corpus-level semantic structure, co-occurrence matrices — none of these can be attributed to strategic intent.

### Cultural Classification (Durkheim 1912; Zerubavel 1991; Douglas 1966)

**Core claim**: Classifications are collective cognitive structures — socially shared distinctions that organise perception and action. They are not strategic acts but sedimented social forms reproduced through practice.

**Evidence required**:
- Aggregate pattern across many texts/actors
- Shared structure (similar patterns across different newspapers/speakers)
- Historical change in what gets grouped together

**Fits well for**: ALC semantic neighbourhoods, co-occurrence structure, diachronic embedding analysis.

### Symbolic Boundaries (Lamont & Molnár 2002)

**Core claim**: Symbolic boundaries are conceptual distinctions used by social actors to categorise persons, practices, and things. They are the raw material from which social boundaries (differential access to resources) are made.

**Position in hierarchy**: a bridge concept — applicable to both individual rhetorical acts AND aggregate discourse structure. More flexible than either pure boundary work or pure classification theory.

**Good for**: when your analysis spans both micro (individual texts) and macro (corpus-level) evidence.

---

## Theory-Method Alignment Checklist

Before writing the conceptual framework section, verify:

- [ ] My method's output is: `_________________________________`
- [ ] The theoretical concept I claim this measures: `_________________________________`
- [ ] The linking assumption (why output = concept): `_________________________________`
- [ ] Alternative interpretations of the same output: `_________________________________`
- [ ] What evidence would falsify my interpretation: `_________________________________`

### Example (ALC analysis)

| Check | Answer |
|-------|--------|
| Method output | Cosine similarity between context embeddings and target word vectors |
| Theoretical concept | Cultural position of a concept in collective discourse |
| Linking assumption | Discourse co-occurrence reflects shared cognitive/cultural categorisation |
| Alternative interpretation | Co-occurrence reflects topic proximity, not cultural evaluation |
| Falsification | If NNS patterns are explained entirely by topic distributions (test with LDA) |

---

## Addressing the Structure vs. Process Distinction

Computational text analysis captures *structure* (the state of a discourse field at a point in time). Most sociological theories are about *process* (how agents produce and reproduce that structure). Bridge this gap explicitly:

> "The ALC analysis captures the aggregate semantic structure that resulted from — and in turn constrained — the boundary-making activities of [agents]. We do not claim to observe these activities directly; rather, we treat the semantic structure as the accumulated trace of discursive practice across [N] texts and [time period]."

---

## Operationalising Multi-Dimensional Frameworks

If your theoretical contribution is a two-dimensional or multi-dimensional framework, each dimension must map to a distinct empirical measure:

### Template

| Dimension | Operationalisation | Method |
|-----------|-------------------|--------|
| Dimension 1: [name] | [how measured in data] | [specific target words / NNS / coefficient] |
| Dimension 2: [name] | [how measured in data] | [specific covariate / subgroup analysis] |

### Example (epistemic boundary study)

| Dimension | Operationalisation | Method |
|-----------|-------------------|--------|
| Boundary classification | Which semantic register does a concept occupy? | NNS of target words |
| Normative orientation | How positively does each community orient toward the boundary? | Press type coefficients in conText regression |

### Example (symbolic boundary / othering study)

| Dimension | Theoretical construct | Semantic structure | Method |
|-----------|----------------------|-------------------|--------|
| Moral exclusion | Dehumanisation | Cluster / region | Centroid of animalistic + objectifying + moral-disgust seeds; CMDist |
| Cultural exclusion | Essentialism | Axis / direction | Contrastive direction (changeable ←→ innate); projection of ALC embeddings |

**Key principle**: the method choice (CMDist vs. projection) must follow from the geometric structure of the concept, not from convenience. Dehumanisation is not directional — forcing it onto an axis produces low PairDir scores and unstable estimates. See `embedding-analysis/references/symbolic-boundary-measurement.md`.

Without this mapping, the two-dimensional framework remains rhetorical — it must do empirical work.
