# Personal Knowledge Tables

Knowledge scoped to one user, stored as text in one table and as vectors in another.

## Objective

Define the personal counterparts of the global knowledge tables, so that each user's own
notes, drafts, and corrections are versioned and embedded under the same rules — and
kept separate from both the shared knowledge and from every other user.

The structure is deliberately the same shape as
[`global-knowledge.md`](global-knowledge.md): a content table and an embedding table,
each keyed by version, each carrying one row per embedding model. The only structural
difference is that `user_id` leads the key. The column definitions are therefore not
restated here beyond what the shared design fixes; read that file for the reasoning
behind the content/embedding split, the composite key, the cascade, and the vector type.

## SQL Statement

```sql
CREATE TABLE personal_knowledge_contents (
    user_id       VARCHAR(255) NOT NULL,
    knowledge_key VARCHAR(255) NOT NULL,
    version       INTEGER      NOT NULL,
    content       TEXT         NOT NULL,
    created_at    TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, knowledge_key, version)
);

CREATE TABLE personal_knowledge_embeddings (
    user_id       VARCHAR(255) NOT NULL,
    knowledge_key VARCHAR(255) NOT NULL,
    version       INTEGER      NOT NULL,
    model_name    VARCHAR(255) NOT NULL,
    embedding     VECTOR(1536) NOT NULL,
    PRIMARY KEY (user_id, knowledge_key, version, model_name),
    FOREIGN KEY (user_id, knowledge_key, version)
        REFERENCES personal_knowledge_contents (user_id, knowledge_key, version)
        ON DELETE CASCADE
);

CREATE INDEX idx_personal_knowledge_embeddings_model
    ON personal_knowledge_embeddings (user_id, model_name);
```

## Explanation

### `user_id` leads the key

The primary key is `(user_id, knowledge_key, version)` and the foreign key is
`(user_id, knowledge_key, version)`. Two consequences follow, and both matter.

First, two users may hold the same `knowledge_key` at the same `version` without
colliding. That is the point: a personal note is scoped to whoever wrote it, not to a
global namespace, so a user's `meeting-notes-2026` is unrelated to anyone else's.

Second, and more easily missed — a global key and a personal key are not the same
namespace. If a user writes `how-to-reset` personally, that row is a different thing
from the global `how-to-reset` in `knowledge_contents`, and nothing in this schema
merges or shadows it. A search that reads both must decide which wins, and that decision
is the application's, not the database's. Keeping the tables separate makes the
collision visible at query time instead of silently letting one overwrite the other.

### Everything else is inherited

`content`, `created_at`, `version`, and the one-row-per-model rule behave exactly as in
[`global-knowledge.md`](global-knowledge.md). That file is the single definition of why
content and embeddings are separate tables, why `version` is part of the key rather than
an overwritten column, and why `ON DELETE CASCADE` is present. Restating those reasons
here would produce two explanations to keep in step, and this vault's rule is one
definition per table.

The `VECTOR(1536)` placeholder carries the same caveat: substitute the real type per
engine, and keep the dimension matched to the model.

### The composite index

`idx_personal_knowledge_embeddings_model` is on `(user_id, model_name)`, not on
`model_name` alone. Every query against this table is scoped to one user, so leading with
`user_id` lets the engine narrow to that user's vectors before considering model at all.
An index on `model_name` alone would be nearly useless here — it would return every
user's vectors for a popular model and filter the rest away.

That is also the security-relevant property. Scoping by `user_id` in the index matches
scoping by `user_id` in the key, so a query that forgets its `WHERE` clause degrades into
a slow full scan rather than a fast leak. That is worth confirming deliberately: check
that every read of these tables filters by `user_id`, because the schema cannot enforce
it and no index will.

### Cascade and enforcement

`ON DELETE CASCADE` removes a user's vectors when their personal knowledge row is
deleted — deleting the whole `(user_id, ...)` subtree in one statement rather than
leaving orphans for a search to return.

As with the global tables, foreign keys are not self-enforcing. SQLite needs
`PRAGMA foreign_keys = ON` per connection. Where that pragma is not set, deleting a
personal knowledge row leaves its embeddings behind, and those vectors remain
searchable.