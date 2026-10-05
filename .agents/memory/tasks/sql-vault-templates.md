---
name: sql-vault-templates
description: Task record for the first batch of SQL Vault templates — four RAG knowledge tables and two chat tables, written as portable SQL inside Markdown.
status: in-progress
---

# SQL Vault templates — knowledge base and chat

First content in this repository. The vault had a README describing a structure and
nothing inside it; this adds the first two categories.

## Confirmed task list

Agreed with the owner before any file was written.

| # | Task | Branch | Files |
|---|---|---|---|
| 1 | Task record | `docs/sql-vault-templates` | `.agents/memory/tasks/sql-vault-templates.md`, `.agents/index/memory-index.md`, `.gitignore` |
| 2 | Knowledge base templates | `docs/sql-vault-templates` | `knowledge-base/README.md`, `knowledge-base/global-knowledge.md`, `knowledge-base/personal-knowledge.md` |
| 3 | Chat templates | `docs/sql-vault-templates` | `chat/README.md`, `chat/chat-sessions.md` |
| 4 | Release | `docs/sql-vault-templates` | root `README.md`, this record |

## Decisions

- **Markdown files, not `.sql` files.** The repository's README defines the template
  format as Markdown carrying a SQL block, and a template that explains its own query
  is what the vault is for. A bare `.sql` file would drop the explanation.
- **Portable SQL, no dialect.** No `AUTOINCREMENT`, no `ENGINE=`, no pgvector type in
  the SQL. The owner asked for legacy-neutral DDL so a template can be read without
  knowing which engine it targets.
- **`VECTOR(1536)` is a placeholder.** Engines disagree on the vector type — pgvector
  has `vector`, MySQL 9 has `VECTOR`, SQLite has nothing and stores a blob or JSON. The
  SQL uses a neutral spelling and the Explanation names the real type per engine.
- **Auto-increment is prose, not syntax.** `INTEGER PRIMARY KEY` is written bare and the
  per-engine way to get a generated value is described in the Explanation, because
  `AUTOINCREMENT` is SQLite-only and would have made the template dialect-specific.
- **`version` is `INTEGER`.** The request allowed INT or VARCHAR.
- **One branch, not a stack.** The workspace convention puts each task on its own
  stacked branch; the owner asked for a single branch and that instruction outranks the
  convention.
- **The four knowledge tables were supplied twice**, once in English and once in Thai.
  Treated as one source. Each table is defined once, in one file; the personal
  templates reference the global ones by name rather than restating their DDL, and the
  category READMEs point rather than copy.

## Entries

### 2026-10-05 — Task 1 — `docs/sql-vault-templates`

Created the memory tree, which did not exist: this record and
`.agents/index/memory-index.md`. Added `.gitignore` excluding `.agents/plans/`,
verified with `git check-ignore -v` — without it the working plan was one `git add -A`
from being published. No template files written yet.

`LICENSE` and `README.md` show as modified in the working tree. The diff is line
endings only — CRLF on disk against LF in the commit — with no content change. Both
are left untouched and never staged, so the template work does not carry an unrelated
whitespace diff.