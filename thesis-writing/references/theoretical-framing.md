# Theoretical Framing — Detailed Reference

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
