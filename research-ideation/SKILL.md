---
name: research-ideation
description: Design, evaluate, and refine research questions for computational discourse analysis — covering theory-method alignment, operationalisation of abstract constructs, contribution framing, and extension directions.
version: 1.0.0
author: css-skills
tags: [research-design, computational-sociology, discourse-analysis, semantic-change, theory-method-alignment, contribution-framing, sociology-of-culture]
dependencies: []
---

# Research Ideation — Computational Discourse Analysis

## Evaluating a Research Question

A good research question for computational discourse analysis satisfies four criteria:

| Criterion | Question to ask |
|-----------|----------------|
| **Tractability** | Can I operationalise this with text data? What corpus, what unit of analysis? |
| **Theory fit** | Does my method's output directly index the theoretical construct I claim? |
| **Substantive gap** | What would change in our understanding if I found X vs. Y? |
| **Scope** | Is the corpus large enough to detect the effect? Is the effect size I expect detectable with embedding methods? |

---

## Theory-Method Alignment

This is the most common failure point in computational social science papers. The method produces one kind of evidence; the theoretical claim requires another.

### Alignment matrix

| Method output | Valid theoretical claims | Invalid claims |
|---------------|------------------------|----------------|
| Word co-occurrence patterns (NNS, FCM) | Aggregate semantic associations in discourse | Individual beliefs, attitudes, or intentions |
| ALC regression coefficients | Direction/magnitude of semantic shift associated with a covariate | Causal mechanism; individual-level behaviour |
| Topic distributions | Thematic emphasis in a corpus | Conscious framing strategies (without additional evidence) |
| Word frequency change | Changing salience of a concept | Change in meaning (frequency ≠ meaning) |
| Sentiment scores | Aggregate emotional valence in texts | Individual emotional states |

### Diagnostic questions

Before writing the paper:

1. What does my method *literally* compute? (Be specific — cosine similarity between what and what?)
2. Why does that computation serve as a valid proxy for my theoretical concept?
3. What would have to be true about language / discourse for that proxy to be valid?
4. What alternative explanation fits the same pattern?

### Typical alignment fix

> **Before**: "The results show that newspapers engaged in boundary work by positioning science near utility."

> **After**: "The co-occurrence pattern — *science* consistently appearing near *utility* and *improvement* — reflects an aggregate semantic association in early nineteenth-century newspaper discourse that is consistent with an instrumentalist orientation toward scientific authority."

---

## Generating Research Questions

### Strategy 1: Take a known qualitative finding and ask "at what scale and with what variation?"

Qualitative historians have documented the republican synthesis of science and religion in early American thought. A computational question: Does this synthesis appear in the full newspaper corpus across all regions and press types, or is it concentrated in elite political papers? Does it erode over time?

### Strategy 2: Take a theoretical construct and ask "what would it look like in embedding space?"

Symbolic boundaries are supposed to be socially shared — similar distinctions should appear across different discourse communities. Test this: do the same conceptual clusters appear in commercial, political, and provincial papers? Or is the boundary community-specific?

### Strategy 3: Take a methodological affordance and ask "what question only this method can answer?"

ALC regression can estimate *direction* of semantic shift, not just magnitude. What question requires directionality? Answer: which concept moved *toward* which other concept? Did "science" move toward "commerce" or toward "philosophy" over time? Standard frequency analysis cannot answer this.

### Strategy 4: Take a comparison and ask "what is the source of variation?"

Two newspapers, same period, different semantic neighbourhoods for "liberty." What explains the difference — region, press type, political affiliation, readership demographics? Regression-based embedding analysis can estimate each factor's contribution.

---

## Operationalising Abstract Constructs

### Step 1: Decompose the construct

Break the abstract concept into observable sub-components:

| Abstract construct | Decomposition |
|-------------------|---------------|
| "Scientific authority" | Semantic proximity to legitimacy terms (reason, evidence, truth); distance from superstition terms |
| "Civic morality" | Proximity to virtue terms (dignity, honour, duty); proximity to political terms (republic, democracy) |
| "Cultural threat" | Proximity to danger/exclusion terms (tyranny, ignorance, corruption) |

