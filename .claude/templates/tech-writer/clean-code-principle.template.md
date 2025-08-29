---
description: Use only sections that apply. Optional sections are marked “(optional)”.
---

# [Principle Name] in [Language]

## Overview

[1–3 lines. Why it matters. Keep pragmatic: how it improves testability,
clarity, or extensibility.]

## Core Concept

- [Short bullets capturing the essence; 3–5 items max.]

## Example

### Scaffolding (optional; so snippets compile)

```[language]
[Only include if needed: types, sentinel errors, tiny helpers.]
````

### BAD — \[Short label of the smell]

```[language]
[Minimal but realistic anti-example. No filler. Inline comments only where helpful.]
```

### GOOD — \[Short label of the fix]

```[language]
[The corrected approach. Show the contract & how it’s honored.]
```

### Testing (optional)

- \[Tiny, table-driven or contract test harness if it adds value.]

## Anti-patterns to Avoid

1. \[One-liner]
2. \[One-liner]
3. \[One-liner]

## \[Language]-Specific Techniques (optional)

- \[Tiny bullets: e.g., Go—consumer-defined interfaces, `errors.Is`, `context` usage.]

### Other Optional heading (optional; e.g., "### Pragmatic scope for 'Gol LSP'")

\[When to add contract tests, when to skip, capability discovery tips, etc.]

## Key Takeaways

- \[3–5 bullets, crisp and implementation-oriented.]

## Related Best Practices (Optional)

See **[best-practices.md](<link to best practices doc>)** for package
structure, where to define interfaces, error mapping, and testing patterns.
