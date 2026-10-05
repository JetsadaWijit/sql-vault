# Knowledge Base

Table structures for retrieval-augmented generation: versioned source text stored apart
from the vectors generated from it, at two scopes — knowledge shared by every user, and
knowledge belonging to one user.

Every table here is defined once, in exactly one template. The indexes below point at
those definitions; they do not restate them.

## Templates

| Template | Tables | What it covers |
|---|---|---|
| [`global-knowledge.md`](global-knowledge.md) | `knowledge_contents`, `knowledge_embeddings` | Shared knowledge, versioned, with one vector per embedding model. The content/embedding split, the composite key, and the vector type are explained here. |
| [`personal-knowledge.md`](personal-knowledge.md) | `personal_knowledge_contents`, `personal_knowledge_embeddings` | The same design scoped to a single user by `user_id`. Read the global template first — this one inherits its reasoning rather than repeating it. |

## The shape both sets share

Four tables, two pairs. In each pair the first table holds the raw text and the second
holds only vectors, and the second points back at the first through a foreign key that
matches its full primary key.

That split exists because the same text is embedded by several models, and a vector is
long enough that carrying the text beside it would roughly double the table. Keeping
them apart lets one content row have any number of embedding rows, one per model.

Two rules carry across all four tables:

- **`version` belongs to the primary key.** A revision creates a row; it never
  overwrites one. The vectors indexed against the old text stay valid because the old
  text stays.
- **`model_name` belongs to the primary key** of every embedding table. One row per
  model per content row, so re-embedding is a conflict rather than a silent duplicate,
  and vectors from an old model can coexist with a new one while a reindex runs.

## Before using these

The `VECTOR(1536)` in every statement is a placeholder, not a real type — no engine
implements it as written. The substitution table is in
[`global-knowledge.md`](global-knowledge.md) § `The VECTOR(1536) placeholder`, and the
dimension must be matched to whichever embedding model you actually use.

Foreign keys are not self-enforcing either. SQLite requires `PRAGMA foreign_keys = ON`
per connection, and without it a deleted content row leaves orphaned vectors that still
match searches.