### Step 2: Select indicator words (anchors)

For each sub-component, select 2–5 words that best represent it. Apply the anchor selection criteria in `embedding-analysis/references/anchor-selection.md`.

### Step 3: Validate the operationalisation

```r
# Before analysis: verify that indicator words form the expected cluster
# They should share top NNS neighbours
for (word in indicator_words) {
  cat(word, "→", top_nns(word, embeddings, n = 5), "\n")
}
# If words in the same group have non-overlapping neighbours, the group is invalid
```

### Step 4: State the linking assumption explicitly

In the methods section: "I treat the semantic neighbourhood of *X* as a proxy for [theoretical construct], under the assumption that [linking assumption]. This operationalisation is valid to the extent that [scope condition]."

---

## Contribution Framing

### Three levels of contribution

| Level | Claim | Framing |
|-------|-------|---------|
| Substantive | New empirical finding about a historical / social phenomenon | "We provide the first systematic evidence that..." |
| Theoretical | New conceptual framework or extension | "We distinguish two previously conflated dimensions of..." |
| Methodological | New application or validation of a method | "We demonstrate that ALC regression can operationalise [construct] at scale..." |

Most strong papers make claims at **two** levels. Avoid claiming all three if the methodological contribution is minor (applying an existing method is not itself a methodological contribution).

### Sociological journals

> "We provide large-scale evidence for [theoretical claim], revealing [pattern] that prior qualitative accounts suggested but could not confirm at scale. Our results support [theory A] over [theory B] for this domain."

### Computational social science journals

> "We demonstrate that [method] enables systematic measurement of [construct] across [corpus type], offering a new toolkit for [substantive domain]. Applied to [domain], we find [result]."

### History / historical sociology journals

> "Quantitative analysis of [N] texts confirms that [historical claim], challenging the [consensus view]. The [period/case] shows [specific finding], consistent with [historiographical tradition] and inconsistent with [alternative account]."

---

## Common Extension Directions

### Temporal extension

- Extend the analysis to a later period to test whether patterns persist, reverse, or intensify
- Identify the period when a pre-institutional boundary becomes institutionalised
- Compare periods of stability vs. disruption (war, economic shock, technological change)

### Geographic/comparative extension

- Compare corpora from different regions, countries, or language communities
- Test whether a pattern described for one case is universal or context-specific

### Vocabulary extension

- Add words from adjacent conceptual domains
- Test robustness: do results hold with different but theoretically equivalent anchor words?
- Extend to near-synonyms or semantic field members

### Community / source extension

- Disaggregate by publisher type, author demographic, publication venue
- Compare elite vs. popular discourse
- Compare institutional (government, church) vs. commercial vs. civic discourse

### Methodological extension

- Validate NNS findings with supervised classifier
- Cross-validate with topic model (do topics align with semantic clusters?)
- Add individual-level metadata (author, reader demographics) if available

---

## Key Theoretical Tensions to Address

### Structure vs. Process

**Tension**: Computational methods capture *structure* (the current state of a semantic field); most sociological theories are about *process* (how agents produce and reproduce structure).

**Resolution**: Frame the analysis as recovering "the accumulated trace of discursive practice" — the structure that agents both produced and were constrained by.

### Aggregate vs. Agent

**Tension**: Corpus-level analysis cannot attribute patterns to individual actors or intentions.

**Resolution**: Explicitly define your unit of analysis as *discourse field*, *discourse community*, or *text corpus* — not individual actors. Cite Bourdieu (field theory) or discourse analysis traditions (Laclau & Mouffe) if needed.

### Language Use vs. Social Meaning

**Tension**: Word co-occurrence in text reflects statistical patterns of language use; social meaning involves interpretation, context, and power.

**Resolution**: Make the proxy argument explicit. State what distributional semantics can and cannot tell us. Add a limitations section addressing this boundary.
