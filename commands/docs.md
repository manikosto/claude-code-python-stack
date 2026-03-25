---
description: Look up current documentation for a library or topic via Context7.
---

# /docs

## Purpose

Look up up-to-date documentation for a library, framework, or API and return a summarized answer with code snippets.

## Usage

```
/docs [library name] [question]
```

Example: `/docs "FastAPI" "How do I add middleware?"`

## Workflow

1. **Resolve library ID** -- Call Context7 `resolve-library-id`
2. **Query docs** -- Call `query-docs` with library ID and question
3. **Summarize** -- Return concise answer with code examples
