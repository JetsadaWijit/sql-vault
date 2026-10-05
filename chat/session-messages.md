# Chat Sessions and Messages

The two tables behind a chat window: one row per conversation, and the ordered messages
inside it.

## Objective

Define `chat_sessions` and `chat_messages` so that a conversation's messages are removed
automatically when the conversation is, and so a message can be addressed by a stable
integer that the embedding table can reference.

The SQL is dialect-neutral. It avoids `AUTOINCREMENT`, which is SQLite-only, and the
Explanation says how each engine generates the value instead.

## SQL Statement

```sql
CREATE TABLE chat_sessions (
    id         VARCHAR(36)  NOT NULL,
    title      VARCHAR(255) NOT NULL DEFAULT 'New Chat',
    created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
);

CREATE TABLE chat_messages (
    id         INTEGER      NOT NULL,
    session_id VARCHAR(36)  NOT NULL,
    role       VARCHAR(20)  NOT NULL,
    content    TEXT         NOT NULL,
    created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    FOREIGN KEY (session_id) REFERENCES chat_sessions (id) ON DELETE CASCADE
);

CREATE INDEX idx_chat_messages_session
    ON chat_messages (session_id, id);
```

## Explanation

### `chat_sessions`

`id` is a `VARCHAR(36)` primary key — a UUID. `VARCHAR(36)` is the width a
hyphenated UUID needs, so a client can generate the value before the insert and the
session exists the moment the user opens a window. An auto-increment id would force the
session to be created only on first message, which is a different and less useful
lifetime.

`title` defaults to `'New Chat'` so a session is insertable with nothing but an id. An
untitled session that fails to insert because someone forgot a title is an annoying
failure; a session called `New Chat` is a visible one the user can rename.

`updated_at` is **not** maintained by this schema. It defaults to the current timestamp
on insert, which is correct for a new session and wrong for every later message — the
value silently goes stale and a query for "recently active conversations" starts
returning creation order. Either update it explicitly in the same statement that inserts
the message, or use a trigger; the application doing it in one transaction with the
message insert is the simpler of the two.

### `chat_messages`

`id` is a bare `INTEGER PRIMARY KEY`. That declares the column unique and makes it the
foreign-key target without adding any engine-specific syntax. The generated value is
deliberately absent — see below.

`role` is `VARCHAR(20)`, enough for `user`, `assistant`, `system`, and `tool`. It is not
constrained to a list, because a check constraint or an enum is dialect-specific and
would defeat the portability of the template. Validating the value is the application's
job; if you would rather the database refuse it, add a `CHECK` and accept that the
statement is no longer portable.

`created_at` carries the time the message was sent, which is what ordering a
conversation actually needs.

`ON DELETE CASCADE` on `session_id` is the reason this template exists. Deleting a
session should take its messages with it in one statement; without the cascade you
either write a second `DELETE` and hope both succeed, or leave orphaned messages that
still reference a session that no longer exists. Note that it cascades to the embeddings
too, but only transitively — see
[`message-embeddings.md`](message-embeddings.md).

### Why `AUTOINCREMENT` is not in the statement

`INTEGER PRIMARY KEY AUTOINCREMENT` is SQLite's way of asking for a generated value, and
it appears in the source schema this template came from. It is not portable: PostgreSQL
wants `GENERATED ALWAYS AS IDENTITY`, MySQL wants `AUTO_INCREMENT`, and SQL Server wants
`IDENTITY(1,1)`. Writing any one of them makes the template run on that engine only.

So the statement declares the key and stops, and the generated value is added per engine:

| Engine | Add to the `id` column |
|---|---|
| SQLite | `AUTOINCREMENT` |
| PostgreSQL | `GENERATED ALWAYS AS IDENTITY` |
| MySQL | `AUTO_INCREMENT` |
| SQL Server | `IDENTITY(1,1)` |

There is one behavioural difference worth knowing: SQLite's bare `INTEGER PRIMARY KEY`
reuses an id after the highest row is deleted, while `AUTOINCREMENT` never reuses one.
For message ids this rarely matters, since nothing outside the database should be
holding one.

### Message order

There is no `message_index` column. Message order comes from `id`, and the composite
index `(session_id, id)` is what makes that cheap: one index serves both "every message
in this session, in order" and the referential check on `session_id`.

Reading a conversation is then `SELECT ... WHERE session_id = ? ORDER BY id`, which the
index satisfies without a sort.

### Two things this schema does not do

**It does not check foreign keys by itself.** SQLite ignores `FOREIGN KEY` unless
`PRAGMA foreign_keys = ON` is set per connection, so on SQLite a session id that does not
exist will be accepted. Set the pragma at connection start, not once at startup.

**It does not enforce that `updated_at` moves.** Nothing in the schema keeps it
honest — see the note above.