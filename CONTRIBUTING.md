# Contributing

Contributions are welcome — whether fixing an error, improving an explanation, or adding a new SKILL.

## Adding a New SKILL

1. Create a directory under the skill's category name (kebab-case):
   ```
   skills/your-skill-name/
   ├── SKILL.md
   └── references/
       └── overview.md    (at minimum)
   ```

2. Follow the SKILL.md format:
   ```markdown
   ---
   name: your-skill-name
   description: One sentence, <150 chars, specific use cases.
   version: 1.0.0
   author: your-name
   tags: [tag1, tag2, ...]
   dependencies: [package1, package2]
   ---

   # Skill Title

   ## When to Use
   (decision table)

   ## Quick Start
   (minimal working code)

   ## Common Workflows
   (2-4 complete patterns)

   ## Common Issues
   (troubleshooting table)
   ```

3. Add an entry to the Skills table in `README.md`.

## Improving an Existing SKILL

- Fix factual errors, outdated package versions, or broken code examples via pull request
- Add a new `references/` document for a topic not yet covered
- Update `version` in the frontmatter when making substantive changes

## SKILL Quality Standards

- Every major section must include **complete, runnable code** (no pseudo-code)
- Troubleshooting tables must include **actual error messages**, not vague descriptions
- Decision tables should have a clear **when NOT to use** column
- Reference files should be substantive (>50 lines) — not just links to external docs

## Scope

This library focuses on:
- **Reproducible research workflows in R** (`{targets}`, `{WORCS}`, `{renv}`) for structured and scalable computational pipelines
- **Word embedding–based computational text analysis** (e.g., ALC, GloVe) for measuring meaning and semantic associations
- **Academic writing assistance** for computational social science (e.g., structuring arguments, refining methodology, improving clarity)
- **Computational sociology research design** (e.g., theory–method alignment, operationalization)