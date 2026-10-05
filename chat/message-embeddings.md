# Chat Message Embeddings

Vectors for chat messages, one row per message per embedding model.

## Objective

Define the table that holds message vectors, so a single message can carry embeddings
from several models at once, and so deleting a message takes its vectors with it.

The design is the same content/embedding split used by the knowledge base in
[`../knowledge-base/`](../knowledge-base/) — vectors stored apart from the text they were
generated from, keyed per model. The reasoning behind that split is written once, in
[`../knowledge-base/global-knowledge.md`](../knowledge-base/global-knowledge.md), and is
not repeated here. What is specific to chat is the shape of the key and the cascade.

## SQL Statement

```sql
CREATE TABLE chat_message_embeddings (
    message_id INTEGER      NOT NULL,
    model_name VARCHAR(255) NOT NULL,
    embedding  VECTOR(1536) NOT NULL,
    PRIMARY KEY (message_id, model_name),
    FOREIGN KEY (message_id) REFERENCES chat_messages (id) ON DELETE CASCADE
);

CREATE INDEX idx_chat_message_embeddings_model
    ON chat_message_embeddings (model_name);
```

## Explanation

### The composite primary key

`(message_id, model_name)` is the primary key, so one message holds one vector per model
and nothing more.

Putting `model_name` in the key is what makes multi-model support possible without
duplication. Re-embedding a message with a model that already has a row for it is a
constraint violation rather than a second row — and that is the behaviour you want,
because a duplicate vector for the same message comes back twice in a search and skews
the ranking. It also lets a reindex run safely: new-model rows go in beside old-model
rows, and the old ones are deleted when the reindex finishes.

The alternative — a `model_name` column outside the key, with duplicates permitted —
turns every re-embedding mistake into a silently wrong search result, and there is no
cheap way to find the duplicates afterwards.

### The vector column

`embedding` is written `VECTOR(1536)`, which is a placeholder rather than a real type:
no engine implements that spelling. Substitute per engine, and match the width to the
model you actually use.

| Engine | Type to use |
|---|---|
| PostgreSQL with pgvector | `vector(1536)` — needs the extension, plus an `ivfflat` or `hnsw` index on `embedding` for search to be fast |
| MySQL 9+ | `VECTOR(1536)` — natively supported |
| SQLite | `BLOB` or `TEXT` holding a JSON array; comparison happens in the application |
| SQL Server | `VARBINARY(MAX)` or `VARCHAR(MAX)` of JSON — no native vector type |

`1536` is the dimension of `openai/text-embedding-3-small`, which is why it appears
throughout this vault. `qwen/qwen3-embedding-8b` produces 4096, so a project using it
must widen every vector column to match — a mismatch is rejected by the column rather
than silently truncated, which is the better failure.

`TEXT` is what the source schema specified, and it is a defensible choice: a JSON array
is portable across every engine including the ones with no vector type at all. The cost
is that distance comparison moves into the application, and a full scan of the table per
query unless you narrow it first.

### The cascade

`ON DELETE CASCADE` on `message_id` removes a message's vectors when the message is
deleted. It also fires when the message goes because its **session** was deleted, but
only where the engine supports cascading deletes more than one level deep — PostgreSQL
and MySQL do, SQLite does not.

On SQLite, deleting a session cascades to `chat_messages` and stops there, leaving this
table's rows orphaned with a `message_id` that no longer resolves. Nothing in the schema
prevents it, and nothing reports it. The two ways out are a `DELETE FROM
chat_message_embeddings WHERE message_id IN (SELECT id FROM chat_messages WHERE
session_id = ?)` before deleting the session, or triggers. Check which engine you are on
before assuming the cascade reached this table.

### The index

`idx_chat_message_embeddings_model` on `model_name` is the same index the knowledge
tables carry. A vector search asks which model to search and compares only that model's
vectors; without this index every query compares across all models, mixing rankings that
were never comparable.

Note what this index cannot do: it narrows by model, not by session. Finding the
embeddings for one conversation means joining through `chat_messages` on `session_id`,
which uses the index in
[`session-messages.md`](session-messages.md) § `chat_messages`.

### Foreign keys are not self-enforcing

SQLite requires `PRAGMA foreign_keys = ON` per connection before it honours any
`FOREIGN KEY` in this schema. Without it, the cascade described above does not happen at
all and orphaned vectors accumulate silently.