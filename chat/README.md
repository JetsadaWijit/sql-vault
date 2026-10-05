# Chat

Table structures for chat history: the conversations, the messages inside them, and the
vectors generated from those messages.

Every table is defined once, in exactly one template. The table below points at those
definitions rather than restating them.

## Templates

| Template | Tables | What it covers |
|---|---|---|
| [`session-messages.md`](session-messages.md) | `chat_sessions`, `chat_messages` | The conversation and its ordered messages. Covers the UUID session key, the cascading delete, and why `AUTOINCREMENT` is left to the per-engine table. |
| [`message-embeddings.md`](message-embeddings.md) | `chat_message_embeddings` | Message vectors, one row per message per model. Covers the composite key, the vector type, and where the session cascade stops on SQLite. |

## The shape

Three tables in a chain. `chat_sessions` holds conversations, `chat_messages` holds their
contents, and `chat_message_embeddings` holds vectors for those messages. Each points at
the one above it, and each deletes downward.

```
chat_sessions
    └── chat_messages        ON DELETE CASCADE
            └── chat_message_embeddings   ON DELETE CASCADE
```

The last step is the one to check on your engine — on SQLite the cascade does not reach
it, and the failure is silent. See
[`message-embeddings.md`](message-embeddings.md) § The cascade.

## Where this matches the knowledge base

`chat_message_embeddings` follows the same rule as the embedding tables in
[`../knowledge-base/`](../knowledge-base/): vectors live apart from the text they came
from, and `model_name` sits inside the primary key so one row holds one vector per model
and a re-embedding is a conflict rather than a duplicate.

The reasoning behind that rule is written once, in
[`../knowledge-base/global-knowledge.md`](../knowledge-base/global-knowledge.md). It is
not restated here, and this template does not restate it either.

## Before using these

- **`AUTOINCREMENT` is not in the statements.** It is SQLite-only, and writing it would
  make every template run on one engine. The per-engine substitution is in
  [`session-messages.md`](session-messages.md) § Why `AUTOINCREMENT` is not in the
  statement.
- **`VECTOR(1536)` is a placeholder**, not a real type. The substitution table is in
  [`message-embeddings.md`](message-embeddings.md) § The vector column, and the width
  must match your embedding model.
- **Foreign keys need enabling per connection** on SQLite — `PRAGMA foreign_keys = ON`.
  Without it neither cascade in the diagram above happens.
- **`chat_sessions.updated_at` is not maintained by this schema.** It has to be set by the
  application or a trigger; see
  [`session-messages.md`](session-messages.md) § `chat_sessions`.