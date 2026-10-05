# 🗄️ SQL Vault

A centralized repository for SQL examples, table structures, and templates for various systems. Designed for readability, quick searching, and easy adaptation into your own database projects.

Every template is a Markdown file that carries the SQL, the parameters it expects, and an explanation of the logic behind it.

---

**📂 Repository Structure**

Data is organized into directories based on the type of system or feature (`{:type}`):

*   **`/{type}/README.md`** — The central index for a category, with file pointers and short descriptions of every template in that folder.
*   **`/{type}/{template-name}.md`** — The deep-dive file containing the actual SQL syntax, expected parameters, and full explanations.

**Current layout:**
```text
sql-vault/
├── README.md                    <-- Main index (You are here)
├── knowledge-base/              <-- RAG tables: versioned text and vectors
│   ├── README.md                <-- Index for the knowledge-base templates
│   ├── global-knowledge.md      <-- knowledge_contents, knowledge_embeddings
│   └── personal-knowledge.md    <-- personal_knowledge_contents, personal_knowledge_embeddings
└── chat/                        <-- Chat history: sessions, messages, message vectors
    ├── README.md                <-- Index for the chat templates
    ├── session-messages.md      <-- chat_sessions, chat_messages
    └── message-embeddings.md    <-- chat_message_embeddings
```

---

**📚 Templates**

| Category | Template | Tables |
|---|---|---|
| `knowledge-base/` | [`global-knowledge.md`](knowledge-base/global-knowledge.md) | `knowledge_contents`, `knowledge_embeddings` |
| `knowledge-base/` | [`personal-knowledge.md`](knowledge-base/personal-knowledge.md) | `personal_knowledge_contents`, `personal_knowledge_embeddings` |
| `chat/` | [`session-messages.md`](chat/session-messages.md) | `chat_sessions`, `chat_messages` |
| `chat/` | [`message-embeddings.md`](chat/message-embeddings.md) | `chat_message_embeddings` |

Each table is defined exactly once, in exactly one template. Where two templates share a
design, the second refers to the first rather than restating it.

---

**🔧 Dialect Notes**

The SQL is written to be dialect-neutral: it uses no vendor-specific syntax, so a template
reads the same on any engine and runs as written on any engine that supports composite
primary keys. Two constructs have no portable form and are handled deliberately:

*   **`VECTOR(1536)`** is a placeholder, not a real type. Each template carries a table of
    the real type per engine, and the width must match your embedding model.
*   **Generated message ids** are declared as a bare `INTEGER PRIMARY KEY`, with the
    per-engine spelling (`AUTOINCREMENT`, `GENERATED ALWAYS AS IDENTITY`, `AUTO_INCREMENT`,
    `IDENTITY`) in the template's Explanation.

Each template also notes what its schema does *not* enforce — foreign keys need enabling
per connection on SQLite, and some cascades stop short.

---

**📝 Standard Template Format**

When template files (`.md`) are added, they will follow a standard structure consisting of:
1. **Objective:** A brief explanation of what the query achieves.
2. **SQL Statement:** The well-formatted SQL code.
3. **Explanation:** Detailed logic, conditions, and parameters used in the query.
