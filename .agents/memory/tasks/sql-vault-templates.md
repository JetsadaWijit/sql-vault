---
name: sql-vault-templates
description: Task record for the first batch of SQL Vault templates — four RAG knowledge tables and two chat tables, written as portable SQL inside Markdown.
status: done
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

### 2026-10-05 — Task 2 — `docs/sql-vault-templates`

Wrote `knowledge-base/README.md`, `knowledge-base/global-knowledge.md`, and
`knowledge-base/personal-knowledge.md`.

Each of the four tables is defined exactly once. The personal template inherits the
content/embedding rationale from the global one by reference rather than restating it,
because two copies of the same reasoning would drift. Verified by grep: four
`CREATE TABLE` statements across three files, no statement appearing twice.

`personal-knowledge.md` adds one point the global template does not cover — that a
personal `knowledge_key` and a global one are separate namespaces, so a user writing
`how-to-reset` does not shadow the shared entry, and which one wins is the
application's decision rather than the database's.

### 2026-10-05 — Task 3 — `docs/sql-vault-templates`

Wrote `chat/README.md`, `chat/session-messages.md`, and `chat/message-embeddings.md`.
This replaced a two-table, one-template task: the owner supplied the full chat schema
after the plan was first written, and it carries a third table
(`chat_message_embeddings`) while dropping `message_index` from `chat_messages`.

Kept `INTEGER PRIMARY KEY` bare rather than `AUTOINCREMENT`, on the owner's
confirmation that portability wins, and put the per-engine spelling in a table. Verified
by extracting every ```sql block and grepping it: no `AUTOINCREMENT` survives in any
statement.

Two behaviours in the source schema are documented as gaps rather than quietly
preserved:

- `chat_sessions.updated_at` defaults correctly on insert and then goes stale, since
  nothing in the schema advances it when a message arrives. Left to the application or a
  trigger, and said so.
- The `chat_messages` → `chat_message_embeddings` cascade does not fire on SQLite, which
  cascades only one level. Deleting a session there leaves orphaned vectors silently.
  Documented with both remedies.

All seven tables across the vault are now defined exactly once, verified by grep.

### 2026-10-05 — Task 4 — `docs/sql-vault-templates`

Rewrote the root `README.md`. It had described a *planned* structure with placeholder
folder names and said categories "will be added soon"; it now lists the two categories
that exist, indexes all four templates against the tables they define, and states the
dialect policy — the two constructs with no portable form, and where each template
explains what its schema does not enforce.

The README's own `Objective` / `SQL Statement` / `Explanation` format was kept. Every
template follows it, which is why the section headers match exactly across all four.

Record closed: `status: done`.

---

## Verification

Extracted every ```sql block and checked the vault mechanically rather than by eye:

- seven `CREATE TABLE` statements, each appearing exactly once across five files
- no `AUTOINCREMENT`, `ENGINE=`, or vendor vector type inside any SQL block — the
  `pgvector` matches are in prose substitution tables, which is where they belong
- `.agents/plans/` stayed out of every commit; confirmed with `check-ignore` before the
  first commit and by inspecting the staged file list before each
- `LICENSE` was never staged and still differs from `master` by line endings only

Not verified: the SQL was never executed. No engine was available in the sandbox, so
these templates are correct by construction and by reading, not by running. Anyone
adopting them should run the statements against their target engine once — which is also
the only way to settle the `VECTOR` and identity-column substitutions for their setup.

Nothing was pushed and no pull request was opened; both are gated on the owner's yes.