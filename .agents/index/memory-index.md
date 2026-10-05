---
name: memory-index
description: Index of this repository's memory — the records a future session reads to continue prior work.
---

# Memory Index

What this repository knows about its own work. Read this at session start, then load
the rows that match the request.

The durable record of completed work lives here, not in `.agents/plans/` — plans are
untracked scratch and are deleted when the work lands.

## Tasks

| File | What it covers | Status |
|---|---|---|
| [`tasks/sql-vault-templates.md`](tasks/sql-vault-templates.md) | The first batch of vault templates: four RAG knowledge tables and two chat tables, written as portable SQL inside Markdown. | done |

## Decisions

None yet.

## State

None yet.

## Sessions

None yet.