# Global Knowledge Tables

Shared knowledge every user of a system can read, stored as text in one table and as
vectors in another.

## Objective

Define the two tables that hold global knowledge — the versioned source text, and the
embeddings generated from that text — so that one piece of knowledge can carry several
embedding models at once, and so that a version can be revised without overwriting the
history of what was published before it.

The SQL below is dialect-neutral on purpose. It runs as written on any engine that
implements composite primary keys and `CREATE INDEX`, and it avoids every construct that
would tie it to one vendor. Where an engine needs different syntax, the difference is
described in the Explanation rather than hidden in the statement.

## SQL Statement

```sql
CREATE TABLE knowledge_contents (
    knowledge_key VARCHAR(255) NOT NULL,
    version       INTEGER      NOT NULL,
    content       TEXT         NOT NULL,
    created_at    TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (knowledge_key, version)
);

CREATE TABLE knowledge_embeddings (
    knowledge_key VARCHAR(255) NOT NULL,
    version       INTEGER      NOT NULL,
    model_name    VARCHAR(255) NOT NULL,
    embedding     VECTOR(1536) NOT NULL,
    PRIMARY KEY (knowledge_key, version, model_name),
    FOREIGN KEY (knowledge_key, version)
        REFERENCES knowledge_contents (knowledge_key, version)
        ON DELETE CASCADE
);

CREATE INDEX idx_knowledge_embeddings_model
    ON knowledge_embeddings (model_name);
```

## Explanation

### `knowledge_contents` — the source text

`knowledge_key` names the knowledge; `version` numbers the revisions of it. Both form
the primary key, so `(how-to-reset, 3)` and `(how-to-reset, 4)` are two rows that
coexist. That is the whole reason `version` is a key rather than a column that gets
overwritten: publishing a corrected document must not destroy the text the previous
correction replaced, because the old vectors are still indexed against it.

`content` holds the raw text and nothing else. No markup, no front matter, no parsed
structure. Whatever ingests this decides separately how to split the text into chunks
before embedding, and it records which chunk went where in a table this schema does not
define — the alternative is putting chunk offsets in the content row, which makes every
revision a full rewrite of a row that did not change.

`created_at` defaults to the current timestamp so an insert never has to supply it. The
default is written on the column rather than applied by the application, because a
timestamp supplied by a client is a timestamp the client can get wrong.

### `knowledge_embeddings` — the vectors

Three columns and no content column, and the split is deliberate: the same text may be
embedded by several models, and the vector is long enough that duplicating the text
alongside it would roughly double the table for no benefit.

`model_name` sits inside the primary key, so `(knowledge_key, version, model_name)` is
unique. One message of knowledge holds one vector per model, and re-embedding the same
version with the same model is a conflict rather than a silent second row — which is the
behaviour you want, because a duplicate vector for the same key returns twice in a search
and skews the ranking. Naming a model explicitly also means you can keep vectors from an
older model alongside the new ones while a reindex runs, then drop the old rows when it
finishes.

`FOREIGN KEY (knowledge_key, version)` matches the composite key of `knowledge_contents`
exactly. The reference has to be composite because the parent table has no single-column
key to point at.

`ON DELETE CASCADE` removes the vectors when a knowledge row goes. Without it, deleting a
knowledge entry leaves orphaned vectors that still match searches and return content that
no longer exists — a search index pointing at nothing.

### The `VECTOR(1536)` placeholder

`VECTOR(1536)` is a neutral spelling, not a real type. No engine implements it, and it is
written this way so the template reads correctly on any of them. Substitute per engine:

| Engine | Type to use |
|---|---|
| PostgreSQL with pgvector | `vector(1536)` — requires the extension, and an index (`ivfflat` or `hnsw`) on `embedding` for search to be fast |
| MySQL 9+ | `VECTOR(1536)` — natively supported; also check the `distance` function names for your release |
| SQLite | `BLOB` or `TEXT` holding a JSON array; search happens in the application, so expect to read candidate rows and compare in memory |
| SQL Server | `VARBINARY(MAX)` or `VARCHAR(MAX)` of JSON — no native vector type |

The dimension must match the model, and `1536` is the dimension of OpenAI's
`text-embedding-3-small`, so treat it as an example rather than a default. `qwen3-embedding-8b`,
named in the chat schema's own examples, produces 4096. If you change one model, every
stored vector has to match the new width or the column will reject the write.

### The index on `model_name`

A vector search does not scan every vector and pick the nearest — it asks which model to
search, then compares that model's vectors. Without this index every query compares
across all models at once, and the answer mixes rankings that were never comparable.

### What this schema does not do

It does not enforce foreign keys itself. SQLite, among others, requires
`PRAGMA foreign_keys = ON` per connection before it honours them, and a session that
forgets will silently accept orphaned vectors. Check it at connection start.