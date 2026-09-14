# MCP Shared Memory Server — Master Study Guide

**What this document is.** A teaching document for the MCP Shared Memory Server in this
repository. It starts from absolute basics and ends at the level you need to defend the
project in a software engineering interview.

**How it was written.** Every claim below was checked against the code in this repository,
in this order: implementation first, then tests, then migrations, then `docs/architecture.md`,
then `README.md`, then git history. Where the documentation and the implementation disagree,
the disagreement is called out and the implementation is treated as the truth. Where something
could not be verified from the repository, the document says so in those words.

**One running example.** Almost everything is explained with the same two sentences:

```
EARLIER KNOWLEDGE:   "The job queue runs on Redis."
LATER KNOWLEDGE:     "The job queue runs on PostgreSQL."
```

**How to read it.** Part 1 and Part 2 need no background. Part 3 onwards builds up. Every
technical word is defined before it is used. If you hit a word you do not know, it is in the
glossary at the end.

---

## Table of contents

| Part | Subject |
|---|---|
| 1 | What does this project actually do? |
| 2 | The project name, word by word |
| 3 | Architecture |
| 4 | Every important word, defined |
| 5 | The actual MCP tools |
| 6 | Tracing `memory_remember` end to end |
| 7 | Database schema |
| 8 | Revision vs supersession |
| 9 | Concurrency from zero |
| 10 | Optimistic concurrency |
| 11 | Concurrency testing |
| 12 | Idempotency |
| 13 | Idempotency vs deduplication |
| 14 | Transactions |
| 15 | Database constraints and invariants |
| 16 | Retrieval from zero |
| 17 | PostgreSQL full-text search |
| 18 | Semantic search |
| 19 | Hybrid retrieval |
| 20 | Stale memory |
| 21 | Structural stale-memory exclusion |
| 22 | Reciprocal Rank Fusion |
| 23 | Background embeddings |
| 24 | PostgreSQL background jobs |
| 25 | Token-budgeted context |
| 26 | Failure handling |
| 27 | Project isolation |
| 28 | Provenance |
| 29 | Testing strategy |
| 30 | Mutation testing and negative verification |
| 31 | Retrieval evaluation |
| 32 | Performance |
| 33 | Tech stack |
| 34 | Repository structure and the 20 files to read |
| 35 | Phase-by-phase reconstruction |
| 36 | Current limitations |
| 37 | Verifying your four resume bullets |
| 38 | Interview explanations at five lengths |
| 39 | Sixty-plus interview questions |
| 40 | Important code, explained line by line |
| — | Glossary |
| — | 23 cheat sheets |

---

# PART 1 — WHAT DOES THIS PROJECT ACTUALLY DO?

## 1.1 The situation before this server exists

You are working on one codebase. On Monday you talk to Claude Desktop and settle a design
question: the background job queue will run on Redis. On Tuesday you open Cursor to write
some code.

Cursor did not participate in Monday's Claude Desktop conversation and does not automatically
know the information discussed there. There is no shared place where that decision was written
down in a form either tool can read. If you want Cursor to know it, you type it again.

Six months later the team moves the queue to PostgreSQL. Now there is a worse problem: two
statements exist, both were true at some point, and only one is true now. If they were both
written down somewhere searchable, a search for "queue" returns both, and nothing in that
storage knows which one is current.

## 1.2 Why don't Claude Desktop and Cursor automatically share project knowledge?

Because they are separate applications, each holding its own conversation transcripts in its
own storage. There is no channel between them. Neither one publishes "here is what the user
decided" to the other.

A precise way to say this, and the phrasing to use in an interview:

> Cursor did not participate in the earlier Claude Desktop interaction and does not
> automatically know the information discussed there. If useful project knowledge was
> explicitly persisted through the MCP Shared Memory Server, Cursor can later retrieve that
> persisted knowledge from the shared backend.

Note what that sentence does **not** say. It does not say the tools have no memory of their
own. It does not say every AI tool starts from zero every time. It says something narrower and
verifiable: one tool does not automatically see another tool's conversation, and this server is
a place where knowledge can be deliberately put so that both can reach it.

## 1.3 What the server adds

One shared, durable place where project knowledge lives, reachable by any MCP client, with
correctness rules a plain notes file does not have:

1. **Shared.** Several client programs read and write the same store.
2. **Durable.** The knowledge survives the client closing, the server process exiting, and the
   machine rebooting, as long as the database's storage volume is kept.
3. **Versioned.** Changing a memory creates a new numbered version; the old text is kept.
4. **Conflict-safe.** Two clients changing the same memory at the same time cannot silently
   overwrite each other.
5. **Truth-maintaining.** When one piece of knowledge replaces another, the replaced one stops
   appearing in searches immediately, but stays fully readable in the history.
6. **Searchable two ways.** By exact words, and by meaning.
7. **Budget-aware.** It can return "the most useful summary that fits in N tokens" rather than
   just a list of matches.

## 1.4 What exactly is persisted

Verified from `src/memhub/persistence/models.py` and the six migrations in
`migrations/versions/`.

Stored:

- The **text** of each memory (`memory_revisions.content`), at most 8192 characters.
- A **type**: one of `DECISION`, `CONSTRAINT`, `FACT`, `TASK` (`memories.type`).
- A **status**: `ACTIVE`, `SUPERSEDED`, or `DELETED` (`memories.status`).
- A **revision number**, and every past revision's full text (`memory_revisions`).
- **Tags** (up to 16), an **importance** number 0–100, and an optional **expiry timestamp**.
- **Provenance**: which client wrote it (`author_client`), whether a human confirmed it
  (`author_kind`), and a free-text `source` string.
- **Project identity**: which namespace the memory belongs to.
- **Supersession links**: which memory replaced which (`memories.superseded_by_id`).
- **A content hash** used for deduplication (`memory_revisions.content_hash`, plus the
  `memory_dedup_keys` table).
- **Attestations**: which distinct clients have asserted the same fact.
- **Idempotency keys**: request identifiers and the stored response for each.
- **Audit events**: what action happened, its outcome, and who did it — never the content.
- **Embeddings**: a 384-number vector per revision per model, plus a job-queue row for
  generating it.

Not stored:

- Conversation transcripts.
- Chat history of any kind.
- Source code files, editor buffers, or the contents of your repository.
- The model's internal reasoning.
- Anything the client did not explicitly pass as a tool argument.

## 1.5 Who decides what should be remembered?

The client — meaning the AI model inside Claude Desktop or Cursor, guided by the user and by the
tool descriptions this server publishes. Nothing is captured automatically.

The server ships instructions that steer that decision, in `src/memhub/mcp/server.py`:

```python
SERVER_INSTRUCTIONS = """\
Shared, versioned project memory for MCP clients.

This server stores knowledge you explicitly record. It cannot see conversations \
- it only ever receives the arguments of the tool calls you make to it.

Record a memory when the user establishes something durable: an architectural \
decision and what it rules out, a constraint the project must respect, a \
load-bearing fact, or the piece of work currently in progress. ...
"""
```

## 1.6 Does the server receive full chat history?

No. An MCP server receives the arguments of the tool calls made to it, and nothing else.

Two honest refinements you should be able to state:

1. The server has no independent access to a transcript. There is no code path in this
   repository that reads a conversation.
2. A client *could* choose to paste conversation text into the `content` argument of
   `memory_remember`. That would be the client's decision and it would be visible in the call.
   The server never obtains conversation text on its own.

A protocol test asserts the server's own instructions do not overclaim:
`tests/protocol/test_mcp_tools.py::test_server_instructions_do_not_overclaim`.

## 1.7 What happens when Claude Desktop closes?

The memhub server process that Claude Desktop started exits with it. That process held no
authoritative state — no cache and no in-memory index that mattered. Everything it wrote was
already committed to PostgreSQL, which is a separate program running in its own container.

`src/memhub/mcp/__main__.py` states this directly:

> "this process holds no authoritative state of its own: a restart is a non-event."

## 1.8 How can Cursor later retrieve something stored through Claude Desktop?

Because both clients start their own copy of the same server program, and both copies are
configured to connect to the same PostgreSQL database. The knowledge is in the database, not in
either process.

There is a test that proves exactly this over the real transport, in
`tests/protocol/test_stdio_transport.py`:

```
test_two_sessions_share_state_over_stdio
  1. Start server process #1 as a subprocess. Call project_use, then memory_remember.
  2. Exit that process completely.
  3. Start server process #2. Call project_use with the same slug, then memory_search.
  4. Assert the memory comes back, with author_client == "claude-desktop" preserved.
```

**Diagram 2 — Claude → shared memory → Cursor**

```
   Monday                                       Tuesday
   ------                                       -------
 Claude Desktop                                  Cursor
      |                                            |
      | starts its own                             | starts its own
      v                                            v
 memhub process #1                          memhub process #2
      |                                            ^
      | INSERT                                     | SELECT
      v                                            |
   +---------------------------------------------------+
   |                    PostgreSQL                     |
   |      memories / memory_revisions / ...            |
   +---------------------------------------------------+

   Process #1 has fully exited before process #2 starts.
   Nothing is shared between them except the database.
```

---

# PART 2 — THE PROJECT NAME, WORD BY WORD

## 2.1 "Server"

**Simple definition.** A program that waits for requests from other programs and answers them.

**General meaning.** The counterpart of a *client*. The client asks; the server answers. This is
about roles, not machines: a server can run on your laptop.

**Meaning here.** `memhub-server` is a Python program. It is started by an MCP client, reads
requests from its standard input, and writes answers to its standard output. Entry point:
`src/memhub/mcp/__main__.py`.

**Why necessary.** The knowledge has to live somewhere both clients can reach, behind an
interface both understand.

**Without it.** Each client would need custom code for talking to your database, and there is no
way to add that code to Claude Desktop or Cursor.

## 2.2 "Memory"

**Simple definition.** Information kept for later.

**General meaning.** In everyday computing, "memory" often means RAM — fast, volatile storage
that disappears when a program stops. That is **not** the meaning here.

**Meaning here.** One durable, self-contained statement about a project, recorded deliberately.
It is a row in the `memories` table plus at least one row in `memory_revisions`. Example:
`"The job queue runs on Redis."`

**Why necessary.** Without a unit of knowledge that has an identity, you cannot version it,
retire it, or point at it. "One sentence with an ID" is what makes revision and supersession
possible.

**Without it.** You would have a text blob. You could search it, but you could not say "this
particular statement is no longer true" in any machine-checkable way.

## 2.3 "Shared"

**Simple definition.** Used by more than one party.

**General meaning.** Two programs both read and write the same data.

**Meaning here.** Two or more MCP clients — each with its own server process — read and write
the same PostgreSQL database. Verified by `test_two_sessions_share_state_over_stdio` and by
`test_a_second_client_sees_the_first_clients_memory` in `tests/protocol/test_mcp_tools.py`.

**Why necessary.** This word is what creates all the hard engineering. With one writer you would
need no compare-and-set, no idempotency protocol, no cross-client deduplication, and no
attestation table.

**Without it.** A single-client memory is a text file. That project is easy and uninteresting.

## 2.4 "MCP" — the Model Context Protocol

This deserves its own section because it is the word interviewers probe hardest.

### 2.4.1 Protocol

**Definition.** An agreed set of rules for how two programs exchange messages: what a message
looks like, what fields it has, what a reply looks like, and what the errors mean.

Analogy: HTTP is a protocol. A browser and a web server agree on `GET /page` and status code
`200`. Neither has to know how the other is built.

### 2.4.2 Model Context Protocol

**Definition.** An open protocol that lets an AI application discover and call capabilities
exposed by a separate program. The AI application is the **client**; the separate program is the
**server**. The server advertises *tools* (things the model can call), *resources* (things that
can be read by address), and *prompts*.

The point is standardisation. Without it, every AI application would need custom integration
code for every backend.

### 2.4.3 Client

**Definition.** The program that starts the conversation and makes requests. Here: Claude
Desktop, Cursor, or the test harness `mcp.Client`.

The client also decides *when* to call a tool. That decision is made by the language model
inside the client, based on the user's request and on the tool descriptions.

### 2.4.4 Tool

**Definition.** A named operation the server offers, with a declared input schema and, here, a
declared output schema. The model calls it with arguments it composes.

This server offers exactly seven, verified in `tests/protocol/manifest.json` and asserted in
`tests/protocol/test_stdio_transport.py`:

```
project_use   memory_remember   memory_revise   memory_forget
memory_search   memory_history   memory_context
```

### 2.4.5 Tool call and tool arguments

**Tool call.** One request naming a tool and supplying arguments.

**Tool arguments.** The values passed. **This is the entirety of what the server sees.**

```json
{
  "name": "memory_remember",
  "arguments": {
    "project_id": "3f2c...",
    "type": "DECISION",
    "content": "The job queue runs on Redis.",
    "client": "claude-desktop"
  }
}
```

That JSON object is the whole input. No transcript accompanies it.

### 2.4.6 Transport

**Definition.** The physical channel the protocol messages travel over. The protocol says what
the messages mean; the transport says how the bytes get there.

### 2.4.7 stdio

**Definition.** Standard input and standard output — the two byte streams every process has.

**Meaning here.** The MCP client starts the server as a **subprocess** and talks to it by writing
to its stdin and reading its stdout.

**The consequence that shapes this whole project:** each client starts its *own* subprocess. Two
clients means two operating-system processes with no shared memory.

**The trap this creates:** stdout is the protocol channel. One stray `print()` corrupts the
message stream and the client fails with a confusing parse error. This repository defends
against that three ways:

- All logs go to stderr (`src/memhub/observability/logging.py`).
- The `ruff` rule `T20` bans `print()` in the package (`pyproject.toml`).
- A test spawns the real subprocess and asserts a clean handshake:
  `test_stdout_carries_only_protocol_traffic`.

### 2.4.8 Subprocess

**Definition.** A process started by another process. Claude Desktop is the parent; the memhub
server is the child.

### 2.4.9 JSON-RPC

**Definition.** A simple message format for remote procedure calls, encoded as JSON. A request
carries a method name, parameters, and an id. A response carries a result or an error, with the
same id so the caller can match them. MCP uses JSON-RPC 2.0 as its message format.

### 2.4.10 The sentence to say out loud in an interview

> **MCP is the interface. It is not the memory engine.**

MCP gives the model a standard way to *reach* this system. Everything that makes the system
correct — versioning, compare-and-set, deduplication, idempotency, structural exclusion of
retired knowledge, hybrid retrieval, the token budget — lives underneath, in the service,
persistence, and retrieval layers, and in PostgreSQL itself.

The repository's MCP layer is deliberately thin. Every handler in `src/memhub/mcp/server.py`
does the same three things: parse the arguments, call one service function, map the result into
an output schema. There is no SQL and no business logic in that file.

---

# PART 3 — ARCHITECTURE

## 3.1 The process picture, verified

Questions first, answers from the code:

**Does Claude Desktop start its own memhub process?** Yes. The client configuration in
`docs/clients.md` gives Claude Desktop a `command` to run. Claude Desktop launches that command
as a subprocess.

**Does Cursor start its own memhub process?** Yes, the same way, from its own `mcp.json`.

**Do the two processes communicate directly?** No. There is no code in this repository that
opens a socket, pipe, or channel between two memhub processes. Grep for it; it does not exist.

**Do they share Python memory?** No. They are separate operating-system processes. A Python
object in one is invisible to the other. `src/memhub/mcp/__main__.py` says: "Each MCP host —
Claude Desktop, Cursor — spawns its *own* copy of this process. The two share no memory and no
cache."

**Where does the shared state actually live?** In PostgreSQL. That is the only thing both
processes touch.

**Diagram 1 — overall architecture**

```
  Claude Desktop                                Cursor
        |                                          |
        | MCP over stdio (JSON-RPC on stdin/stdout)|
        v                                          v
  +------------------+                    +------------------+
  | memhub process 1 |                    | memhub process 2 |
  |   (stateless)    |                    |   (stateless)    |
  +------------------+                    +------------------+
        |                                          |
        |  asyncpg connection pool                 |
        +--------------------+---------------------+
                             v
                  +----------------------+
                  |     PostgreSQL 16    |   <-- the ONLY shared state
                  |     + pgvector       |       and the only place two
                  +----------------------+       writers can be adjudicated
                             ^
                             | SELECT ... FOR UPDATE SKIP LOCKED
                  +----------------------+
                  | embedding worker     |  one asyncio task inside each
                  | (in-process task)    |  server process
                  +----------------------+
```

## 3.2 Why PostgreSQL becomes the shared state

Because it is the only component that can see both writers.

An in-process Python lock (`asyncio.Lock`) protects nothing here — the other writer is a
different operating-system process and never sees that lock. A cache in one process is invisible
to the other. Only the database sits underneath both.

This is why the concurrency work in this project is real engineering and not decoration. The
transport (stdio, one process per client) *forces* it.

## 3.3 The layered picture

**Diagram 3 — MCP request flow**

```
 MCP handler        src/memhub/mcp/server.py
      |             validate arguments, call one service, map the result
      v
 Service            src/memhub/services/*.py
      |             transactions, invariants, idempotency, dedup, orchestration
      v
 Repository         src/memhub/persistence/repositories/*.py
      |             SQL and ORM, every method scoped to a project
      v
 PostgreSQL         durability, isolation, uniqueness, referential integrity
```

Three supporting groups sit beside that stack:

```
 domain/          pure types, enums, errors, validation, normalisation — no I/O
 retrieval/       the stage-0 filter, lexical, semantic, fusion, ranking — read only
 context/         token estimation, selection under a budget, rendering
 embeddings/      the embedding port, adapters, and the background worker
 cli/             operator commands, deliberately not reachable over MCP
 observability/   JSON logs to stderr, in-process metrics registry
```

## 3.4 Every layer: what, why, input, output, files, callers, dependencies

### domain/

- **What.** Pure types and rules. Enums, error classes, frozen result dataclasses, input
  validation, content normalisation and hashing, per-type policy.
- **Why.** So the rules of the system can be tested with no database and no protocol. It also
  keeps the vocabulary in one place.
- **Input.** Plain Python values.
- **Output.** Plain Python values, or a raised `MemhubError`.
- **Files.** `enums.py`, `errors.py`, `models.py`, `normalize.py`, `policy.py`, `validation.py`.
- **Callers.** Services, MCP layer, persistence.
- **Dependencies.** None inside this package. `domain/` imports from no other memhub package.

### services/

- **What.** The business logic. One transaction per operation, and all orchestration:
  idempotency claim, deduplication, supersession, audit, metrics, enqueueing embedding work.
- **Why.** So the same logic can be reached from MCP today and from an HTTP layer later without
  rewriting anything.
- **Input.** An `AsyncSession` (an open database transaction), a project id, and typed arguments.
- **Output.** Frozen domain dataclasses (`RememberResult`, `ReviseResult`, `SearchResult`, …).
- **Files.** `memories.py`, `projects.py`, `idempotency.py`, `retrieval.py`, `context.py`.
- **Callers.** `mcp/server.py`, the CLI, and most tests.
- **Dependencies.** `domain/`, `persistence/`, `retrieval/`, `context/`, `observability/`.
- **Must never.** Know that MCP exists. Confirmed: no service module imports from `memhub.mcp`.

### persistence/

- **What.** Database access. ORM models that carry the schema, repositories that run queries, an
  async engine and session factory, six hand-written `.sql` files for the statements whose
  correctness depends on exact PostgreSQL semantics, and a startup schema check.
- **Why.** So SQL lives in one place and the invariants live in the schema.
- **Input.** A session plus scoped arguments.
- **Output.** ORM rows, or scalar results.
- **Files.** `models.py`, `engine.py`, `base.py`, `schema.py`, `sqlstate.py`,
  `repositories/{projects,memories,truth,audit}.py`, `sql/*.sql`.
- **Callers.** Services only.
- **Dependencies.** `domain/`, `retrieval/filters.py`.

### retrieval/

- **What.** Reading. The stage-0 filter (`filters.py`), full-text matching and scoring
  (`lexical.py`), vector search (`semantic.py`), rank fusion (`fusion.py`), and ranking priors
  (`ranking.py`).
- **Why.** So there is exactly one definition of "which memories are visible", written once and
  composed by every retrieval path.
- **Input.** A project id and a query.
- **Output.** SQLAlchemy `Select` objects, SQL expressions, or lists of ids and scores.
- **Callers.** `persistence/repositories/memories.py` and `services/retrieval.py`.
- **Must never.** Mutate anything. Confirmed: no `INSERT`, `UPDATE`, or `DELETE` in this package.

### embeddings/

- **What.** A `Protocol` describing an embedder, a local adapter, a deterministic fake, a factory,
  and the outbox worker.
- **Why.** So the storage layer never learns which model produced a vector, and so a failing
  embedder cannot fail a write.
- **Input.** A batch of strings.
- **Output.** One unit-normalised list of 384 floats per string.
- **Files.** `base.py`, `local.py`, `fake.py`, `factory.py`, `worker.py`.

### context/

- **What.** Token estimation (`tokens.py`), constrained selection (`builder.py`), and markdown
  rendering (`render.py`).
- **Why.** `memory_context` is a selection problem, not a search problem.
- **Input.** A list of candidates with scores and token costs, plus a budget.
- **Output.** A `Selection` and a rendered brief.

### mcp/

- **What.** The protocol boundary. Tool and resource registration, output schemas, error mapping,
  and the stdio entry point.
- **Why.** To keep the protocol details in one place and everything else testable without it.
- **Files.** `server.py`, `schemas.py`, `mapping.py`, `__main__.py`.

### cli/

- **What.** `memhub-admin` with three subcommands: `purge`, `gc`, `status`.
- **Why.** `purge` destroys content irreversibly. That does not belong in a language model's tool
  surface.
- **File.** `cli/admin.py`.

### observability/

- **What.** JSON structured logging to stderr, and a small in-process metrics registry that
  refuses unbounded label values.
- **Why.** You cannot debug a 50-way concurrency test without them.
- **Files.** `logging.py`, `metrics.py`.

## 3.5 What PostgreSQL, pgvector, and MCP are each responsible for

**PostgreSQL is responsible for:**

- Durability — data survives a process exit or a crash.
- Atomicity — a group of statements either all take effect or none do.
- Isolation — deciding what concurrent transactions see of each other.
- Uniqueness — primary keys and unique indexes.
- Referential integrity — foreign keys.
- Serialising conflicting writers — the row lock in the compare-and-set.
- Full-text search — `tsvector`, `websearch_to_tsquery`, `ts_rank_cd`, the GIN index.
- The clock — every stored timestamp comes from `now()` on the server.
- Being the job queue — `SELECT ... FOR UPDATE SKIP LOCKED`.

**pgvector is responsible for:**

- Storing a fixed-width vector as a column type (`vector(384)`).
- Computing cosine distance between vectors (the `<=>` operator).
- Approximate nearest-neighbour search through an HNSW index.

pgvector is a PostgreSQL **extension**: it adds a data type, operators, and index types to
PostgreSQL itself. It is not a separate database. That is the whole reason a superseded memory
cannot be returned by a vector search here — the vector search runs inside the same query, under
the same filter, as everything else.

**MCP is responsible for:**

- Advertising the seven tools and three resources, with their schemas and descriptions.
- Carrying tool calls in and results out, over stdio, as JSON-RPC.
- Distinguishing a protocol error from a tool error.

**MCP is NOT responsible for:**

- Storage, versioning, or concurrency control.
- Ranking, retrieval, or the token budget.
- Deciding what is true.
- Sending conversations anywhere.

---

# PART 4 — EVERY IMPORTANT WORD, DEFINED

Read this part once now, and come back to it whenever a word in a later part is unfamiliar. The
most important twelve terms get the full treatment in their own parts later; this is the
dictionary.

## 4.1 Persistence and state

**Persistent** — surviving beyond the program that created it. A memory written by Claude
Desktop is persistent because it is still there after Claude Desktop closes.

**Persistence** — as a noun in this codebase, the layer that talks to the database
(`src/memhub/persistence/`).

**Durable** — written in a way that survives a crash. PostgreSQL achieves this by writing
changes to a write-ahead log on disk before reporting a commit as successful.

**State** — data that changes over time and that later behaviour depends on. `memories.status`
is state; the text of a revision is not, because it never changes.

**Stateful** — holds state between requests. PostgreSQL is stateful.

**Stateless** — holds no state between requests that matters. The memhub server process is
stateless: kill it and restart it, and nothing is lost.

**Source of truth** — the one place that decides the answer when copies disagree. Here it is
PostgreSQL, always.

## 4.2 Processes and protocols

**Client** — the program making requests.
**Server** — the program answering them.
**Process** — one running program, with its own memory space, managed by the operating system.
**Subprocess** — a process started by another process.
**Protocol** — the agreed rules for the messages.
**Transport** — the channel the messages travel on.
**stdio** — standard input and standard output as that channel.
**JSON-RPC** — the JSON message format MCP uses: method, params, id; result or error.

## 4.3 Projects and scope

**Project** — a namespace for memories. One row in `projects`. Identified by a server-issued
UUID, with a human-readable `slug` as a unique secondary key.

**Namespace** — a boundary that keeps names or data separate. Two projects can both hold a
memory about "the queue" without those memories mixing.

**Project isolation** — the guarantee that a memory in project A never appears while operating
in project B. Enforced three times over here: by required arguments in Python, by a `WHERE`
clause in every query, and by a composite foreign key in the schema.

**Provenance** — where a piece of information came from. Here: `author_client`, `author_kind`,
`source`, and the attestation rows.

## 4.4 The memory lifecycle

**Memory** — one logical statement, with an identity. Row in `memories`.

**Revision** — one version of that statement's text. Row in `memory_revisions`, keyed by
`(memory_id, revision_no)`.

**Version** — the same idea as revision, in ordinary language.

**Immutable** — never changed after it is written. Revision rows are immutable in their content:
the only column ever updated on an existing revision row is `is_current`, the flag that says
which revision is the live one.

**Append-only** — new rows are added; existing rows are not rewritten or removed. That is how
`memory_revisions` is used.

**Supersession** — one memory retiring a *different* memory, because the knowledge itself was
replaced.

**ACTIVE** — the memory is current and appears in retrieval.

**SUPERSEDED** — the memory was retired by another memory. `memories.superseded_by_id` points at
the winner. Never appears in normal retrieval; fully readable through `memory_history`.

**DELETED** — the memory was tombstoned by `memory_forget`. Never appears in normal retrieval;
fully readable through `memory_history`.

**EXPIRED** — **not a stored status in this system.** Expiry is computed at read time as
`expires_at IS NULL OR expires_at > now()`. A stored `EXPIRED` value would need a background
sweeper to be true, and between expiry and sweep the column would be lying. Verified in
`src/memhub/domain/enums.py` (only three statuses) and `src/memhub/retrieval/filters.py` (the
derived predicate).

**Tombstone** — a marker that says "this is gone" while the data is still physically present. A
`memory_forget` sets `status='DELETED'` and `deleted_at`; it deletes nothing.

## 4.5 Databases

**Invariant** — a rule that must always be true. Example: "a memory has exactly one current
revision."

**Transaction** — a group of statements treated as one unit: all of them take effect, or none of
them do.

**Atomic** — all-or-nothing. A transaction is atomic.

**Commit** — make a transaction's changes permanent and visible to others.

**Rollback** — discard a transaction's changes as if it never ran.

**Constraint** — a rule the database itself enforces on the data. If you try to write data that
breaks it, the write fails.

**Foreign key** — a constraint saying a column's value must exist as a key in another table.

**Composite foreign key** — a foreign key on more than one column at once. Here,
`(superseded_by_id, project_id)` referencing `(id, project_id)` — which is what makes
cross-project supersession impossible to write down at all.

**Unique constraint / unique index** — a rule that no two rows may share a value.

**Partial index** — an index that covers only the rows matching a condition, for example
`... WHERE is_current`. Smaller, faster to maintain, and usable only when the planner can prove
your query's condition implies the index's condition.

**Primary key** — the unique identifier for a row.

## 4.6 Concurrency

**Concurrency** — more than one thing happening in overlapping time.

**Race condition** — a bug where the result depends on the timing of concurrent operations.

**Lost update** — a specific race: two writers read the same value, both write based on it, and
the second silently erases the first's change.

**Stale client** — a client acting on a value that has since changed.

**Optimistic concurrency** — assume conflicts are rare. Do not lock in advance. Detect the
conflict at write time and refuse.

**Pessimistic locking** — assume conflicts are likely. Take a lock first, make everyone else wait.

**Compare-and-set (CAS)** — write only if the current value still equals the value you read.

**expected_revision** — the caller's claim about what revision it read.

**current_revision** — what the database actually holds now.

**READ COMMITTED** — PostgreSQL's default isolation level. Each statement sees only data
committed before that statement started.

**SERIALIZABLE** — the strictest isolation level. Concurrent transactions are guaranteed to
produce a result equivalent to running them one after another; conflicts raise error `40001` and
must be retried.

**MVCC (Multi-Version Concurrency Control)** — how PostgreSQL implements isolation: an update
creates a *new version* of a row rather than overwriting the old one, and each transaction is
shown the version appropriate to its snapshot. Readers never block writers and writers never
block readers.

**EvalPlanQual** — PostgreSQL's behaviour under READ COMMITTED when an `UPDATE` was blocked by
another transaction that then committed: instead of proceeding with the row version it first
found, it walks to the newest committed version and re-evaluates the `WHERE` clause against it.
This is the mechanism that makes this project's compare-and-set correct.

## 4.7 Retries and duplicates

**Idempotent** — doing it twice has the same effect as doing it once.

**Idempotency** — the property, and the mechanism that provides it.

**Retry** — sending the same request again after not receiving an answer.

**client_request_id** — the caller-generated string that identifies one logical request, 8 to
128 characters. It is the idempotency key.

**Duplicate** — a second copy of something that should exist once.

**Deduplication** — recognising that two *independently submitted* pieces of knowledge are the
same, and not storing both.

**Normalisation** — reducing text to a comparison form so that trivially different spellings
compare equal. Here: NFKC Unicode normalisation, casefold, collapse whitespace, strip trailing
punctuation (`src/memhub/domain/normalize.py`).

**Content hash** — a fixed-size fingerprint of the normalised text. Here SHA-256, 32 bytes.

**Attestation** — a record that a particular client asserted a particular fact. Row in
`memory_attestations`.

**Corroboration** — two or more distinct clients independently asserting the same fact. Counted
as `COUNT(DISTINCT client_name)`.

## 4.8 Retrieval

**Retrieval** — finding the memories worth returning for a request.

**Keyword search / lexical search / full-text search** — matching on the words themselves.

**Token (text)** — in full-text search, one indexed word unit. PostgreSQL calls the stemmed form
a *lexeme*.

**Tokenization** — splitting text into those units.

**tsvector** — PostgreSQL's data type holding a document's processed lexemes and their positions.

**GIN index** — Generalised Inverted Index. An index type suited to columns holding many values
per row, such as a `tsvector` or an array. It maps each value to the rows containing it.

**Ranking** — ordering results by how good they are.

**Candidate** — a row that survived filtering and is eligible for ranking.

**Score** — the number used for ordering.

**Semantic search** — matching on meaning rather than on exact words.

**Embedding** — a list of numbers produced by a model, representing a piece of text's meaning.

**Vector** — that list of numbers. Here, 384 of them.

**Dimension** — how many numbers are in the vector. Here 384, fixed by the column type.

**pgvector** — the PostgreSQL extension that adds the `vector` type, distance operators, and
vector index types.

**Cosine similarity** — a measure of how similar two vectors' directions are, from -1 to 1.

**Cosine distance** — `1 - cosine similarity`, from 0 to 2. Smaller means more similar. The
`<=>` operator in pgvector.

**Nearest neighbour** — the stored vector closest to a query vector.

**Approximate nearest neighbour (ANN)** — a search that finds *probably* the closest vectors,
much faster than checking every one. It trades a small amount of accuracy for a large amount of
speed.

**HNSW (Hierarchical Navigable Small World)** — a graph-based ANN index. It builds layers of
links between vectors and walks the graph from a coarse layer to a fine one. Fast, and does not
require the whole dataset up front.

**Reciprocal Rank Fusion (RRF)** — combining two or more ranked lists using positions rather
than scores.

**Stale memory** — a memory whose content was true once and is not current now.

**Candidate filtering** — removing rows from the candidate set before ranking runs.

## 4.9 Context and budgets

**Token (model)** — the unit a language model reads text in. Roughly a word fragment. Different
models tokenise the same sentence into different counts.

**Token budget** — a limit on how many tokens a response may cost.

**Context window** — the total amount of text a model can hold at once. A budget exists so a
brief does not consume it.

**MMR (Maximal Marginal Relevance)** — a selection method that balances "how good is this item"
against "how different is it from what I already picked".

**Diversity** — the property MMR is protecting: not filling a budget with three restatements of
the same thing.

**Knapsack problem** — the classic problem of choosing items with weights and values to maximise
value under a weight limit. Selecting memories under a token budget is a knapsack problem.

**Greedy selection** — repeatedly taking the currently best-looking option. Not guaranteed
optimal, but fast and, for the ratio-based version, provably close.

## 4.10 Background work

**Background worker** — code that runs separately from request handling, doing deferred work.

**Job** — one unit of that deferred work. Row in `embedding_jobs`.

**Queue** — the collection of pending jobs.

**Outbox** — a table where a job row is written *in the same transaction* as the change it
relates to.

**Transactional outbox** — the pattern that guarantees "the change and its follow-up job either
both exist or neither does". It removes the dual-write bug.

**Dual write** — writing to two systems without a transaction spanning them, so a crash between
them leaves the systems disagreeing.

**Row lock** — a lock taken on one row, held until the transaction ends.

**FOR UPDATE** — a SQL clause that takes an exclusive row lock on the rows a `SELECT` returns.

**SKIP LOCKED** — a modifier telling PostgreSQL to skip over rows another transaction has locked
instead of waiting for them. This is what turns a table into a work queue several workers can
drain at once.

**FOR SHARE** — a shared row lock. Several readers may hold it; it blocks writers. Used here to
wait for an in-flight idempotency claim to finish.

## 4.11 Connections and failures

**Connection pool** — a set of already-open database connections that are reused, because opening
one is expensive. Here: 10, plus 5 overflow (`src/memhub/config.py`).

**Timeout** — a limit on how long to wait. Two matter here: `pool_timeout` (2 seconds to get a
connection) and `statement_timeout` (5000 milliseconds for one SQL statement, enforced by
PostgreSQL itself).

**Retryable failure** — one where nothing happened, so trying again is safe.

**Ambiguous failure** — one where you cannot tell whether it happened.

**UNKNOWN_OUTCOME** — this project's name for that ambiguous case: the connection died while a
statement was in flight, so the write may or may not have committed.

## 4.12 Schema and tooling

**Schema** — the structure of the database: tables, columns, types, constraints, indexes.

**Migration** — a versioned script that changes the schema from one state to the next.

**Alembic** — the migration tool used here. Six migrations, `0001` through `0006`.

**SQLAlchemy** — the Python library used for defining models and building queries.

**asyncpg** — the low-level PostgreSQL driver, used asynchronously.

## 4.13 Testing

**Unit test** — tests one piece in isolation, with no database.

**Integration test** — tests several pieces together against real PostgreSQL.

**Concurrency test** — runs several writers at the same time, against real PostgreSQL, on
genuinely separate connections.

**Protocol test** — drives the MCP interface and checks the contract.

**Failure test** — deliberately causes a failure and checks the response.

**Performance test** — measures latency and how cost grows with corpus size.

**Mutation testing** — deliberately breaking a mechanism to confirm that a test notices.

## 4.14 Measurement

**Precision@k** — of the top k results, what fraction are relevant.

**Recall@k** — of all the relevant memories, what fraction appear in the top k.

**nDCG@k** — Normalised Discounted Cumulative Gain. A ranking quality measure that rewards
putting highly relevant items near the top, using graded relevance rather than yes/no.

**MRR** — Mean Reciprocal Rank. The average of `1 / position of the first relevant result`.

**Benchmark** — a repeatable measurement.

**Latency** — how long something takes.

**Query plan** — PostgreSQL's chosen strategy for executing a query.

**Sequential scan** — reading the table row by row.

**Index scan** — using an index to jump to the relevant rows.

---

# PART 5 — THE ACTUAL MCP TOOLS

## 5.0 Verification first

The seven tool names were verified three ways:

1. The `@server.tool(name=...)` decorators in `src/memhub/mcp/server.py`.
2. The committed snapshot `tests/protocol/manifest.json`.
3. `tests/protocol/test_stdio_transport.py::test_stdout_carries_only_protocol_traffic`, which
   asserts the exact set returned by `tools/list` over a real subprocess.

All three agree:

```
project_use   memory_remember   memory_revise   memory_forget
memory_search   memory_history   memory_context
```

There is no `memory_supersede` tool. Superseding is an argument to `memory_remember`, because
retiring the old fact and asserting the new one must happen in one transaction. There is no
purge or hard-delete tool; that is the operator CLI only.

Three resources also exist (read-only, addressed by URI):

```
memory://projects                         (static)
memory://memories/{memory_id}             (template)
memory://memories/{memory_id}/history     (template)
```

The generic shape of every tool call:

```
Client
  |  tool call with arguments
  v
MCP handler   (src/memhub/mcp/server.py)  — parse ids, call one service
  v
Service       (src/memhub/services/*.py)  — one transaction, all the logic
  v
Repository    (src/memhub/persistence/…)  — SQL
  v
PostgreSQL
  v
Response      (src/memhub/mcp/schemas.py) — a Pydantic output model
```

Every handler is wrapped in `@domain_errors` (`src/memhub/mcp/mapping.py`) and runs inside
`async with session_scope(session_factory)`, which is the transaction boundary: commit on
success, roll back on any exception.

---

## 5.1 `project_use`

**Purpose.** Resolve which project namespace you are working in, and return its canonical UUID.
Optionally create it.

**Input.** `slug`, `project_id`, `git_remote`, `workspace_path`, `display_name`, `create`
(default `false`). All optional individually, but at least one identifier is required.

**Output.** `ProjectOut`: `project_id`, `slug`, `display_name`, `created` (boolean).

**Validation.** `project_id` must parse as a UUID. `slug` must match
`^[a-z0-9][a-z0-9._-]{0,62}[a-z0-9]$` — enforced by `validate_slug` in Python *and* by a `CHECK`
constraint in the schema.

**Handler.** `project_use` in `src/memhub/mcp/server.py`.
**Service.** `memhub.services.projects.use_project`.
**Repository.** `ProjectRepository` (`get_by_id`, `get_by_slug`, `resolve_alias`, `create`,
`add_alias`).

**Database changes.** Read-only unless creating, in which case it inserts one `projects` row and
possibly one or two `project_aliases` rows.

**The important rule.** Resolution never guesses and never auto-creates.

Every hint supplied is resolved independently, and the results must agree. If two hints point at
different projects, you get `AMBIGUOUS_PROJECT` listing both. If nothing matches and
`create=false`, you get `PROJECT_NOT_FOUND` telling you to pass `create=true`.

Why this matters: a client opened in the wrong directory would otherwise silently start a second,
empty memory namespace and split the project's knowledge in half. Both halves would look healthy
and nothing would surface the problem.

**Failure cases.** `PROJECT_NOT_FOUND`, `AMBIGUOUS_PROJECT`, `PROJECT_EXISTS`,
`VALIDATION_FAILED`.

**Tests.** `tests/integration/test_projects.py` (12 tests), `test_creates_and_resolves` and
`test_missing_project_is_a_tool_error_not_a_protocol_error` in the protocol suite.

**Example.**

```json
→ {"slug": "queue-service", "create": true}
← {"project_id": "3f2c…", "slug": "queue-service",
   "display_name": "queue-service", "created": true}
```

---

## 5.2 `memory_remember`

**Purpose.** Record one durable piece of project knowledge. Optionally retire the memories it
replaces, in the same transaction.

**Input.** `project_id`, `type`, `content` (required); `tags`, `importance`, `supersedes`,
`source`, `client`, `human_confirmed`, `client_request_id` (optional).

**Output.** `RememberOut`: `outcome` (`created` | `deduplicated` | `idempotent_replay`),
`memory`, `superseded`, `not_superseded`, `attestation_count`.

**Validation.** Content 1–8192 characters after stripping; up to 16 tags each matching a
lowercase pattern; importance 0–100; `type` one of the four; a `TASK` gets a default 7-day TTL
and may not exceed 30 days; `client_request_id` 8–128 characters.

**Handler.** `memory_remember`. **Service.** `memhub.services.memories.remember`.
**Repositories.** `MemoryRepository.create`, `TruthRepository.claim_dedup_key` / `attest` /
`supersede` / `enqueue_embedding`, `AuditRepository.record`.

**Database changes on the ordinary path.** One `memories` row, one `memory_revisions` row, one
`memory_dedup_keys` row, one `memory_attestations` row, one `audit_events` row, plus — if an
embedder is configured — one `embedding_jobs` row, plus one `idempotency_keys` row if a key was
supplied. Zero or more `memories` rows updated to `SUPERSEDED`, each with its own audit row.

**Transaction boundary.** One transaction for all of it, opened by `session_scope`. There is one
nested savepoint inside it, around the speculative insert used for deduplication.

**Failure cases.** `VALIDATION_FAILED`, `MEMORY_NOT_FOUND`, `IDEMPOTENCY_KEY_REUSED`, and the
four infrastructure codes.

**Tests.** `tests/integration/test_memories.py`, `tests/integration/test_stale_memory.py`,
`tests/concurrency/test_idempotency.py`, `tests/protocol/test_mcp_tools.py`.

**Example (the running example, step 1).**

```json
→ {"project_id": "3f2c…", "type": "DECISION",
   "content": "The job queue runs on Redis.",
   "tags": ["queue"], "client": "claude-desktop", "human_confirmed": true}

← {"outcome": "created",
   "memory": {"memory_id": "a1…", "revision_no": 1, "status": "ACTIVE",
              "author_client": "claude-desktop", "author_kind": "human_confirmed"},
   "superseded": [], "not_superseded": [], "attestation_count": 1}
```

**Example (the running example, step 4 — the replacement).**

```json
→ {"project_id": "3f2c…", "type": "DECISION",
   "content": "The job queue runs on PostgreSQL. Redis was removed.",
   "supersedes": ["a1…"], "client": "cursor"}

← {"outcome": "created", "memory": {"memory_id": "b2…", "revision_no": 1},
   "superseded": ["a1…"], "not_superseded": []}
```

---

## 5.3 `memory_revise`

**Purpose.** Change the text of a memory you have already read, without silently overwriting
anybody else's change.

**Input.** `project_id`, `memory_id`, `expected_revision`, `content` (required); `tags`,
`change_reason`, `client`, `client_request_id` (optional).

**Output.** `ReviseOut`: `outcome` (`revised` | `conflict` | `idempotent_replay`), `memory`,
`previous_revision`, `current_revision`, `expected_revision`, `guidance`.

**Validation.** `expected_revision >= 1` (declared in the schema with `ge=1`), content rules as
above.

**Handler.** `memory_revise`. **Service.** `memhub.services.memories.revise`.
**Repository.** `MemoryRepository.compare_and_set`, which runs
`src/memhub/persistence/sql/cas_revise.sql`.

**Database changes on success.** `memories.current_revision_no` incremented by one; the previous
revision's `is_current` set to `false`; a new `memory_revisions` row appended; an
`embedding_jobs` row if configured; an audit row.

**Database changes on conflict.** Only an audit row, with `outcome='conflict'`. Nothing else
changes.

**Failure cases.** A conflict is **not** a failure — it is a normal result with
`outcome="conflict"`. Real errors: `MEMORY_NOT_FOUND`, `VALIDATION_FAILED`,
`IDEMPOTENCY_KEY_REUSED`.

**Tests.** `tests/concurrency/test_compare_and_set.py` (5 tests, 50 writers each),
`test_conflict_is_a_result_not_an_error` in the protocol suite.

**Example (the running example, step 3 — a refinement, not a replacement).**

```json
→ {"project_id": "3f2c…", "memory_id": "a1…", "expected_revision": 1,
   "content": "The job queue runs on Redis, with dedicated worker processes.",
   "change_reason": "added detail after review"}

← {"outcome": "revised", "memory": {"revision_no": 2, …}, "previous_revision": 1}
```

**Example (a conflict).**

```json
→ {"memory_id": "a1…", "expected_revision": 1, "content": "…"}

← {"outcome": "conflict",
   "memory": {"revision_no": 2, "content": "…what actually won…",
              "author_client": "cursor"},
   "current_revision": 2, "expected_revision": 1,
   "guidance": "Another client changed this memory to revision 2 (by cursor)
                while you held revision 1. Read the content above, merge your
                change into it, and call memory_revise again with
                expected_revision=2. Do not simply resend your original text…"}
```

---

## 5.4 `memory_forget`

**Purpose.** Stop a memory appearing in search, without destroying it.

**Input.** `project_id`, `memory_id`, `reason`, `client`.
**Output.** `ForgetOut`: `outcome` (`forgotten` | `already_forgotten`), `memory_id`, `note`.

**Handler.** `memory_forget`. **Service.** `memhub.services.memories.forget`.
**Repository.** `TruthRepository.forget`, running `cas_forget.sql`.

**Database changes.** `memories.status='DELETED'`, `deleted_at=now()`; the memory's rows in
`memory_dedup_keys` are deleted so the same sentence can be asserted again later; an audit row.
**No content is deleted.**

**Failure cases.** `MEMORY_NOT_FOUND` if the memory is not in this project. Forgetting an
already-retired memory is not an error; it returns `already_forgotten`.

**Tests.** `TestForget` in `tests/integration/test_stale_memory.py` (4 tests),
`test_forget_hides_but_history_still_answers` in the protocol suite.

---

## 5.5 `memory_search`

**Purpose.** Retrieve the memories that are currently true and match a query.

**Input.** `project_id`, `query` (optional), `types`, `tags`, `limit` (1–100, default 10).
**Output.** `SearchOut`: `results`, `returned`, `total_matched`, `filtered_out`.

**Handler.** `memory_search`. It branches:

```python
if embedder is not None and query:
    result = await retrieval_service.hybrid_search(...)   # lexical + vector, RRF
else:
    result = await memory_service.search(...)             # lexical only
```

**Services.** `memhub.services.retrieval.hybrid_search` or `memhub.services.memories.search`.

**Database changes.** None. Reads only.

**Failure cases.** `VALIDATION_FAILED`, `DEADLINE_EXCEEDED` if the statement timeout fires.

**Tests.** `tests/integration/test_lexical_search.py` (14), `test_hybrid_search.py` (8),
`test_stale_memory.py`, `tests/eval/`.

**Example (the running example, step 5 — the point of the project).**

```json
→ {"project_id": "3f2c…", "query": "redis"}

← {"results": [{"content": "The job queue runs on PostgreSQL. Redis was removed.", …}],
   "returned": 1, "total_matched": 1,
   "filtered_out": "superseded, deleted and expired memories are excluded from
                    normal retrieval"}
```

The retired memory contains the word "Redis" twice and matches the query far better. It does not
appear, because it was removed from the candidate set before any ranking ran.

**Two discrepancies to know about this tool** (both verified):

1. Its published description still says *"The optional query is a substring match today;
   relevance ranking arrives in a later version."* That is stale. The implementation uses
   PostgreSQL full-text search with `ts_rank_cd` and ranking priors, and hybrid retrieval when an
   embedder is configured. The stale sentence is committed in `tests/protocol/manifest.json`, so
   the snapshot test passes — the snapshot is doing its job of pinning the description, but the
   description itself is out of date.
2. `SearchResult` (the domain object) carries `match_strategy`, `semantic_coverage`, and
   `degraded`. `SearchOut` (the MCP output schema) carries none of them. So a client calling
   `memory_search` is not told whether the search was widened, how much of the corpus was
   vector-indexed, or whether the semantic leg failed. `memory_context` *does* report
   `semantic_coverage` and `degraded`.

---

## 5.6 `memory_history`

**Purpose.** Show everything known about one memory, including memories that search deliberately
hides.

**Input.** `project_id`, `memory_id`.
**Output.** `HistoryOut`: `memory`, `status`, `revisions` (all of them, oldest first),
`superseded_by`, `supersedes`, `attestations`, `audit`.

**Handler.** `memory_history`. **Service.** `memhub.services.memories.history`.
**Repositories.** `MemoryRepository.get(include_retired=True)`, plus `TruthRepository.revisions`,
`superseded_by`, `supersedes`, `attestations`, `audit_trail`.

This is one of only two places in the codebase where `include_retired=True` is used. That is what
lets normal retrieval be unconditional about what it excludes.

**Database changes.** None.

**Tests.** `test_history_still_shows_the_retired_fact`,
`test_history_shows_the_full_revision_chain`.

**Example (the running example, step 5b).**

```json
→ {"project_id": "3f2c…", "memory_id": "a1…"}

← {"status": "SUPERSEDED",
   "memory": {"content": "The job queue runs on Redis.", …},
   "revisions": [{"revision_no": 1, "content": "The job queue runs on Redis.",
                  "is_current": true, "author_client": "claude-desktop"}],
   "superseded_by": {"memory_id": "b2…",
                     "content": "The job queue runs on PostgreSQL. Redis was removed.",
                     "at": "2026-…"},
   "supersedes": [], "attestations": [...], "audit": [...]}
```

---

## 5.7 `memory_context`

**Purpose.** Return the most useful project brief that fits inside a token budget. This is
selection under a constraint, not search.

**Input.** `project_id`, `query` (optional), `token_budget` (100–32000, default 2000).
**Output.** `ContextOut`: `brief` (markdown), `memories`, `budget` (a `BudgetOut` with
`requested`, `estimated_used`, `utilisation`, `estimator`, `considered`, `selected`, `dropped`),
`semantic_coverage`, `degraded`.

**Handler.** `memory_context`. **Service.** `memhub.services.context.build_context`, which calls
`memhub.context.builder.select` and `memhub.context.render.render_brief`.

**Database changes.** None.

**Failure cases.** `VALIDATION_FAILED` if the budget is outside 100–32000.

**Tests.** `tests/integration/test_context.py` (11), `tests/unit/test_context_builder.py` (13).

**Example.**

```json
→ {"project_id": "3f2c…", "token_budget": 500}

← {"brief": "# Project memory: queue-service\n\n## Constraints — these must not be violated\n…",
   "budget": {"requested": 500, "estimated_used": 331, "utilisation": 0.662,
              "estimator": "heuristic(chars/3.2)", "considered": 12, "selected": 6,
              "dropped": {"too_similar": 2, "no_budget_left": 4}},
   "semantic_coverage": null, "degraded": null}
```

---

## 5.8 The tool surface at a glance

| Tool | Kind | Writes? | Transaction | The one thing to remember |
|---|---|---|---|---|
| `project_use` | write (idempotent) | only on create | one | never resolves by guessing, never creates implicitly |
| `memory_remember` | write | yes | one, with a savepoint | supersession is folded in here, atomically |
| `memory_revise` | write (CAS) | yes | one | a conflict is a result, not an error |
| `memory_forget` | write | yes | one | tombstone, never destruction |
| `memory_search` | read | no | one read | retired memories are excluded before ranking |
| `memory_history` | read | no | one read | the only normal way to see retired memories |
| `memory_context` | read | no | one read | selection under a budget, not search |

---

# PART 6 — TRACING `memory_remember` END TO END

This part follows one real call all the way down. Every step below exists in the code; nothing is
invented. Read this alongside `src/memhub/services/memories.py`.

The call:

```json
{"name": "memory_remember",
 "arguments": {
   "project_id": "3f2c…", "type": "DECISION",
   "content": "The job queue runs on PostgreSQL. Redis was removed.",
   "supersedes": ["a1…"],
   "client": "cursor",
   "client_request_id": "req-4f19c2ab-8c31-4f6f-9b0e-1d2e3f4a5b6c"}}
```

**Diagram 4 — memory_remember**

```
 MCP request (JSON-RPC over stdin)
   |
   v
 SDK validates against the tool's input schema        <- protocol error if malformed
   |
   v
 @domain_errors wrapper                               (mcp/mapping.py)
   |
   v
 handler: parse UUIDs, map type string to enum        (mcp/server.py)
   |
   v
 session_scope(...)  ==  BEGIN                        (persistence/engine.py)
   |
   v
 services.memories.remember(...)
   |
   +-- 1. validate content / tags / importance / expiry
   |
   +-- 2. idempotency.claim(client_request_id)
   |        INSERT INTO idempotency_keys ... ON CONFLICT DO NOTHING
   |        if someone else owns it: SELECT ... FOR SHARE  (blocks)
   |        -> Replayed?  return the stored response, done
   |
   +-- 3. SAVEPOINT
   |        INSERT INTO memories        (speculative)
   |        INSERT INTO memory_revisions (revision 1, is_current = true)
   |        INSERT INTO memory_dedup_keys ... ON CONFLICT DO NOTHING
   |          -> zero rows?  ROLLBACK TO SAVEPOINT, attest the existing
   |                         memory, return outcome="deduplicated"
   |        RELEASE SAVEPOINT
   |
   +-- 4. supersession: for each target, in ascending UUID order
   |        UPDATE memories SET status='SUPERSEDED' ... WHERE status='ACTIVE'
   |        release that memory's dedup keys
   |        write an audit row per retirement
   |
   +-- 5. INSERT INTO embedding_jobs ... ON CONFLICT DO NOTHING   (the outbox)
   |
   +-- 6. INSERT INTO memory_attestations ... ON CONFLICT DO UPDATE
   |
   +-- 7. INSERT INTO audit_events (action='remember', outcome='ok')
   |
   +-- 8. UPDATE idempotency_keys SET state='COMPLETED', response=…
   |
   v
 COMMIT                                               <- everything above, or nothing
   |
   v
 RememberOut.of(result)                               (mcp/schemas.py)
   |
   v
 JSON-RPC response on stdout
```

Now each step in detail.

### Step 1 — schema parsing

**What.** The MCP SDK validates the incoming arguments against the tool's input schema, which it
derived by introspecting the handler function's type annotations.

**Why.** A malformed request should be rejected at the protocol boundary, before any code runs.

**File.** The SDK, driven by the annotations in `src/memhub/mcp/server.py`.

**A detail worth knowing.** `src/memhub/mcp/server.py` deliberately does **not** use
`from __future__ import annotations`, and `@domain_errors` uses `functools.wraps`. Both exist for
the same reason: the SDK reads runtime annotations to build the schema, and either omission would
produce a tool advertising no parameters at all. There is a regression test for it:
`test_tool_inputs_are_introspected_not_empty`.

**On failure.** A JSON-RPC protocol error. Nothing in the database is touched.

### Step 2 — id parsing and enum mapping

**What.** `_parse_uuid` and `_parse_type` in the handler convert strings into `uuid.UUID` and
`MemoryType`.

**Why.** So a bad type name produces a message naming the allowed values instead of a database
error.

**On failure.** `ValidationFailedError`, mapped by `@domain_errors` into a **tool error** —
`is_error=True` with a message like `[VALIDATION_FAILED] Unknown memory type 'OBSERVATION'. Use
one of: DECISION, CONSTRAINT, FACT, TASK.`

**Test.** `test_unknown_type_is_rejected_with_the_allowed_values`.

### Step 3 — the transaction opens

**What.** `async with session_scope(session_factory) as session:` obtains a connection from the
pool and opens a transaction.

**Why.** Everything that follows must be atomic.

**File.** `src/memhub/persistence/engine.py`:

```python
async with factory() as session:
    try:
        yield session
    except BaseException:
        await session.rollback()
        raise
    else:
        await session.commit()
```

**On failure.** If the pool has no free connection within 2 seconds, `BACKEND_BUSY`. If the
database is unreachable, `BACKEND_UNAVAILABLE`.

### Step 4 — domain validation

**What.** `validate_content`, `validate_tags`, `validate_importance`, `resolve_expiry` in
`src/memhub/domain/validation.py`.

**Why.** Every one of these rules is *also* a database `CHECK` constraint. The constraint is the
guarantee; the validator is the good error message. A caller should learn "content is 9102
characters, the limit is 8192", not `IntegrityError: ck_memory_revisions_content_length`.

**On failure.** `VALIDATION_FAILED`, nothing written.

### Step 5 — idempotency claim

**What.** Because `client_request_id` was supplied, `idempotency.claim(...)` runs before anything
is written.

```sql
INSERT INTO idempotency_keys
    (project_id, client_request_id, operation, request_fingerprint, state)
VALUES (:project_id, :client_request_id, :operation, :request_fingerprint, 'IN_PROGRESS')
ON CONFLICT (project_id, client_request_id) DO NOTHING
RETURNING client_request_id;
```

**Why here, before the write.** So a retry of a request that already succeeded replays the
original answer instead of creating a second memory.

**Three outcomes:**

- One row returned: this caller owns the key. Continue.
- Zero rows and a row exists with state `COMPLETED`: return the stored response as
  `outcome="idempotent_replay"`.
- Zero rows and no row exists: the previous owner rolled back. Loop and try again, at most three
  times.

If the key exists but the `request_fingerprint` differs, raise `IDEMPOTENCY_KEY_REUSED` rather
than answering a question the caller never asked.

**Files.** `src/memhub/services/idempotency.py`,
`src/memhub/persistence/sql/claim_idempotency.sql`, `wait_idempotency.sql`.

### Step 6 — deduplication, on a savepoint

**What.** A savepoint is opened, the memory and its first revision are inserted speculatively, and
then the dedup key is claimed:

```sql
INSERT INTO memory_dedup_keys (project_id, hash_version, content_hash, memory_id)
VALUES (:project_id, :hash_version, :content_hash, :memory_id)
ON CONFLICT (project_id, hash_version, content_hash) DO NOTHING
RETURNING memory_id;
```

**Why a savepoint.** The dedup key carries a foreign key to the memory, so the memory row must
exist before the key can be claimed. That makes the insert speculative. A savepoint is a marker
inside a transaction you can roll back *to*, undoing only what happened after it. If the key turns
out to be taken, only the speculative insert is undone — the idempotency claim made in step 5
survives, as does anything the caller's own transaction already contained.

**Why not a check-then-insert.** Because "SELECT to see if it exists, then INSERT" races: two
clients can both pass the SELECT and both insert. One statement with `ON CONFLICT DO NOTHING`
cannot, because the primary key adjudicates atomically.

**On a dedup hit.** Roll back to the savepoint, look up who holds the key, record an attestation
for this client, write an audit row with `outcome='dedup'`, and return the existing memory with
`outcome="deduplicated"`.

**Files.** `src/memhub/services/memories.py` (`remember`, `_deduplicate`),
`src/memhub/persistence/sql/claim_dedup_key.sql`.

### Step 7 — the memory and its first revision

**What.** `MemoryRepository.create` adds a `Memory` row (`current_revision_no=1`,
`status='ACTIVE'`) and a `MemoryRevision` row (`revision_no=1`, `is_current=True`).

**Why both in one place.** So a memory can never exist without a current revision — the state that
would make the current-revision pointer disagree with the log.

**Note on the content hash.** `content_hash` and `hash_version` are computed and stored on every
revision, including revisions that dedup never consults. That was a deliberate early decision:
`content_hash` is `NOT NULL` on an append-only table, so adding it later would have required a
data migration to backfill every existing row.

### Step 8 — supersession

**What.** `_apply_supersession` walks the target ids **in ascending order** and runs
`cas_supersede.sql` on each:

```sql
UPDATE memories
   SET status = 'SUPERSEDED', superseded_at = now(),
       superseded_by_id = :winner_id, updated_at = now()
 WHERE id = :memory_id AND project_id = :project_id AND status = 'ACTIVE'
RETURNING id;
```

**Why ascending order.** Every write path in this system takes memory row locks in the same order.
A single consistent lock order across all writers means no cycle of waits can form, so this
cannot deadlock.

**Why `status = 'ACTIVE'` in the WHERE.** It makes supersede, forget and revise serialise on the
same row lock rather than racing around each other, and zero rows returned is a clean answer
meaning "already retired, already deleted, or not in this project".

**Why report rather than skip.** Targets that did not retire come back in `not_superseded`.
Telling a caller "I retired 3 memories" when only 2 existed is a lie they would act on.

**Also.** Retiring a memory releases its dedup keys, so the same sentence can legitimately be
asserted again later.

**Test.** `test_already_retired_targets_are_reported_not_silently_skipped`,
`test_supersession_cannot_cross_a_project`, `test_one_memory_can_retire_several`.

### Step 9 — the embedding outbox row

**What.** If an embedding model is configured:

```sql
INSERT INTO embedding_jobs (memory_id, revision_no, project_id, model)
VALUES (:mid, :rev, :pid, :model)
ON CONFLICT (memory_id, revision_no, model) DO NOTHING
```

**Why in this transaction.** This is what makes it an outbox rather than a dual write. The job row
and the revision commit together, so there is no window in which a memory exists with no vector
and nothing knows to produce one, and no window in which a job points at a revision that was
rolled back.

**Test.** `test_a_write_enqueues_a_job_in_the_same_transaction`,
`test_a_rolled_back_write_leaves_no_job`.

### Step 10 — attestation

**What.**

```sql
INSERT INTO memory_attestations (memory_id, project_id, client_name)
VALUES (…)
ON CONFLICT (memory_id, client_name)
DO UPDATE SET times_seen = times_seen + 1, last_seen_at = now()
```

then `COUNT(*)` over that memory's attestation rows, which is a count of distinct client names
because `client_name` is part of the primary key.

**Why an upsert.** This also runs on the deduplication path, where two clients may be asserting at
the same instant.

**Why distinct clients rather than call count.** So one client retrying in a loop cannot
manufacture the appearance of independent corroboration.

### Step 11 — audit

**What.** One `audit_events` row: `action='remember'`, `outcome='ok'`, the actor client, the
memory id, revision number, and a JSON detail object holding the content **length** and the number
of memories superseded.

**Why not the content.** So the audit log can be read freely without exposing what the memories
say.

**Why a table rather than a log line.** The audit trail is part of the product — `memory_history`
returns it. Application logs rotate away; this does not.

### Step 12 — completing the idempotency key

**What.** `idempotency.complete(...)` sets `state='COMPLETED'`, stores
`{"memory_id": …, "revision_no": 1}` as the response, and sets `completed_at` from
`SELECT now()` — the database clock, never Python's.

**Why in the same transaction.** So there is no state where the key says COMPLETED but the memory
does not exist.

### Step 13 — commit

**What.** `session_scope` commits on the way out.

**Why it matters.** Every row from steps 5 through 12 becomes visible at the same instant. Another
client can never observe the new memory existing while the old one is still ACTIVE.

**On failure here.** If the connection dies during the commit, the classifier returns
`UNKNOWN_OUTCOME`. See Part 26.

### Step 14 — the response

**What.** `RememberOut.of(result)` maps the frozen domain dataclass into a Pydantic model, which
the SDK serialises as `structuredContent`.

**Why a declared output schema.** So a client can validate the response, and so the golden
manifest test can catch an accidental API change.

---

# PART 7 — DATABASE SCHEMA

## 7.1 The tables

Ten application tables, plus Alembic's own `alembic_version`. Verified from
`src/memhub/persistence/models.py` and migrations `0002` through `0006`.

**Diagram 5 — ER diagram (simplified)**

```
   projects
   +----------------+
   | id (PK)        |<-------------------+
   | slug (UNIQUE)  |                    |
   | display_name   |                    |
   +----------------+                    |
        ^                                |
        | FK                             | FK
   project_aliases                       |
   +--------------------------+          |
   | id (PK)                  |          |
   | project_id               |          |
   | kind, value_norm         |          |
   | UNIQUE (kind,value_norm) |  <- global, not per project
   +--------------------------+          |
                                         |
   memories                              |
   +------------------------------------+|
   | id (PK)                            ||
   | project_id  --------------------->--+
   | type, status, importance            |
   | current_revision_no                 |
   | expires_at, deleted_at              |
   | superseded_at                       |
   | superseded_by_id  ---+              |
   | UNIQUE (id, project_id)             |
   +---------------------|---------------+
        ^   ^   ^   ^    |
        |   |   |   |    +--- composite FK (superseded_by_id, project_id)
        |   |   |   |         -> memories (id, project_id)      SELF-REFERENCE
        |   |   |   |
        |   |   |   +----------------------------+
        |   |   +---------------+                |
        |   +------+            |                |
        |          |            |                |
  memory_revisions |     memory_dedup_keys   memory_attestations
  +--------------+ |     +----------------+  +------------------+
  | memory_id  PK| |     | project_id  PK |  | memory_id     PK |
  | revision_no PK| |    | hash_version PK|  | client_name   PK |
  | project_id   | |     | content_hash PK|  | project_id       |
  | content      | |     | memory_id      |  | times_seen       |
  | content_hash | |     +----------------+  | first/last_seen  |
  | tags[]       | |                          +------------------+
  | is_current   | |
  | source       | |     audit_events  (NO foreign key, deliberately)
  | author_client| |     +--------------------+
  | author_kind  | |     | id (PK, bigserial) |
  | content_tsv  | |     | project_id         |
  +------+-------+ |     | memory_id          |
         ^         |     | action, outcome    |
         | FK      |     | actor_client       |
         | (memory_id, revision_no)           |
         |         |     | detail (jsonb)     |
  +------+-------+ |     +--------------------+
  |memory_       | |
  | embeddings   | |     idempotency_keys
  | memory_id PK | |     +-----------------------+
  | revision_no PK|     | project_id         PK |
  | model      PK|      | client_request_id  PK |
  | embedding    |      | operation             |
  |  vector(384) |      | request_fingerprint   |
  +--------------+      | state, response       |
                        | expires_at            |
  embedding_jobs        +-----------------------+
  +----------------------------+
  | id (PK, bigserial)         |
  | memory_id, revision_no  FK |
  | project_id, model          |
  | state, attempts            |
  | next_attempt_at, last_error|
  | UNIQUE (memory_id, revision_no, model)
  +----------------------------+
```

## 7.2 Table by table

### `projects`

**One row represents:** one memory namespace.

- **Primary key.** `id`, a server-issued UUID (`gen_random_uuid()`).
- **Important columns.** `slug` (unique, human-facing), `display_name`, `archived_at`.
- **Constraints.** `UNIQUE (slug)`; `CHECK (slug ~ '^[a-z0-9][a-z0-9._-]{0,62}[a-z0-9]$')`.
- **Why the UUID is identity and the slug is not.** A slug can in principle be renamed. A UUID
  never changes and is unguessable.
- **Migration.** `0002`.

### `project_aliases`

**One row represents:** one hint that resolves to a project — a git remote or a workspace path.

- **Primary key.** `id`.
- **Foreign key.** `project_id` → `projects.id`, `ON DELETE CASCADE`.
- **The key design point.** `UNIQUE (kind, value_norm)` is **global**, not per project. An alias
  value therefore cannot resolve to two projects; ambiguity is impossible by construction rather
  than by careful query writing.
- **Normalisation.** `normalize_git_remote` collapses `git@github.com:me/repo.git` and
  `https://github.com/me/repo` to the same value, so SSH in one client and HTTPS in another do not
  create two namespaces.
- **Migration.** `0002`.

### `memories`

**One row represents:** one logical fact's identity and lifecycle. **It never holds content.**

- **Primary key.** `id` (UUID).
- **Important columns.** `project_id`, `type`, `status`, `current_revision_no`, `importance`,
  `expires_at`, `superseded_by_id`, `superseded_at`, `deleted_at`, `created_at`, `updated_at`.
- **Unique constraint.** `UNIQUE (id, project_id)` — which looks redundant next to the primary
  key, and is not: it is what lets child tables carry a *composite* foreign key that pins the
  project.
- **Foreign keys.** `project_id` → `projects.id` `ON DELETE RESTRICT`; and the important one,
  `(superseded_by_id, project_id)` → `(memories.id, memories.project_id)`.
- **CHECK constraints.** type in the four values; status in the three; `current_revision_no >= 1`;
  `importance BETWEEN 0 AND 100`; `superseded_by_id IS DISTINCT FROM id`;
  `(status='SUPERSEDED') = (superseded_at IS NOT NULL)`;
  `status <> 'SUPERSEDED' OR superseded_by_id IS NOT NULL`;
  `(status='DELETED') = (deleted_at IS NOT NULL)`;
  `type <> 'TASK' OR expires_at IS NOT NULL`.
- **Indexes.** `(project_id, type, importance DESC) WHERE status='ACTIVE'`;
  `(expires_at) WHERE expires_at IS NOT NULL AND status='ACTIVE'`;
  `(superseded_by_id) WHERE superseded_by_id IS NOT NULL`.
- **Migration.** `0002`.

### `memory_revisions`

**One row represents:** one version of one memory's text. Append-only.

- **Primary key.** `(memory_id, revision_no)`. This alone makes it impossible for two concurrent
  writers to both create revision 5.
- **Important columns.** `content` (1–8192 chars), `content_hash`, `hash_version`, `tags` (array,
  max 16), `is_current`, `change_reason`, `source`, `author_client`, `author_kind`, `created_at`,
  and `content_tsv`.
- **`content_tsv`** is a **generated column**: `to_tsvector('english', content)`, computed and
  stored by PostgreSQL itself. There is no code path that writes content without also producing
  its search vector, so the two cannot drift.
- **Foreign key.** `(memory_id, project_id)` → `(memories.id, memories.project_id)`
  `ON DELETE CASCADE`.
- **The invariant index.** `UNIQUE (memory_id) WHERE is_current` — at most one current revision
  per memory, enforced by the database.
- **Other indexes.** GIN on `content_tsv WHERE is_current`; GIN on `tags WHERE is_current`; a
  plain index on `project_id`.
- **Why `project_id` is duplicated here.** Two reasons: it makes the composite foreign key
  possible, and it lets project-scoped scans avoid a join.
- **Migrations.** `0002`, then `0005` added `content_tsv` and the two GIN indexes.

### `idempotency_keys`

**One row represents:** one logical client request and the answer it produced.

- **Primary key.** `(project_id, client_request_id)`. Scoped to a project, so two projects using
  the same key do not collide.
- **Important columns.** `operation`, `request_fingerprint` (SHA-256 of the canonical arguments),
  `state` (`IN_PROGRESS` | `COMPLETED`), `response` (JSONB), `expires_at` (default 7 days).
- **CHECK.** `state <> 'COMPLETED' OR response IS NOT NULL` — a completed claim with no stored
  response could not be replayed, which would silently turn a retry into a second write.
- **Index.** `(expires_at)`, supporting the bounded GC sweep.
- **Migration.** `0003`.

### `audit_events`

**One row represents:** one thing that happened to a memory.

- **Primary key.** `id` (bigserial).
- **Important columns.** `at`, `project_id`, `memory_id`, `revision_no`, `action`, `outcome`,
  `actor_client`, `request_id`, `detail` (JSONB).
- **CHECKs.** `action IN ('remember','revise','forget','supersede','purge')`;
  `outcome IN ('ok','conflict','dedup','replay','rejected')`.
- **No foreign key, on purpose.** An audit row must survive the thing it describes. An operator
  purge destroys a memory and every revision of it, and the record that the purge happened has to
  outlive that. A `CASCADE` would erase the evidence along with the subject. There is a test for
  this: `test_the_audit_record_outlives_the_memory`.
- **Migration.** `0003`.

### `memory_dedup_keys`

**One row represents:** one distinct active fact in a project.

- **Primary key.** `(project_id, hash_version, content_hash)`.
- **Foreign key.** `(memory_id, project_id)` → `memories`, `ON DELETE CASCADE`.
- **Why a separate table rather than a unique index on revisions.** The rule wanted is "no two
  *active* memories in a project have identical normalised content". But `is_current` lives on the
  revision and `status` lives on the memory, and a unique index cannot span two tables. A
  dedicated table whose rows are inserted on create and deleted on forget or supersede gives a
  real unique constraint over exactly the right set.
- **Why `hash_version` is in the key.** So a future change to the normaliser can be rolled out
  alongside the old one rather than as a big-bang re-hash.
- **Migration.** `0004`.

### `memory_attestations`

**One row represents:** one client's assertion of one fact.

- **Primary key.** `(memory_id, client_name)`.
- **Important columns.** `times_seen`, `first_seen_at`, `last_seen_at`.
- **Why keyed on client name.** So `COUNT(*)` is a count of *distinct clients*, and one client
  retrying cannot manufacture corroboration.
- **Migration.** `0004`.

### `memory_embeddings`

**One row represents:** one vector, for one revision, under one model.

- **Primary key.** `(memory_id, revision_no, model)`.
- **Important columns.** `project_id`, `dim`, `embedding` of type `vector(384)`.
- **Foreign key.** `(memory_id, revision_no)` → `memory_revisions`, `ON DELETE CASCADE`.
- **CHECK.** `dim = 384`.
- **Index.** HNSW on `embedding` with `vector_cosine_ops`.
- **Why its own table.** A 384-dimension vector is around 1.5 KB. Putting it inline would widen
  the metadata row that every filter and join touches. Re-embedding under a new model would
  rewrite the append-only content log. And having the model in the key means vectors from
  different models can coexist and can never be compared to each other — which would be
  meaningless, because embedding spaces are not comparable.
- **Migration.** `0006`, which first runs `CREATE EXTENSION IF NOT EXISTS vector`.

### `embedding_jobs`

**One row represents:** one pending piece of embedding work — the transactional outbox.

- **Primary key.** `id` (bigserial).
- **Unique.** `(memory_id, revision_no, model)` — a revision needs embedding once per model.
- **Important columns.** `state` (`PENDING` | `DONE` | `DEAD`), `attempts`, `next_attempt_at`,
  `last_error`.
- **Index.** `(next_attempt_at) WHERE state = 'PENDING'` — partial, so at steady state, when
  almost every row is `DONE`, the index stays near-empty.
- **Migration.** `0006`.

## 7.3 The model split, in one sentence

> **A mutable identity-and-lifecycle row (`memories`) pointing into an immutable content log
> (`memory_revisions`).**

That is the whole data model, and it is what makes history free and audit trivial.

## 7.4 Verified discrepancies between the architecture document and the schema

| # | `docs/architecture.md` says | The implementation has | Impact |
|---|---|---|---|
| 1 | `embedding vector(768)` (§4.4) | `vector(384)`, `EMBEDDING_DIM = 384` | The doc predates the choice of `BAAI/bge-small-en-v1.5`, which produces 384 dimensions. The code and migration `0006` agree with each other. |
| 2 | `memory_revisions` DDL has no `source` column | `source: Mapped[str \| None]` exists and is returned in `MemoryOut.source` | The doc's DDL is incomplete; provenance is slightly richer than documented. |
| 3 | `audit_events` outcome is `ok / conflict / dedup / rejected` | The CHECK also allows `replay` | Minor; the code's set is wider. |
| 4 | `memory_revisions` FK is described plainly | It is `ON DELETE CASCADE` | Consistent with purge behaviour. |

Treat the implementation as the truth in all four cases.

---

# PART 8 — REVISION VS SUPERSESSION

These two are constantly confused. Getting them apart is one of the strongest things you can
demonstrate about this project.

## 8.1 The problem

Knowledge changes in two different ways, and they are not the same change.

**Way one: the same fact gets better.** You wrote "The job queue runs on Redis." Later you want to
say "The job queue runs on Redis, with dedicated worker processes." Same fact, more detail. The
author might be the same person; the creation time of the underlying fact has not changed.

**Way two: the fact is replaced.** You wrote "The job queue runs on Redis." Six months later the
team moved to PostgreSQL. "The job queue runs on PostgreSQL. Redis was removed." is a *different*
statement, decided at a different time, possibly by a different person, and it might replace three
old memories at once, not one.

If you force both into one mechanism, you break something.

- Force everything to be a revision, and the replacement has to pretend to be "revision 2 of the
  Redis memory". You then have to lie about who authored it and when it was first asserted, and
  you cannot express one memory retiring three.
- Force everything to be a supersession, and every typo correction creates a new memory with a new
  identity, and the revision history of a single idea is scattered across many ids.

## 8.2 Simple intuition

- **Revision** = editing a document. Same document, new version.
- **Supersession** = writing a new document that says "this replaces the old one", and stamping
  the old one VOID while keeping it in the filing cabinet.

## 8.3 Concrete example

**Revision (one memory, two revisions):**

```
Memory A
  revision 1: "The job queue runs on Redis."                        is_current = false
  revision 2: "The job queue runs on Redis, with dedicated workers." is_current = true
  memories.current_revision_no = 2, status = ACTIVE
```

**Supersession (two memories):**

```
Memory A  "The job queue runs on Redis."
            status = SUPERSEDED, superseded_by_id = B, superseded_at = <timestamp>

Memory B  "The job queue runs on PostgreSQL. Redis was removed."
            status = ACTIVE
```

## 8.4 Diagrams

**Diagram 6 — revision**

```
  memories row A                     memory_revisions
  +---------------------+            +---------------------------------------+
  | id = A              |            | (A, 1)  "…runs on Redis."   is_current=F|
  | current_revision_no |--- 2 ----->| (A, 2)  "…with dedicated…"  is_current=T|
  | status = ACTIVE     |            +---------------------------------------+
  +---------------------+
        one identity                  many immutable versions
```

**Diagram 7 — supersession**

```
   memories row A                          memories row B
   +--------------------------+            +--------------------------+
   | id = A                   |            | id = B                   |
   | "…runs on Redis."        |            | "…runs on PostgreSQL."   |
   | status = SUPERSEDED      |-- points ->| status = ACTIVE          |
   | superseded_by_id = B     |            |                          |
   | superseded_at = <ts>     |            |                          |
   +--------------------------+            +--------------------------+
        leaves retrieval                        appears in retrieval
        stays in memory_history

   Many-to-one is allowed:   A1 --+
                             A2 --+--> B
                             A3 --+
```

## 8.5 The comparison table

| | Revision | Supersession |
|---|---|---|
| Question it answers | "How did *this fact* change?" | "Which fact replaced *that* fact?" |
| Cardinality | 1 memory → N revisions | N old memories → 1 new memory |
| Number of `memories` rows | one | two or more |
| Authorship | may differ per revision, same logical fact | different memories, different authors, different creation times |
| Where it lives | `memory_revisions` | `memories.superseded_by_id` |
| Which tool | `memory_revise` | `memory_remember(supersedes=[...])` |
| Guarded by | compare-and-set on `current_revision_no` | compare-and-set on `status = 'ACTIVE'` |
| SQL file | `sql/cas_revise.sql` | `sql/cas_supersede.sql` |
| Status after | still `ACTIVE` | old one becomes `SUPERSEDED` |
| Appears in search after | yes, at the new revision | no — the old one is gone from retrieval |
| Old text still readable | yes, in `memory_history` | yes, in `memory_history` |
| Running example | "…on Redis" → "…on Redis, with dedicated workers" | "…on Redis" retired by "…on PostgreSQL" |

## 8.6 Why revisions are immutable

**Immutable** here means the content of a revision row is never rewritten. When you revise, the
old row's `is_current` flag flips to `false` and a *new* row is appended.

Three reasons:

1. **History is the product.** `memory_history` can only show "what did we believe before" if the
   before still exists.
2. **Audit.** Someone will ask why a decision changed. If the old text was overwritten, there is
   no answer.
3. **Correctness under concurrency.** Because the primary key is `(memory_id, revision_no)`, two
   concurrent writers cannot both create revision 5 — the database rejects the second with error
   `23505`. That is a safety net underneath the compare-and-set.

## 8.7 Why superseded memories stay stored

Because retirement removes a memory from *retrieval*, not from the *record*. If retirement deleted
the row, the system would simply be erasing inconvenient history and nobody could ask why the
project changed its mind. The docstring in `services/memories.py` calls `memory_history` "the
counterweight to stale-memory suppression".

## 8.8 Why a superseded memory cannot appear as current knowledge

Because every retrieval path composes on one filter that requires `status = 'ACTIVE'`. This is
Part 21 in full. In one line: a superseded memory is not ranked low — it is never a candidate.

## 8.9 How `memory_history` behaves

It calls `MemoryRepository.get(project_id, memory_id, include_retired=True)`. That is one of only
two places the flag is used. It returns:

- the memory at its current revision, with its status;
- every revision, oldest first, each with its own author and timestamp;
- `superseded_by` — the memory that retired this one, with its content;
- `supersedes` — the memories this one retired;
- `attestations` — which clients asserted it;
- `audit` — the most recent 50 events.

## 8.10 Transactions involved

Supersession happens inside the *same* transaction as the creation of the replacing memory. That
is the reason there is no `memory_supersede` tool. If retiring and asserting were two calls, there
would be a window in which the project appears to have no opinion about something it has a firm
opinion about.

## 8.11 Tests

| Behaviour | Test |
|---|---|
| Retired memory absent at every limit and every query | `test_the_retired_fact_leaves_retrieval_entirely` |
| Searching the stale term returns only the current answer | `test_querying_the_stale_term_still_finds_only_the_current_answer` |
| History still shows it, with the link and the original author | `test_history_still_shows_the_retired_fact` |
| One memory retiring three | `test_one_memory_can_retire_several` |
| Already-retired targets reported, not skipped | `test_already_retired_targets_are_reported_not_silently_skipped` |
| Supersession cannot cross a project | `test_supersession_cannot_cross_a_project` |
| Revisions survive a later forget | `test_a_revised_memory_can_be_forgotten_and_keeps_all_revisions` |
| A retired sentence can be asserted again | `test_a_retired_sentence_can_be_asserted_again` |

---

# PART 9 — CONCURRENCY FROM ZERO

## 9.1 What is concurrency?

**Concurrency** is more than one thing happening in overlapping time.

Here it is very literal: Claude Desktop's memhub process and Cursor's memhub process are two
separate programs running at the same time, both connected to the same database. Neither knows
the other exists.

## 9.2 What is a race condition?

A **race condition** is a bug where the outcome depends on the exact timing of concurrent
operations. Run it a thousand times and it is fine; run it once at the wrong moment and it is
wrong. That is what makes race conditions hard: they are not reliably reproducible.

## 9.3 What is a lost update?

A **lost update** is one specific race, and it is the one this project is built to prevent.

Two clients read the same value. Both compute a new value from what they read. Both write. The
second write silently erases the first one, and nobody is told.

**Concrete example, with the running example:**

```
The memory currently says:  "The job queue runs on Redis."     revision 4

10:00:00.000  Client A reads it.  A sees revision 4.
10:00:00.001  Client B reads it.  B sees revision 4.
10:00:00.100  Client A writes "…on Redis, with dedicated workers."
10:00:00.200  Client B writes "…on Redis, monitored by BRPOP timeouts."
```

Without protection, the memory now says B's text. A's change is gone. Nobody knows. A believes it
succeeded. Anyone reading later sees only B's sentence and has no idea a second edit existed.

**Diagram 8 — the lost-update race**

```
   time ->

   A:  read(rev 4) ------------ write("…dedicated workers") ------> OK
   B:       read(rev 4) --------------- write("…BRPOP timeouts") --> OK

   final content: B's text
   A's edit: silently destroyed
   A's belief: "my change was saved"
```

## 9.4 What is a stale client?

A **stale client** is a client acting on a value that has since changed. In the diagram above,
client B is stale from the moment A commits: B still believes revision 4 is current, and it is
not.

The word "stale" appears twice in this project with two different meanings. Keep them apart:

- A **stale client** holds an out-of-date revision number.
- A **stale memory** is knowledge that was true once and is not current now.

Both matter. They are solved by different mechanisms — compare-and-set for the first, structural
exclusion for the second.

## 9.5 The idea that fixes it, in plain words

Do not let a writer say "set the content to X". Make it say:

> "Set the content to X, **but only if the revision is still 4**."

The database checks that condition and the write at the same instant, so nothing can slip in
between. If the revision has moved on, zero rows change and the writer is told it lost.

Break the name into its two halves before using it:

- **Compare** — check that the current value equals what the caller read.
- **Set** — write the new value.

Both in one atomic operation. Hence **compare-and-set**, abbreviated **CAS**.

**Diagram 9 — CAS**

```
   Starting state: memories.current_revision_no = 4

   Client A:   expected = 4     current = 4     ->  MATCH  -> update to 5, SUCCESS
   Client B:   expected = 4     current = 5     ->  NO MATCH -> 0 rows,  CONFLICT
                                                              (and B is told
                                                               current is 5)
```

## 9.6 The actual SQL

`src/memhub/persistence/sql/cas_revise.sql`:

```sql
UPDATE memories
   SET current_revision_no = current_revision_no + 1,
       updated_at          = now()
 WHERE id                  = :memory_id
   AND project_id          = :project_id
   AND current_revision_no = :expected_revision
   AND status              = 'ACTIVE'
RETURNING current_revision_no AS new_revision;
```

Line by line, with what breaks if you remove each part:

**`UPDATE memories`** — the target is the identity row, not the content row. This is deliberate:
the identity row is the single point every writer must pass through, so it is where they can be
adjudicated.

**`SET current_revision_no = current_revision_no + 1`** — the increment is computed by the
database from the row's own current value, not from a number Python calculated. Even if the
application's arithmetic were wrong, the sequence cannot develop a gap.

**`SET updated_at = now()`** — `now()` is the *database* clock. If Python supplied the timestamp,
two server processes with slightly different clocks could disagree about the order of events.

**`WHERE id = :memory_id`** — which memory. Remove it and you would update every row.

**`WHERE project_id = :project_id`** — namespace isolation, in the write path. Remove it and a
caller who somehow held an id from another project could modify it. This condition is why a
cross-project supersession attempt simply matches zero rows rather than raising.

**`WHERE current_revision_no = :expected_revision`** — **this is the compare.** It is the whole
mechanism. Remove it and every writer succeeds, which is exactly the lost update. The README
records that removing this line makes the concurrency tests fail — that is the only evidence the
line was ever doing anything.

**`WHERE status = 'ACTIVE'`** — this does two jobs. It stops a revise from resurrecting a
tombstoned or superseded memory. And it makes `revise`, `forget` and `supersede` all serialise on
the same row lock, so they cannot race around each other. Without it, you could revise a memory
that another client was in the middle of retiring.

**`RETURNING current_revision_no AS new_revision`** — the winner gets the new number back in the
same round trip. And crucially, **zero rows returned is the conflict signal**. There is no
exception, no error code to interpret: `scalar_one_or_none()` returns `None` and the service layer
turns that into a `ReviseConflicted`.

The Python side, in `MemoryRepository.compare_and_set`:

```python
new_revision = (await self._session.execute(load(CAS_REVISE), {...})).scalar_one_or_none()
if new_revision is None:
    return None                       # conflict
# ... otherwise: demote the old revision, append the new one
```

## 9.7 Why the three statements are in that order

```
1. CAS UPDATE on memories            <- takes the row lock AND checks the predicate
2. UPDATE memory_revisions SET is_current = false WHERE revision_no = expected
3. INSERT the new memory_revisions row
```

Taking the `memories` row lock *first*, in every write path, is what makes the whole system
deadlock-free. A **deadlock** happens when transaction 1 holds lock X and wants Y while
transaction 2 holds Y and wants X — neither can proceed. It requires a cycle. A single consistent
lock order across all writers means no cycle can form.

`memory_remember(supersedes=[...])` follows the same rule: it locks its targets in ascending UUID
order.

---

# PART 10 — OPTIMISTIC CONCURRENCY

## 10.1 Why "optimistic"?

Because it assumes conflicts are **rare**. It does not prevent anyone from starting. It lets
everyone try, and detects the collision at the moment of writing.

The opposite is **pessimistic locking**: assume a conflict is likely, so take a lock before you
even read, and make everyone else wait until you are done. In SQL that would look like:

```sql
SELECT * FROM memories WHERE id = :id FOR UPDATE;   -- lock now, nobody else may proceed
-- ... think, decide, then write ...
COMMIT;                                              -- lock released here
```

## 10.2 Why optimistic was chosen here

Four reasons, each of which you can defend:

1. **Conflicts genuinely are rare.** In the real deployment there are two clients, driven by one
   human. Two clients revising the same memory in the same second is unusual.

2. **The loser gets a useful answer instead of a wait.** Under pessimistic locking, the second
   writer blocks. It learns nothing while it waits, and when it wakes up it still has to work out
   what changed. Under CAS it is refused immediately and handed the winning revision number,
   content, and author — everything it needs to merge and retry in one round trip.

3. **A held lock across an MCP round trip is dangerous.** The client is a language model. If a
   lock were held while waiting for the model to decide what to write next, one slow or abandoned
   model turn would block every other writer.

4. **The version number is meaningful.** `revision_no` is a real, human-readable, model-readable
   concept. It appears in `memory_history`. Using PostgreSQL's internal `xmin` system column for
   optimistic locking would work technically but would expose an opaque internal value to clients.

**The tradeoff, stated honestly:** under heavy contention on one row, optimistic concurrency
wastes work — 49 out of 50 writers here do their validation and then get refused. Pessimistic
locking would let all 50 eventually succeed, one after another. For this workload that is the
wrong trade, because a later writer succeeding by overwriting the earlier one is not obviously
what the user wants: the two edits might contradict each other, and only the caller can merge
them.

## 10.3 The isolation level

**READ** — a transaction reading data.
**COMMITTED** — changes that have been made permanent by their transaction finishing successfully.
**READ COMMITTED** — each statement sees only data that was committed before *that statement*
started. It never sees another transaction's uncommitted work. Two statements in the same
transaction can see different data if something committed in between.

This is PostgreSQL's default, and this project uses it. Nothing in the codebase sets a different
isolation level — verified by grep: there is no `SET TRANSACTION ISOLATION LEVEL` anywhere, and no
`isolation_level` argument to `create_async_engine`.

## 10.4 What PostgreSQL actually does during the conflict

Start simple, then add the machinery.

**The simple version.** Two transactions both try to `UPDATE` the same row. PostgreSQL will not
let them both proceed. The second one waits. When the first finishes, the second wakes up, looks
at the row *as it is now*, re-checks its own `WHERE` clause against it, finds that
`current_revision_no` is 5 rather than 4, decides the row does not match after all, and reports
"0 rows updated".

**Diagram — what the loser experiences**

```
   T1:  UPDATE ... WHERE current_revision_no = 4   -> matches, takes the row lock
   T2:  UPDATE ... WHERE current_revision_no = 4   -> blocked, waiting on T1's lock
   T1:  COMMIT                                      -> row is now revision 5
   T2:  wakes up
        re-reads the newest committed version of the row
        re-evaluates its own WHERE against it:  5 = 4 ?  no
        skips the row
        reports 0 rows updated
```

**Now the machinery: MVCC.**

**MVCC** stands for **Multi-Version Concurrency Control**. It is how PostgreSQL implements
isolation. When a row is updated, PostgreSQL does not overwrite it in place. It writes a *new
version* of the row and marks the old version as no longer valid from that transaction onward.
Each transaction is shown whichever version is appropriate to its snapshot.

The immediate benefit is that readers never block writers and writers never block readers. A
`SELECT` running at the same time as our `UPDATE` simply sees the older version and does not wait.

**And the specific behaviour that makes CAS correct: EvalPlanQual.**

Under READ COMMITTED, when an `UPDATE` is blocked by another transaction that then commits,
PostgreSQL does not simply proceed with the row version it originally found — that version is now
out of date. Instead it follows the chain to the newest committed version of that row and
**re-evaluates the `WHERE` clause against it**. This step is called `EvalPlanQual`.

That is the crux of the whole argument, and it is worth being able to say precisely:

> The read and the write are the same statement. There is no window between checking the revision
> and writing the new one, because the storage engine's own visibility machinery does the check at
> the moment of the write, against the newest committed row.

## 10.5 Why not SERIALIZABLE?

Under `REPEATABLE READ` or `SERIALIZABLE`, this same collision does not produce zero rows. It
raises PostgreSQL error `40001`, "could not serialize access due to concurrent update", and the
application is expected to catch it and retry the whole transaction.

That is also correct. It is worse *for this system*, for two reasons:

1. **It turns a clean domain outcome into an exception.** "Someone else changed this; here is
   their version" is a meaningful answer the model can act on. "Serialization failure, please
   retry" is not.
2. **It creates retry storms.** With 50 concurrent writers, you get 49 retries, each of which may
   collide again. The test assertion also stops being exact: how many succeed depends on how many
   retries each caller made.

With READ COMMITTED plus an explicit CAS, the result is deterministic: **1 success and 49
conflicts, every time, with no retries**.

The sentence for an interview:

> I chose READ COMMITTED *because of* the conflict semantics I wanted, not because it was the
> default.

## 10.6 Defence in depth

Suppose someone introduced a bug that skipped the CAS statement entirely. Two writers could still
not corrupt the corpus:

- `PRIMARY KEY (memory_id, revision_no)` rejects a second revision 5 with SQLSTATE `23505`.
- `UNIQUE (memory_id) WHERE is_current` rejects a second current revision, also `23505`.

The invariant survives wrong application code. Both are proved by
`tests/integration/test_invariants.py`, which bypasses the service layer and writes raw SQL.

---

# PART 11 — CONCURRENCY TESTING

## 11.1 The test file

`tests/concurrency/test_compare_and_set.py`, five tests, `CONCURRENCY = 50`.

## 11.2 The headline test, step by step

`test_exactly_one_of_fifty_writers_wins`:

**Setup.**

1. Build an engine with `db_pool_size = 51`, `db_max_overflow = 0`, `pool_timeout = 30s`,
   `statement_timeout = 30s` (`concurrent_engine` in `tests/concurrency/conftest.py`).
2. Call `assert_backends_are_distinct(factory, 50)` — see below.
3. Seed: create a project, then `remember(DECISION, "Redis is the task queue.")` authored by
   `claude-desktop`. That memory is at **revision 1**.

**The writers.** 50 coroutines. Each one:

- opens its **own session**, on its **own connection**, in its **own transaction**;
- calls `revise(project_id, memory_id, expected_revision=1, content=f"PostgreSQL is the task
  queue. Writer {index} won.", author_client="cursor")`.

**Synchronisation.** `run_together` puts an `asyncio.Barrier(50)` in front of the work. A barrier
makes every task wait until all 50 have arrived, then releases them together. Without it, tasks
start microseconds apart and the first routinely finishes before the last begins — which would not
be a race at all.

**Expected outcome.**

- exactly **1** result is `ReviseSucceeded`;
- exactly **49** results are `ReviseConflicted`;
- no result is an exception.

**Final state, checked with raw SQL:**

```
memories.current_revision_no                    == 2      (one increment, not fifty)
count(memory_revisions where memory_id = M)     == 2
count(... where memory_id = M and is_current)   == 1
the surviving content                           == the winner's payload
```

**Then** the whole invariant suite runs against the database
(`assert_invariants_hold`), because "exactly one succeeded" is not sufficient — the corpus also
has to be internally consistent afterwards.

## 11.3 The connection-pool trap

This is the detail worth naming in an interview.

If the connection pool holds 10 connections, launching 50 tasks does **not** give you 50-way
concurrency. It gives you five sequential waves of ten. The test still passes — and it passes for
entirely the wrong reason. Green, meaningless, and almost impossible to notice.

The fixture defends against it twice:

```python
sized = Settings(db_pool_size=concurrency + 1, db_max_overflow=0, ...)
assert sized.max_concurrent_connections > concurrency, (
    f"pool of {sized.max_concurrent_connections} cannot deliver "
    f"{concurrency}-way concurrency"
)
```

and then, at runtime, `assert_backends_are_distinct` actually proves it:

```python
barrier = asyncio.Barrier(concurrency)

async def hold() -> int:
    async with factory() as session:
        pid = (await session.execute(text("SELECT pg_backend_pid()"))).scalar_one()
        await barrier.wait()          # hold the connection until everyone has one
        return int(pid)

pids = await asyncio.gather(*(hold() for _ in range(concurrency)))
assert len(set(pids)) == concurrency
```

`pg_backend_pid()` returns the process id of the PostgreSQL server process handling this
connection. Fifty distinct values means fifty distinct backends. And because every task holds its
connection until the barrier trips, a pool that serialised them would hang rather than quietly
pass.

`docker-compose.yml` raises `max_connections=200` for exactly this reason.

## 11.4 The other four tests

| Test | What it adds |
|---|---|
| `test_losers_are_told_what_beat_them` | every loser gets `expected_revision=1`, `current.revision_no=2`, the winner's content, and `author_client="cursor"`. It first asserts `len(losers) == 49` — because without that assertion, the loop over losers passes vacuously when there are none, which is exactly what removing the version predicate produced. |
| `test_a_second_round_advances_by_exactly_one` | runs three rounds of 50. Final revision is 4, and `revision_no` values are exactly `[1,2,3,4]` — gapless and monotonic. |
| `test_history_is_preserved_not_overwritten` | revision 1 still says "Redis is the task queue." with `author_client="claude-desktop"` and `is_current=false`; revision 2 is the new text by `cursor`. |
| `test_every_attempt_is_audited` | `audit_events` holds 1 row with `outcome='ok'` and 49 with `outcome='conflict'`. Auditing only winners would hide the conflict rate, which is an operational signal that two clients are fighting over the same knowledge. |

## 11.5 What this test proves

- A lost update cannot happen through the `revise` path under real concurrent load.
- The conflict outcome is deterministic — exactly one winner, always, with no retries.
- Losers receive enough information to recover in one round trip.
- Revision numbers stay gapless and monotonic under repeated contention.
- History is appended, never overwritten.
- The database's own invariants still hold afterwards.

## 11.6 What this test does NOT prove

Be able to say this unprompted; it is a credibility marker.

- **It is not 50 MCP clients.** It is 50 concurrent callers of the service layer. With stdio there
  are realistically two client processes. This is a synthetic load test of the write path, and a
  completely valid correctness test — but the README should not imply 50 MCP clients, and it does
  not.
- **It does not prove throughput or scalability.** It is a correctness test, not a benchmark.
- **It does not prove the MCP layer behaves the same way.** That is a separate test:
  `test_conflict_is_a_result_not_an_error` in the protocol suite.
- **It does not prove anything about network partitions**, replica lag, or multi-node PostgreSQL.
  There is one database.
- **It does not prove the system is correct under `SERIALIZABLE`.** It is specifically a
  READ COMMITTED result.

## 11.7 Why it needs real PostgreSQL

Because every guarantee being tested is a PostgreSQL behaviour:

- row-level locking on `UPDATE`;
- `EvalPlanQual` re-checking the predicate under READ COMMITTED;
- partial unique indexes;
- `ON CONFLICT DO NOTHING` waiting on an uncommitted insert;
- `FOR SHARE` blocking until a transaction resolves.

SQLite has none of these semantics. A test that passed on SQLite would prove nothing about the
system actually being shipped. The test harness (`tests/conftest.py`) says exactly this, and there
is no mocked database anywhere in the integration suite.

The harness makes real PostgreSQL affordable with two tricks:

1. **Template databases.** Migrations run once per session into a template database. Each test
   module then does `CREATE DATABASE ... TEMPLATE`, which PostgreSQL implements as a file copy.
   Running the migration chain per module would grow linearly with the number of migrations;
   cloning does not.
2. **Transaction rollback per test** — for everything except concurrency tests, which cannot use
   it, because two writers inside one transaction are not two writers.

---

# PART 12 — IDEMPOTENCY

## 12.1 The problem, told as a story

Client A sends `memory_remember` with content `"The job queue runs on PostgreSQL."`.

The server receives it. The transaction commits. The row is in the database.

Then the connection drops — the network hiccups, the pipe closes, the process is killed — and the
response never reaches the client.

Client A is now in the worst possible position. It knows what it sent. It does not know what
happened. From its side, these two situations look **identical**:

```
  (a) the write committed, and the acknowledgement was lost
  (b) the write never happened
```

If A assumes (b) and retries, and the truth was (a), the project now has the same decision stored
twice. If A assumes (a) and does nothing, and the truth was (b), the decision is lost.

**Diagram 10 — idempotency**

```
   WITHOUT a key                            WITH a key
   -------------                            ----------

   A --- remember("X") ---> [commit]        A --- remember("X", key=R) ---> [commit,
        <--- response lost ---                    key R stored COMPLETED
                                                  with the response]
   A --- remember("X") ---> [commit]             <--- response lost ---
                                             A --- remember("X", key=R) --->
   result: TWO memories                           key R already COMPLETED
                                                  -> replay the stored response

                                             result: ONE memory, and A gets
                                                     the same answer it would
                                                     have got the first time
```

## 12.2 The mechanism, step by step

**The key.** A caller-supplied string, `client_request_id`, 8 to 128 characters. A UUID is ideal.
It identifies *one logical request*, not one attempt. Every retry of that request carries the same
value.

**Storage.** The `idempotency_keys` table, primary key `(project_id, client_request_id)`.

**Why not "check then insert".** Because that races. Two retries can both run the `SELECT`, both
see nothing, and both insert. A single statement cannot race, because the primary key adjudicates
atomically.

**Step 1: claim.** `src/memhub/persistence/sql/claim_idempotency.sql`:

```sql
INSERT INTO idempotency_keys
    (project_id, client_request_id, operation, request_fingerprint, state)
VALUES
    (:project_id, :client_request_id, :operation, :request_fingerprint, 'IN_PROGRESS')
ON CONFLICT (project_id, client_request_id) DO NOTHING
RETURNING client_request_id;
```

One row back means this caller owns the key. Zero rows means someone else does.

There is a subtlety in that statement worth knowing, and the SQL file spells it out: under READ
COMMITTED, if a *concurrent uncommitted* transaction has already inserted this key,
`ON CONFLICT DO NOTHING` does not return immediately — it **waits** for that transaction to
finish, then returns zero rows. So a zero-row result already means "the other writer has committed
or rolled back", not "might still be in flight".

**Step 2: wait and read.** `wait_idempotency.sql`:

```sql
SELECT state, response, request_fingerprint, operation
  FROM idempotency_keys
 WHERE project_id = :project_id
   AND client_request_id = :client_request_id
   FOR SHARE;
```

`FOR SHARE` takes a shared row lock, which blocks until the owning transaction commits or rolls
back. That blocking is the point — it is how a duplicate waits for the original without polling
and without a sleep loop.

Two outcomes, distinguished by whether a row comes back at all:

- **One row** — the owner committed. `state` is `COMPLETED` and `response` holds the result. Return
  that instead of writing anything.
- **No rows** — the owner rolled back, which rolled its `INSERT` back too. The key is free. Loop
  and try to claim it again, at most `MAX_CLAIM_ATTEMPTS = 3` times so two clients cannot ping-pong
  forever.

**Step 3: fingerprint check.** If the key exists but its `request_fingerprint` differs from this
request's, raise `IDEMPOTENCY_KEY_REUSED`.

The fingerprint is SHA-256 over a canonical JSON encoding — sorted keys, no incidental whitespace
— of the semantically meaningful arguments. It deliberately excludes incidental fields such as
`source`, so a retry that differs only in free-text metadata still replays.

Why fail rather than return the stored answer? Because returning a result for a *different*
request answers a question the caller never asked, and does it silently. Failing loudly is better.

**Step 4: complete.** After the write, in the same transaction:

```python
row.state = "COMPLETED"
row.response = {"memory_id": ..., "revision_no": ...}
row.completed_at = (await session.execute(select(func.now()))).scalar_one()
```

Because the claim, the write, and the completion all commit atomically, there is no state in which
the key says `COMPLETED` but the memory does not exist.

**Step 5: replay.** On a replay, `_replay_memory` re-reads the memory the stored response points
at and returns it with `outcome="idempotent_replay"`. The caller gets a real, current view of the
memory it created, not a stale snapshot.

**Step 6: garbage collection.** Keys carry `expires_at` defaulting to 7 days.
`idempotency.purge_expired` deletes them in bounded batches of 1000, because an unbounded `DELETE`
on a busy table takes a long lock and becomes its own outage. It is invoked by
`memhub-admin gc` — nothing schedules it automatically.

## 12.3 Which operations use it

| Operation | Key | Why |
|---|---|---|
| `memory_remember` | optional, strongly recommended | naturally non-idempotent; the retry hazard is real |
| `memory_revise` | optional | already safe without one, but the key turns a confusing answer into a clear one |
| `memory_forget` | not used | naturally idempotent: tombstoning a tombstoned memory returns `already_forgotten` |
| `project_use` | not used | naturally idempotent via the unique slug |
| reads | not applicable | no side effects |

**The subtle point about `revise`, and it is the actual insight.** `memory_revise` does not *need*
an idempotency key for correctness — the compare-and-set already makes a duplicate write
impossible, because a retry of a committed revise fails the version check. But without a key, that
retry is reported as a **conflict**, which tells the caller it lost a race it actually won. The key
turns that into a clean `idempotent_replay`.

This is why the ordering in `services/memories.revise` matters: the idempotency claim happens
**before** the compare-and-set.

Knowing when idempotency is required for *correctness* versus for *ergonomics* is the thing to say
in an interview.

## 12.4 Tests

`tests/concurrency/test_idempotency.py`, seven tests:

| Test | What it proves |
|---|---|
| `test_fifty_concurrent_retries_create_one_memory` | 50 simultaneous retries with the same key → exactly 1 `created`, 49 `idempotent_replay`, all 50 see the same `memory_id` and `revision_no`, exactly 1 row in `memories`. |
| `test_different_keys_create_different_memories` | the control case: 20 different keys → 20 memories. Without this, an implementation that ignored the key and always returned the first memory would pass the test above. |
| `test_identical_writes_deduplicate_even_without_a_key` | 10 identical writes with no key collapse to one memory — by *deduplication*, not idempotency. |
| `test_without_a_key_a_changed_retry_still_duplicates` | 10 writes with trivially different wording produce 10 memories. Honest about the limit of dedup, and the reason idempotency exists separately. |
| `test_key_reuse_with_a_different_payload_is_rejected` | `IdempotencyKeyReusedError`. |
| `test_keys_are_scoped_to_a_project` | the same key in two projects creates two memories. |
| `test_concurrent_revise_retries_replay_rather_than_conflict` | 50 concurrent revises with the same key → 1 `revised` and 49 `idempotent_replay`, **not** 49 conflicts. |

Also `test_retry_with_an_idempotency_key_replays` in the protocol suite.

---

# PART 13 — IDEMPOTENCY VS DEDUPLICATION

These two get conflated constantly. Keeping them apart is a design point, and this repository
keeps them apart deliberately — separate keys, separate tables, separate outcomes, separate tests.

## 13.1 The one-line difference

- **Idempotency** — *one* client retries *the same request*.
- **Deduplication** — *two different* clients independently assert *the same fact*.

## 13.2 Side by side

| | Idempotency | Deduplication |
|---|---|---|
| Trigger | one client retrying after a dropped connection | two clients each deciding the same thing is worth recording |
| Identified by | `client_request_id`, opaque, caller-generated | a SHA-256 hash of the normalised content |
| Table | `idempotency_keys` | `memory_dedup_keys` |
| Key | `(project_id, client_request_id)` | `(project_id, hash_version, content_hash)` |
| SQL file | `claim_idempotency.sql`, `wait_idempotency.sql` | `claim_dedup_key.sql` |
| Correct response | replay the original stored response | return the existing memory, record an attestation |
| `outcome` returned | `idempotent_replay` | `deduplicated` |
| Side effect | none | a row in `memory_attestations`, corroboration count goes up |
| Failure if absent | a network retry creates a duplicate memory | the corpus fills with near-identical rows |
| Sensitive to wording? | no — the same request may be rephrased and still replay, because the fingerprint excludes incidental fields; but a genuinely different payload is refused | yes — one different word and the hash differs, so it does not collapse |
| Can it help the other's case? | no | no |

## 13.3 Why neither replaces the other

**Deduplication cannot cover a retry.** Its key is the content. A client rebuilding a request after
a dropped connection may phrase it slightly differently — "The job queue runs on PostgreSQL." vs
"The job queue runs on PostgreSQL (attempt 2)." — and the hashes differ, so two memories are
created. `test_without_a_key_a_changed_retry_still_duplicates` asserts exactly this.

**Idempotency cannot cover two clients.** Its key is caller-generated. Claude Desktop and Cursor
each generate their own; they will never match. Only content addressing can notice that they said
the same thing.

## 13.4 The normalisation and the hash

`src/memhub/domain/normalize.py`:

```python
def normalize_content(content: str) -> str:
    normalized = unicodedata.normalize("NFKC", content)   # 1
    normalized = normalized.casefold()                    # 2
    normalized = _WHITESPACE.sub(" ", normalized).strip() # 3
    return _TRAILING_PUNCTUATION.sub("", normalized)      # 4
```

1. **NFKC Unicode normalisation** — so visually identical text spelled with different code points
   compares equal.
2. **Casefold** — a more aggressive lowercase that handles non-ASCII case rules `lower()` does not.
3. **Collapse internal whitespace** to a single space, then strip the ends.
4. **Strip trailing sentence punctuation.**

Then `content_hash` is `sha256(normalized.encode("utf-8")).digest()` — 32 bytes.

**The normaliser is deliberately conservative.** It collapses formatting noise and nothing else:

```
"PostgreSQL is the queue"  ==  "postgresql   is the queue."     -> same hash
"PostgreSQL is the queue"  !=  "We use PostgreSQL for queueing" -> different hashes
```

Anything smarter than that is semantic similarity, and merging on a similarity score would
silently destroy real distinctions.

**Why SHA-256 rather than a fast non-cryptographic hash.** This value backs a uniqueness
constraint. A collision would silently merge two distinct project facts. 32 bytes per revision is
not a cost worth optimising against that risk.

**`hash_version`** is stored beside every hash and is part of the dedup key, so a future change to
the normaliser can be rolled out alongside the old one instead of as a big-bang re-hash.

## 13.5 Attestation and corroboration

**Diagram 11 — deduplication**

```
  Claude Desktop:  remember("The job queue runs on PostgreSQL.")
      -> hash H not taken -> memory M created, dedup key H -> M
      -> attestation (M, "claude-desktop")     attestation_count = 1

  Cursor:          remember("The job queue runs on PostgreSQL.")
      -> speculative insert of M2, then claim H -> ZERO ROWS
      -> ROLLBACK TO SAVEPOINT   (M2 leaves no trace)
      -> look up who holds H -> M
      -> attestation (M, "cursor")             attestation_count = 2
      -> return M with outcome = "deduplicated"
```

A deduplicated write is **evidence, not a nuisance**. When Cursor states something Claude Desktop
already stored, that second independent assertion is a signal the fact is load-bearing. Instead of
discarding the event, the system records it.

**Honest note.** `docs/architecture.md` §7.4 describes an attestation-based ranking prior
(`w_att`). That prior is **not implemented** — `src/memhub/retrieval/ranking.py` applies importance,
recency, and type weight only. `attestation_count` is returned to the caller and shown in
`memory_history`, but it does not currently affect ranking.

## 13.6 Dedup key lifecycle

The dedup key is deleted when a memory is superseded or forgotten
(`TruthRepository.release_dedup_keys`). Without that, retiring "The job queue runs on Redis." would
permanently block anyone from ever storing that sentence again — including legitimately, if the
decision were later reversed. Tested by `test_a_retired_sentence_can_be_asserted_again`.

---

# PART 14 — TRANSACTIONS

## 14.1 The words

**Transaction** — a group of database statements treated as one unit.

**Atomic** — all-or-nothing. Either every statement takes effect, or none does. There is no
in-between state visible to anyone else.

**BEGIN** — start a transaction.

**COMMIT** — make its changes permanent and visible to others. All of them become visible at the
same instant.

**ROLLBACK** — discard its changes as if it never ran.

## 14.2 The problem, with the running example

Superseding involves two changes:

```
1. create memory B: "The job queue runs on PostgreSQL. Redis was removed."
2. mark memory A:   "The job queue runs on Redis."   ->  SUPERSEDED
```

**Without an atomic transaction, and the process crashes between them:**

```
  memory B exists, ACTIVE
  memory A exists, still ACTIVE
```

Now a search for "queue" returns **both**, presented as equally current, and they contradict each
other. Nothing in the system knows this is wrong, and nothing will ever repair it.

**Or the other order, crashing in between:**

```
  memory A is SUPERSEDED... by nothing
  memory B does not exist
```

Now the project has no opinion about its queue at all, and `memories.superseded_by_id` is NULL on
a SUPERSEDED row — a state a `CHECK` constraint in this schema actually forbids, so this particular
version cannot even be written.

**Diagram 12 — transactions**

```
   WITHOUT one transaction                WITH one transaction

   INSERT B          -> visible           BEGIN
       *** crash ***                        INSERT B
   UPDATE A          -> never ran           UPDATE A -> SUPERSEDED
                                          COMMIT

   result: A and B both ACTIVE,           result: both changes appear at the
           contradicting each other               same instant, or neither does
```

## 14.3 Where it is implemented

`src/memhub/persistence/engine.py`:

```python
@asynccontextmanager
async def session_scope(factory):
    async with factory() as session:
        try:
            yield session
        except BaseException:
            await session.rollback()
            raise
        else:
            await session.commit()
```

Every MCP handler wraps its whole body in this. So one tool call is exactly one transaction.

For `memory_remember` with `supersedes`, that one transaction contains: the idempotency claim, the
memory insert, the revision insert, the dedup key claim, the supersession updates, the dedup key
releases, the outbox row, the attestation upsert, every audit row, and the idempotency completion.

**They all become visible at the same instant.** No client can ever observe the new memory existing
while the old one is still ACTIVE.

## 14.4 Savepoints

A **savepoint** is a marker inside a transaction that you can roll back *to*, undoing only what
happened after it, without abandoning the whole transaction.

`remember` uses one:

```python
savepoint = await session.begin_nested()
memory, revision = await repo.create(...)              # speculative
claimed = await truth.claim_dedup_key(...)
if claimed is None:
    await savepoint.rollback()                          # undo only the speculative insert
    ...                                                 # dedup path
else:
    await savepoint.commit()
```

Why not roll back the whole session on a dedup hit? Because the enclosing transaction belongs to
the caller and may already contain an idempotency claim that must survive — and because a caller
managing its own transaction would be broken by it.

## 14.5 Tests

- `test_a_failed_write_leaves_nothing_behind` — a `remember` that touches five tables, then a
  rollback: all five counts are zero afterwards.
- `test_a_constraint_violation_does_not_leave_an_orphan` — a rejected insert leaves no partial rows.
- `test_a_rolled_back_write_leaves_no_job` — the outbox row disappears with the write. This is the
  test that distinguishes a transactional outbox from the dual-write bug.

---

# PART 15 — DATABASE CONSTRAINTS AND INVARIANTS

## 15.1 What an invariant is

**Invariant.** A rule that must always remain true, no matter what code runs.

Example: "a memory has exactly one current revision." Not usually. Not if the code is right.
**Always.**

## 15.2 Why enforcing rules in PostgreSQL beats enforcing them in Python

Four reasons:

1. **A Python check can be skipped.** A new code path, a bug, a direct SQL statement from a script,
   or a migration can all bypass it. A constraint cannot be bypassed by anything that goes through
   the database — which is everything.
2. **A Python check is not atomic.** "Check that no current revision exists, then insert one"
   races. A unique index does the check and the write in one operation.
3. **A constraint documents itself in the schema.** Anyone reading the DDL sees the rule.
4. **A constraint catches the case you did not think of.** The compare-and-set is the primary
   mechanism preventing two revision 5s. The primary key is the backstop that holds even if the
   compare-and-set is written wrongly.

The pattern used throughout this repository: **the constraint is the guarantee; the Python
validator is the good error message.** Both exist for the same rule. `validate_content` tells you
"content is 9102 characters; the limit is 8192". `ck_memory_revisions_content_length` makes sure
that limit is real.

## 15.3 The invariants, verified

Fourteen are claimed in `docs/architecture.md` §13. Here is what is actually in the schema and in
the tests.

| # | Invariant | Enforced by | Where | Test |
|---|---|---|---|---|
| 1 | Exactly one current revision per memory | database | `UNIQUE (memory_id) WHERE is_current` (`uq_memory_revisions_memory_id`) | `test_two_current_revisions_are_rejected` |
| 2 | Revision numbers unique per memory | database | `PRIMARY KEY (memory_id, revision_no)` | `test_duplicate_revision_number_is_rejected` |
| 3 | A stale write cannot overwrite the current revision | application SQL | `cas_revise.sql` predicate | `test_exactly_one_of_fifty_writers_wins` |
| 4 | A memory can only be superseded within its own project | database | composite FK `(superseded_by_id, project_id) → (id, project_id)` | `test_cross_project_supersession_is_unrepresentable` |
| 5 | A memory cannot supersede itself | database | `CHECK (superseded_by_id IS DISTINCT FROM id)` | `test_memory_cannot_supersede_itself` |
| 6 | Status columns and their timestamps agree | database | paired CHECKs on `superseded_at` / `deleted_at` | `test_superseded_status_requires_a_target` |
| 7 | An idempotency key produces at most one write | database | `PRIMARY KEY (project_id, client_request_id)` + `ON CONFLICT DO NOTHING` | `test_fifty_concurrent_retries_create_one_memory` |
| 8 | No two active memories share normalised content | database | `memory_dedup_keys` primary key | `test_identical_writes_deduplicate_even_without_a_key` |
| 9 | Superseded / deleted / expired never appear in normal retrieval | application, one function | `retrieval/filters.py::current_revisions` | `test_the_retired_fact_leaves_retrieval_entirely` |
| 10 | Every revision belongs to a memory in the same project | database | composite FK on `memory_revisions` | invariant-suite query |
| 11 | Every TASK has an expiry | database | `CHECK (type <> 'TASK' OR expires_at IS NOT NULL)` | `test_task_without_expiry_is_rejected` |
| 12 | Content is never destroyed by a normal operation | application design | append-only log; only `memhub-admin purge` deletes | `test_content_survives_forgetting`, `test_purge_actually_erases_content` |
| 13 | A vector never exists without its revision | database | FK `ON DELETE CASCADE` | `test_purge_clears_every_derived_table` |
| 14 | Every timestamp comes from the database clock | application SQL | `now()` server-side defaults, `SELECT now()` where needed | `test_expiry_uses_the_database_clock` |

Nine of the fourteen are schema-level. That ratio is the point.

## 15.4 The invariant suite

`tests/integration/test_invariants.py` defines eight SQL queries, each of which must return **zero
rows**:

```sql
-- more than one current revision per memory
SELECT memory_id FROM memory_revisions WHERE is_current
GROUP BY memory_id HAVING count(*) > 1;

-- the current-revision pointer disagrees with the log
SELECT m.id FROM memories m
JOIN memory_revisions r ON r.memory_id = m.id AND r.is_current
WHERE r.revision_no <> m.current_revision_no;

-- a gap in a revision sequence
SELECT memory_id FROM memory_revisions
GROUP BY memory_id HAVING max(revision_no) <> count(*);

-- a memory with no current revision
-- superseded without a target
-- a revision in a different project from its memory
-- a supersession crossing a project boundary
-- a TASK with no expiry
```

`assert_invariants_hold(session)` runs all eight and is called at the end of **every concurrency
test**. That is what turns "exactly one writer won" into "exactly one writer won *and the database
is still consistent*".

Note the third query. `max(revision_no) <> count(*)` catches a **gap** in the sequence — revisions
1, 2, 4 with no 3. That would make history unreadable and would mean the increment logic had gone
wrong somewhere.

## 15.5 The constraint tests bypass the service layer

Every test in `TestConstraintsAreDeployed` writes **raw SQL**, on purpose. The point is what
happens when application logic is wrong or absent. If these tests went through the service layer,
they would be testing the service layer's checks, not the database's.

They also serve a second purpose: they prove the constraints are actually **deployed**. A
migration that silently failed to create an index would pass every other test in the suite and
fail these.

## 15.6 The single most interesting constraint

```python
ForeignKeyConstraint(
    ["superseded_by_id", "project_id"],
    ["memories.id", "memories.project_id"],
)
```

This is a **composite foreign key on a self-referencing table**. It says: the memory you point at
as your successor must exist, **and it must have the same `project_id` as you**.

The consequence is stronger than a rule: cross-project supersession is not *prevented*, it is
**unrepresentable**. There is no sequence of SQL statements that produces that state. It is not a
`WHERE` clause someone might forget to write.

It only works because `memories` also carries `UNIQUE (id, project_id)` — a foreign key must
reference a unique constraint, and the primary key on `id` alone is not enough.

`test_cross_project_supersession_is_unrepresentable` proves it by writing the forbidden `UPDATE`
directly and asserting the error names
`fk_memories_superseded_by_id_project_id_memories`.

---

# PART 16 — RETRIEVAL FROM ZERO

## 16.1 What does retrieval mean?

**Retrieval** is finding the memories worth returning for a particular request.

## 16.2 Why not simply return everything?

Three reasons, in increasing importance:

1. **Size.** A mature project's corpus is hundreds to tens of thousands of memories. That does not
   fit in a model's context window.
2. **Cost.** The caller is spending tokens. Irrelevant memories displace useful ones.
3. **Correctness.** This is the one that matters. Returning everything means returning the retired
   "The job queue runs on Redis." next to the current "The job queue runs on PostgreSQL." The
   caller now has two contradictory statements and no way to tell which is current. That is worse
   than returning nothing.

Retrieval here therefore has two separate jobs, and this project keeps them strictly separate:

```
  Job 1  — CORRECTNESS:  which memories are even allowed to be returned?
  Job 2  — RELEVANCE:    of those, which best answer this question?
```

Job 1 runs first and is non-negotiable. Job 2 is tuning on top of a set that is already
guaranteed correct.

## 16.3 The two ways to find text

**Keyword / lexical / full-text search** — match on the words themselves. If the memory contains
"PostgreSQL" and the query contains "PostgreSQL", it matches.

**Semantic / vector search** — match on meaning. If the memory says "JWTs are validated at the
edge" and the query says "jwt", a lexical search may miss it (the stemmer does not reduce "JWTs"
to "jwt"), but a semantic search finds it, because the two texts sit close together in the
model's vector space.

Both are implemented here, and combined. Parts 17 through 19 build them up.

---

# PART 17 — POSTGRESQL FULL-TEXT SEARCH

## 17.1 The problem

You want to find memories containing a word. The naive answer is:

```sql
WHERE content LIKE '%postgres%'
```

That has three problems. It cannot use an index effectively with a leading wildcard, so it reads
every row. It does not know that "queue" and "queues" and "queueing" are the same word. And it has
no notion of how *well* a row matches — it is a yes/no test.

## 17.2 The words

**Keyword search / lexical search** — matching on the actual words. "Lexical" means "about words".

**Token** (in this context) — one word unit extracted from text.

**Tokenization** — splitting text into those units.

**Stemming** — reducing a word to its root form so related forms match each other:
`queueing → queue`, `running → run`. PostgreSQL's English configuration uses the Snowball stemmer.

**Lexeme** — PostgreSQL's word for a normalised, stemmed token.

**tsvector** — a PostgreSQL data type holding a document's lexemes with their positions. For
example, `to_tsvector('english', 'The job queue runs on Redis.')` produces roughly:

```
'job':2 'queue':3 'redi':6 'run':4
```

Note what happened: stop words ("the", "on") were dropped, everything was lowercased, "runs"
became "run", and "Redis" became "redi". Positions were kept, which is what lets proximity ranking
work.

**tsquery** — the same processing applied to a search string, producing a query expression.

**GIN index** — Generalised Inverted Index. It maps each lexeme to the list of rows containing it,
which is exactly the shape you want when one row contains many values.

**Ranking** — assigning a score so results can be ordered.

## 17.3 How this repository does it

**The stored vector.** `memory_revisions.content_tsv` is a **generated column**:

```python
content_tsv: Mapped[str] = mapped_column(
    TSVECTOR,
    Computed("to_tsvector('english', content)", persisted=True),
    nullable=False,
)
```

PostgreSQL computes and stores it. There is no code path that writes content without also
producing its search vector, so the two cannot drift. A trigger or an application-side write
could; a generated column cannot.

**The index.**

```python
Index("ix_memory_revisions_content_tsv", "content_tsv",
      postgresql_using="gin", postgresql_where=text("is_current"))
```

It is **partial** — only current revisions. At steady state most rows are superseded revisions
that no search will ever read, so indexing them would inflate the index and slow every insert for
no benefit.

**Parsing the query.** `src/memhub/retrieval/lexical.py`:

```python
def to_query(query: str) -> ColumnElement[str]:
    return func.websearch_to_tsquery(literal_column("'english'::regconfig"), query)
```

`websearch_to_tsquery` rather than `to_tsquery`, and this choice matters: a language model
composes the query string, and it will write things like `"task queue" -redis` or
`postgres or sqlite`. `to_tsquery` raises a syntax error on anything that is not a strict boolean
expression, which would turn an ordinary question into a tool failure. `websearch_to_tsquery`
accepts what a search box accepts and never raises.

**Matching.**

```python
def matches(query: str) -> ColumnElement[bool]:
    return MemoryRevision.content_tsv.op("@@")(to_query(query))
```

`@@` is the "matches" operator. Because it is applied to the **stored** column, the GIN index can
be used directly. Computing `to_tsvector()` at query time instead would force a sequential scan
over every row.

**Scoring.**

```python
def relevance(query: str) -> ColumnElement[float]:
    return func.ts_rank_cd(MemoryRevision.content_tsv, to_query(query), 32).cast(Float)
```

`ts_rank_cd` rather than `ts_rank`: the `cd` stands for **cover density**, which rewards query
terms appearing *near each other*. Memories are one or two sentences, so proximity is close to the
whole signal — a memory saying "PostgreSQL is the task queue" should beat one that mentions
PostgreSQL in one clause and queues in another.

The `32` is a normalisation flag meaning "divide by rank + 1", which maps the unbounded raw score
into `[0, 1)`.

## 17.4 The scale caveat, which matters later

`ts_rank_cd` is **unbounded and corpus-dependent**. Its ordering is meaningful within a single
query and meaningless across queries. This module therefore never thresholds on the value, never
compares it between queries, and never adds it to anything on a different scale.

Remember this. It is the entire reason Part 22 exists.

## 17.5 The two-stage matching fallback

This is a real defect the evaluation harness found on its first run, and the fix is worth
understanding.

`websearch_to_tsquery` joins bare terms with **AND**. So the query `"connection pool size"`
demands all three lexemes, and misses a memory saying "connection pooling is bounded at 10" —
which has no "size". Natural-language questions are mostly like this. `"what queue does this
project use"` demands *queue* AND *project* AND *use* together, and returns nothing.

Measured: the strict all-terms baseline scored **nDCG 0.478**, and queries as ordinary as
"migration rules" returned nothing at all.

The fix, in `services/memories.search`: if the strict query finds nothing, retry with any-term
matching.

```python
def any_term_query(query: str) -> str | None:
    if any(marker in query for marker in ('"', "-", " or ", " OR ")):
        return None                       # explicit syntax: do not rewrite
    terms = query.split()
    if len(terms) < 2:
        return None
    return " or ".join(terms)
```

Three details worth noting:

- **Precision is preserved for queries that do match strictly.** The widening only happens when the
  alternative is returning nothing.
- **Relevance still orders the result.** `ts_rank_cd` ranks a memory matching three terms above one
  matching a single term, so widening the net does not flatten the ordering.
- **Queries using explicit syntax are never rewritten.** A caller who wrote `queueing -redis` meant
  it, and turning that into an OR would invert their intent.

The result: nDCG went from **0.478 to 0.802** and recall from **0.468 to 0.817**, and stale
inclusion stayed at exactly **0.000** — loosening the match did not loosen the correctness
guarantee, because suppression is structural rather than a ranking effect.

The strategy used is reported back in `SearchResult.match_strategy` as `all_terms` or `any_term`,
so a caller reading loosely matched results knows that is what they are. (As noted in Part 5, that
field is not currently forwarded into the MCP `SearchOut` schema.)

## 17.6 The ranking priors

Relevance answers "does this text match the query". It does not answer "is this worth telling the
agent about". `src/memhub/retrieval/ranking.py` applies three multiplicative priors on top:

```python
final(d) = ts_rank_cd(...)
         * (1 + 0.5 * importance/100)          # importance prior
         * (1 + 0.3 * 0.5^(age_days/half_life)) # recency prior
         * type_weight                          # 1.15 / 1.10 / 1.00 / 0.95
```

with half-lives by type:

| Type | Half-life | Type weight | Reasoning |
|---|---|---|---|
| CONSTRAINT | 3650 days | 1.15 | a constraint from two years ago still binds |
| DECISION | 365 days | 1.10 | decisions age slowly |
| FACT | 180 days | 1.00 | the neutral case |
| TASK | 7 days | 0.95 | "currently implementing X" is stale within days |

**Why multiplicative rather than additive.** A prior should *scale* relevance, not substitute for
it. A memory that does not match the query at all scores zero, and no amount of importance can
rescue it.

**Why the recency clock is the database's.** `func.now()` inside the query, never Python's
`datetime.now()`. Otherwise two server processes could disagree about how old a memory is, and
which results came back would depend on which process answered.

**Honest note.** These weights are untuned. The module docstring says so. They were set before the
evaluation harness existed, and were deliberately not fitted afterwards.

## 17.7 Why exact technical terminology benefits from lexical search

A project's knowledge is full of identifiers: `PostgreSQL`, `SKIP LOCKED`, `BRPOP`, `pgvector`,
`alembic`. These are exactly the words an embedding model handles worst — they are rare, they are
often split into several sub-word pieces, and their meaning is a name rather than a concept.
Lexical search handles them perfectly, because it matches the literal token.

That is half the argument for hybrid retrieval. The other half is in Part 18.

## 17.8 The `IS TRUE` trap

This is a genuinely good interview story, and it has its own test file.

`src/memhub/retrieval/filters.py` ends with:

```python
MemoryRevision.is_current,      # NOT .is_(True)
```

with a comment explaining why. The full-text index is **partial**: `... WHERE is_current`.
PostgreSQL will only use a partial index if it can *prove* the query's predicate implies the
index's predicate. It proves that for a bare boolean column. It does **not** prove it for
`is_current IS TRUE`, because `IS TRUE` is null-safe and is therefore a different expression.

Written the wrong way, the index becomes unusable at any corpus size. Nothing fails. Results are
still correct. Tests stay green. The cost appears only once the corpus is large enough to matter,
by which point nobody connects it to a two-word change made months earlier.

It was verified empirically: at 20k rows, `IS TRUE` produced a Seq Scan and the bare column
produced a Bitmap Index Scan.

The guard is `tests/unit/test_filter_sql.py`, which compiles the query and asserts on the SQL
string. Its docstring explains why that is the right place to check: a latency test cannot catch it
on a small corpus, and a plan assertion cannot either, because at 10k rows PostgreSQL legitimately
prefers other access paths whether or not the index is usable.

**Diagram 13 — full-text search**

```
  "The job queue runs on Redis."
        |
        | to_tsvector('english', content)      <- generated column, by PostgreSQL
        v
  'job':2 'queue':3 'redi':6 'run':4           <- stored as content_tsv
        |
        | GIN index, partial: WHERE is_current
        v
  lexeme -> list of rows containing it

  query "redis"
        |
        | websearch_to_tsquery('english', ...)
        v
  'redi'
        |
        | content_tsv @@ tsquery     -> which rows match
        | ts_rank_cd(...)            -> how well
        v
  ordered candidates
```

---

# PART 18 — SEMANTIC SEARCH

## 18.1 The problem

Relevant text may use different wording.

A memory says: *"JWTs are validated at the API gateway."*
The query is: *"jwt"*.

Lexical search misses this, and the reason is specific and measured. PostgreSQL's English Snowball
stemmer does not reduce `JWTs` to `jwt`. In this project's evaluation set, query `q22` — literally
`"jwt"` — scored **nDCG 0.000** under full-text search alone. That failing case was recorded
deliberately as motivation for building semantic search.

## 18.2 The idea

Turn text into numbers that represent its meaning, so that "close in meaning" becomes "close in
space".

```
   text
     |
     | embedding model
     v
   vector of 384 numbers
     |
     | stored in PostgreSQL by pgvector
     v
   similarity search: which stored vectors are nearest the query's vector?
```

## 18.3 The words

**Embedding** — a list of numbers produced by a model, representing a piece of text's meaning.
Two texts that mean similar things get vectors that point in similar directions.

**Vector** — that list of numbers.

**Dimension** — how many numbers. Here **384**, because the model is `BAAI/bge-small-en-v1.5`.

**Unit-normalised** — scaled so the vector's length is exactly 1. The `EmbeddingPort` contract
requires this, so that cosine distance is the only thing the index has to compute. An adapter
returning unnormalised vectors would still "work" and would quietly rank by magnitude as much as
by direction.

**pgvector** — a PostgreSQL extension adding the `vector` column type, distance operators, and
vector index types. Enabled in migration `0006` with `CREATE EXTENSION IF NOT EXISTS vector`.

**Cosine similarity** — a measure of the angle between two vectors, from -1 (opposite) through 0
(unrelated) to 1 (identical direction).

**Cosine distance** — `1 - cosine similarity`, so 0 to 2. **Smaller means more similar.** In
pgvector this is the `<=>` operator.

**Nearest neighbour** — the stored vector closest to the query's vector.

**Approximate nearest neighbour (ANN)** — a search that finds *probably* the closest vectors,
much faster than comparing against every one. Exact search is O(n) in the number of stored vectors;
ANN is far cheaper, at the cost of occasionally missing a true nearest neighbour.

**HNSW — Hierarchical Navigable Small World** — the ANN index type used here. Intuitively: it
builds a graph where each vector is linked to some of its neighbours, in several layers. The top
layer is sparse and lets you jump across the space quickly; lower layers are dense and let you
refine. A search enters at the top, greedily walks towards the query, then descends. It is fast,
it has good recall, and it does not need the whole dataset up front.

```python
Index("ix_memory_embeddings_hnsw", "embedding",
      postgresql_using="hnsw",
      postgresql_ops={"embedding": "vector_cosine_ops"})
```

## 18.4 Why "nearest" does not mean "actually relevant"

This is the most important paragraph in this part.

Approximate nearest-neighbour search returns the *k* closest vectors **whether or not anything is
close**. Ask it about Kubernetes over a corpus that never mentions Kubernetes, and the index will
cheerfully return the ten least-unrelated memories it has, with every appearance of confidence.

There is no "no match" answer built into the mechanism. You have to add one.

## 18.5 The threshold, and how it was chosen

```python
MAX_COSINE_DISTANCE = 0.35
```

This value was **measured, not guessed**, and the evidence is committed in
`docs/eval/threshold-sweep.md`.

The first hybrid implementation had no threshold at all. Measured against full text:

| | nDCG@10 | Recall@10 | Precision@10 | Empty for unanswerable |
|---|---|---|---|---|
| full text (baseline) | 0.803 | 0.817 | 0.691 | 0.667 |
| hybrid, no threshold | **0.881** | 0.876 | **0.113** | **0.000** |

nDCG went **up**. Precision **collapsed by a factor of six**, and *every* unanswerable query
started returning ten results.

Reporting "hybrid improves nDCG from 0.803 to 0.881" and stopping there would have been true and
badly misleading.

The sweep, over the same 200-memory corpus and 34 graded queries:

| max cosine distance | nDCG@10 | Recall@10 | Precision@10 | Stale | Empty for unanswerable |
|---|---|---|---|---|---|
| 1.00 (none) | 0.881 | 0.876 | 0.113 | 0.000 | 0.000 |
| 0.60 | 0.881 | 0.876 | 0.113 | 0.000 | 0.000 |
| 0.50 | 0.881 | 0.876 | 0.113 | 0.000 | 0.000 |
| 0.45 | 0.876 | 0.860 | 0.144 | 0.000 | 0.000 |
| 0.40 | 0.873 | 0.844 | 0.371 | 0.000 | 0.000 |
| **0.35** | **0.853** | **0.828** | **0.671** | **0.000** | **0.333** |
| 0.30 | 0.659 | 0.634 | 0.624 | 0.000 | 0.667 |
| 0.25 | 0.468 | 0.468 | 0.452 | 0.000 | 0.667 |
| 0.20 | 0.478 | 0.468 | 0.484 | 0.000 | 0.667 |

How to read it:

- **Above 0.50 nothing changes** — with this model almost no pair of unrelated sentences is that
  far apart, so the threshold is not binding.
- **Between 0.45 and 0.35, precision climbs from 0.144 to 0.671 while nDCG gives up 0.023.** That
  is the whole trade, and it is heavily in favour of tightening.
- **Below 0.30 it falls apart** — the threshold starts excluding genuine matches, and by 0.25 the
  numbers converge on the lexical-only baseline, which is what you would expect when one of two
  retrievers has effectively been switched off.

**0.35 is the knee of the curve.** It keeps most of the ranking gain and recovers nearly all of
the precision.

## 18.6 The filtered-ANN problem

This is the genuinely interesting engineering detail in the semantic path.

An HNSW index scan collects roughly `ef_search` nearest neighbours from the graph, and *then* the
planner applies the `WHERE` clause. With a restrictive filter — one project out of many, active
only — a scan can return 40 neighbours of which 3 survive filtering. The query looks like it
worked and quietly missed most of what it should have found.

pgvector 0.8 added iterative index scans for exactly this: the scan continues until enough rows
survive filtering. This project turns it on per query:

```python
await session.execute(text("SET LOCAL hnsw.iterative_scan = relaxed_order"))
```

`relaxed_order` rather than `strict_order`, deliberately: strict ordering guarantees results come
back in exact distance order at noticeably higher cost, and it is not needed here, because the
vector ranking is one input to a rank-fusion step that reorders everything anyway.

`SET LOCAL` means it applies only for the current transaction, because it costs more when the
filter is not selective.

## 18.7 The query

`src/memhub/retrieval/semantic.py::search_by_vector`:

```sql
SELECT m.id, e.embedding <=> CAST(:vec AS vector) AS distance
  FROM memories m
  JOIN memory_revisions r ON r.memory_id = m.id AND r.project_id = m.project_id
  JOIN memory_embeddings e ON e.memory_id = r.memory_id
                          AND e.revision_no = r.revision_no
                          AND e.model = :model
 WHERE m.project_id = :pid
   AND m.status = 'ACTIVE'
   AND (m.expires_at IS NULL OR m.expires_at > now())
   AND r.is_current
   AND (e.embedding <=> CAST(:vec AS vector)) <= :max_distance
 ORDER BY distance
 LIMIT :limit
```

Notice lines 5 to 8. **They are the same stage-0 conditions as every other retrieval path.** A
retired memory is no more reachable through a vector than through a keyword.

Notice `e.model = :model`. Vectors from different models occupy different spaces and their
similarities are not comparable. Mixing them would not raise an error — it would silently return
nonsense rankings, which is far worse. So every stored vector records its model and every query
filters on it.

Notice it is an **inner join** to `memory_embeddings`. A memory without a vector is not a semantic
candidate at all. That is the correct behaviour in the window after a write and before the outbox
catches up: the memory is simply absent from this retriever, contributes nothing to fusion, and is
still findable by full text.

Notice it returns `(memory_id, distance)` rather than full rows. The only consumer is rank fusion,
which needs an ordering and nothing else. Hydrating full rows here would fetch content for
candidates fusion may well discard.

## 18.8 Which model, and why local

`BAAI/bge-small-en-v1.5`, 384 dimensions, run through `fastembed`.

- **Local rather than hosted.** A hosted embedding API would make the demo depend on a network and
  an API key, would send every memory to a third party — a strange property for a system whose job
  is holding a developer's private project knowledge — and would put an external outage on the path
  to semantic search.
- **fastembed rather than sentence-transformers.** It runs ONNX on CPU: roughly a 50 MB model
  instead of a multi-gigabyte torch install.
- **Optional.** It is an extra: `pip install -e ".[local-embeddings]"`. The default
  `embedding_adapter` is `none`, so a fresh install works with no model download and search is
  full-text only. A server that tries to download a model on first use is a server that fails to
  start on a machine without a network, for a feature that is supposed to be an enhancement.
- **Loaded lazily.** `LocalEmbedder._load()` imports fastembed and constructs the model on first
  use, not at construction. Constructing the adapter must never be what blocks a server from
  starting.

## 18.9 The deterministic fake

`src/memhub/embeddings/fake.py` hashes text into a stable unit vector.

**It carries no semantic signal, and that is not a defect.** It exists so CI can exercise the
*plumbing* — the outbox, the vector column, the HNSW index, fusion, coverage reporting, retry and
backoff — with no model download, no GPU, and byte-identical results on every machine.

It cannot measure retrieval quality: two sentences meaning the same thing get unrelated vectors.
The quality numbers in `docs/eval/results.md` come from the real local adapter, and the hybrid
evaluation is marked `real_embeddings` and deselected by default.

The factory logs a **warning** when the fake is selected, because silently ranking by hash would
be very hard to notice from the outside.

**Diagram 14/15 — embeddings and pgvector retrieval**

```
   "JWTs are validated at the API gateway."
        |
        | BAAI/bge-small-en-v1.5  (fastembed, ONNX, CPU)
        v
   [0.031, -0.114, ..., 0.007]      384 numbers, length 1.0
        |
        | INSERT INTO memory_embeddings (memory_id, revision_no, model, embedding)
        v
   +----------------------------------+
   |  pgvector column vector(384)     |
   |  HNSW index, vector_cosine_ops   |
   +----------------------------------+

   query "jwt"  ->  embed  ->  query vector q
        |
        | SET LOCAL hnsw.iterative_scan = relaxed_order
        | SELECT ... WHERE <stage-0 filter>
        |             AND (embedding <=> q) <= 0.35
        | ORDER BY embedding <=> q
        v
   [(memory_id, 0.21), (memory_id, 0.29), ...]     nearest first
```

---

# PART 19 — HYBRID RETRIEVAL

## 19.1 When keyword search wins

- **Exact identifiers.** `SKIP LOCKED`, `pgvector`, `BRPOP`, `alembic`. An embedding model
  represents these poorly; a lexical index matches them exactly.
- **Rare, distinctive words.** If a memory is the only one containing "quiescence", the lexical
  index finds it instantly.
- **Negation and phrase syntax.** `queueing -redis` and `"task queue"` mean something specific.
- **Knowing when there is no answer.** If no memory contains any query term, full text correctly
  returns nothing. Measured: full text returned nothing for **0.667** of the unanswerable queries;
  hybrid for only **0.333**.

## 19.2 When semantic search wins

- **Different wording for the same idea.** The measured example is query `q22`, `"jwt"`: nDCG went
  from **0.000** under full text to **1.000** under hybrid, because the Snowball stemmer never
  matches `JWTs` to `jwt` but the embedding model puts them close together.
- **Natural-language questions** that share no exact vocabulary with the answer.

## 19.3 Where both still lose

Recorded honestly in `docs/eval/results.md`: query `q13`, `"deadlock prevention"`, stays at
**0.000** under both strategies. The matching memory describes deadlock prevention without ever
using the phrase, and 384 dimensions of a small local model do not close that gap. It is recorded
as an open case rather than smoothed over.

## 19.4 Why combining helps, in one sentence

The two retrievers fail on **different** queries, so a document that either one ranks highly gets
a chance, and a document **both** rank highly rises above documents only one of them liked.

## 19.5 Why not only full text?

Because `q22` scored 0.000, and there is a whole class of queries like it. Measured overall:
hybrid improved nDCG from 0.803 to 0.853 and recall from 0.817 to 0.828.

## 19.6 Why not only pgvector?

Three reasons, in increasing importance.

1. **Precision.** Without a distance threshold, precision was 0.113. Even with the threshold,
   hybrid is slightly *worse* on precision than full text alone (0.671 vs 0.691) and meaningfully
   worse at recognising an unanswerable question (0.333 vs 0.667).

2. **Identifiers.** Vector search is weakest exactly where a project's vocabulary lives.

3. **The one that actually matters: truth maintenance.** Nearest-neighbour search over a corpus
   containing both "The job queue runs on Redis." and "The job queue runs on PostgreSQL." returns
   **both**, ranked by similarity — and similarity has no opinion about which is true. A dedicated
   vector database also has no transactions, so the lifecycle metadata (status, revision,
   supersession) and the vector would live in two systems that could disagree. Keeping the vectors
   inside PostgreSQL means the filter and the vector scan commit together and are read under one
   snapshot.

That third point is the strongest architectural argument in the whole project. Say it as: **"the
filter and the vector search are the same query, so 'a superseded memory is never returned' is one
guarantee, not two systems that have to agree."**

## 19.7 How the implementation runs both

`src/memhub/services/retrieval.py::hybrid_search`:

```python
candidates = limit * OVERFETCH          # OVERFETCH = 3

lexical_rows = await repo.search(project_id, query=query, ..., limit=candidates)
lexical_ranking = [memory.id for memory, _ in lexical_rows]

semantic_ranking = []
try:
    vector = await _embed_query(embedder, query)
    neighbours = await semantic.search_by_vector(..., limit=candidates, max_distance=...)
    semantic_ranking = [mid for mid, _ in neighbours]
except EmbeddingError as exc:
    degraded = f"lexical_only: {exc}"
except DBAPIError as exc:
    if sqlstate != QUERY_CANCELED: raise
    degraded = "lexical_only: semantic search exceeded the statement timeout"

fused = fusion.reciprocal_rank_fusion(
    {"lexical": lexical_ranking, "semantic": semantic_ranking})[:limit]
```

Three decisions in there:

**Both retrievers over-fetch, by a factor of 3.** Fusion can only rank what it is given. Fetching
exactly `limit` from each would mean a document ranked 11th lexically and 1st semantically never
reaches fusion at all — and that document is precisely the kind hybrid retrieval exists to find.
Three is a compromise: wide enough to catch those, narrow enough that the cost stays close to a
single search.

**Semantic failure is not search failure.** If the embedder is unavailable, or nothing has been
embedded yet, the semantic ranking is empty and RRF degrades to pure lexical ordering with no
special case. The response says `degraded: "lexical_only: ..."` rather than pretending the search
was complete.

**Hydration.** Fusion returns ids. Rows for lexical hits are already in hand; a memory found only
by the semantic retriever needs one extra `repo.get`. `missing` is computed and fetched
individually.

Then `semantic_coverage` is computed — the fraction of the project's retrievable memories that
currently have a vector for this model — and returned, so a caller can tell "the semantic half saw
everything" apart from "the semantic half saw 60% of it".

**Diagram 16 — hybrid search**

```
                     query "redis"
                          |
            +-------------+--------------+
            |                            |
      lexical leg                  semantic leg
      ts_rank_cd + priors          embed -> HNSW, distance <= 0.35
      limit = 3 x N                limit = 3 x N
            |                            |
            |   BOTH compose on the SAME stage-0 filter
            |   (project, ACTIVE, not expired, is_current)
            |                            |
            v                            v
      [A, C, B, ...]              [B, A, D, ...]
            |                            |
            +-------------+--------------+
                          |
                Reciprocal Rank Fusion (k = 60)
                          |
                          v
                 top N, then hydrate rows
```

## 19.8 A discrepancy worth knowing

`docs/architecture.md` §7.4 describes RRF followed by "a small number of interpretable
multiplicative priors" — importance, recency, attestation count, type weight — applied to the
fused score, plus a `why` block in the response showing component ranks.

The implementation does something different, and simpler. The priors are applied **inside the
lexical leg**, as part of its `ORDER BY` (`ranking.final_score`). The fused RRF score has no priors
applied to it afterwards, there is no attestation prior anywhere, and there is no `why` block in
any output schema.

This is not a bug — the lexical ranking that feeds fusion is already prior-adjusted — but it is a
real difference between the document and the code, and the code is what ships.

---

# PART 20 — STALE MEMORY

## 20.1 The setup

```
  OLD:      "The job queue runs on Redis."               <- was true, is not now
  CURRENT:  "The job queue runs on PostgreSQL.
             Redis was removed."                          <- is true now
```

A user asks: **"redis"**.

## 20.2 Why the stale memory is the *better* match

Count the evidence a retrieval system has:

**Lexically.** The old memory is short and contains "Redis" — high term density, and `ts_rank_cd`
rewards exactly that. The new memory is longer and mentions Redis once, in a subordinate clause
saying it was removed. **The old memory wins on lexical relevance.**

**Semantically.** The old memory is *about* Redis. Its embedding sits close to a query embedding
for "redis". The new memory is about PostgreSQL, with a passing mention. **The old memory wins on
semantic similarity too.**

So a system that ranks by relevance — any relevance, lexical or semantic or both — puts the
**wrong answer first**.

## 20.3 The distinction that matters

> **Relevance is not the same thing as current truth.**

A retrieval system measures how well a document matches a query. It has no opinion at all about
whether the document is still true. Those are different questions, and no amount of tuning turns
one into the other.

This is the single sentence that justifies the whole project. A vector store cannot solve it — not
because its vectors are bad, but because "is this still true" is not a property of a vector.

## 20.4 Why this is worse than an ordinary ranking error

An ordinary ranking error costs the user some scrolling. This one hands a confident, well-ranked,
**false** answer to a language model, which will then act on it — write Redis code, recommend
Redis configuration, and explain to the user why Redis was chosen. The user has no signal that
anything is wrong.

And in `memory_context`, where results are injected into a token budget with nobody reading them
first, there is not even a human in the loop to notice.

---

# PART 21 — STRUCTURAL STALE-MEMORY EXCLUSION

## 21.1 Two ways to solve it

**The weak way — ranking-based suppression:**

```
   retrieve everything, including the retired memory
        |
        v
   give the retired one a lower score (multiply by 0.1, or subtract a penalty)
        |
        v
   hope it does not appear in the top N
```

**The way this project does it — structural exclusion:**

```
   exclude invalid lifecycle states from the candidate set
        |
        v
   only valid candidates remain
        |
        v
   ranking runs, over a set that is already correct
```

## 21.2 Why the ranking approach is wrong

Four reasons, and you should be able to give all four:

1. **It leaks the moment someone asks for more results.** A penalty pushes the retired memory to
   position 11. Ask for `limit=20` and it is back. The correctness of the system would depend on
   the caller's page size.

2. **The penalty has to beat an unbounded score.** `ts_rank_cd` is unbounded and
   corpus-dependent. There is no penalty factor you can prove is always large enough.

3. **It couples correctness to tuning.** Every future ranking change — a new prior, a new
   retriever, a re-weighted fusion — becomes a chance to accidentally resurrect a retired memory.
   You would need to re-verify the guarantee after every tuning change.

4. **It cannot be stated as a guarantee.** "Retired memories rank low" is not a sentence you can
   put in an API contract. "Retired memories are never returned" is.

## 21.3 The implementation: one function, one place

`src/memhub/retrieval/filters.py`:

```python
def current_revisions(project_id, *, include_retired: bool = False):
    stmt = select(Memory, MemoryRevision).join(
        MemoryRevision,
        (MemoryRevision.memory_id == Memory.id)
        & (MemoryRevision.project_id == Memory.project_id),
    )

    stmt = stmt.where(Memory.project_id == project_id)     # always, in every mode

    if include_retired:
        return stmt

    return stmt.where(
        Memory.status == MemoryStatus.ACTIVE.value,
        or_(Memory.expires_at.is_(None), Memory.expires_at > func.now()),
        MemoryRevision.is_current,
    )
```

Four conditions, each load-bearing:

**`project_id`** — namespace isolation. Never optional. It is a positional argument, so there is no
way to call this function without one. It is applied even when `include_retired=True`: isolation
is not negotiable, even for debug paths.

**`status = 'ACTIVE'`** — excludes `SUPERSEDED` and `DELETED`. **This is the line that makes the
whole demo work.** Once "The job queue runs on Redis." is retired, no ranking strategy can
resurrect it, because no ranking strategy ever sees it.

**`expires_at IS NULL OR expires_at > now()`** — expiry, evaluated at read time against the
**database** clock. Not a stored status, so it cannot be stale; not the application clock, so two
server processes cannot disagree about what has expired.

**`is_current`** — the current revision only. Superseded revisions of a live memory stay in the
append-only log for `memory_history` and are never retrieved.

## 21.4 The four lifecycle states, in the implementation's own terms

| State | Stored where | In normal retrieval? | In `memory_history`? |
|---|---|---|---|
| `ACTIVE` | `memories.status = 'ACTIVE'` | yes | yes |
| `SUPERSEDED` | `memories.status = 'SUPERSEDED'`, plus `superseded_by_id` and `superseded_at` | no | yes, with the link to what replaced it |
| `DELETED` | `memories.status = 'DELETED'`, plus `deleted_at` | no | yes |
| expired | **not a status.** Derived from `expires_at <= now()` at read time | no | yes |

The fourth row is a design decision worth explaining. A stored `EXPIRED` status would need a
background sweeper to become true, and between the moment of expiry and the moment the sweeper
runs, the column would be **lying**. A derived predicate is always correct with zero moving parts.

## 21.5 Every retrieval path composes on it

Verified by reading the call sites:

| Path | How it uses the filter |
|---|---|
| `MemoryRepository.get` | `current_revisions(project_id)` |
| `MemoryRepository.search` | `current_revisions(project_id)` then `_apply_filters` |
| `MemoryRepository.count` | same |
| `MemoryRepository.explain_search` | same |
| `semantic.semantic_candidates` | `current_revisions(project_id).join(MemoryEmbedding, ...)` |
| `semantic.search_by_vector` | the same four conditions, written out in raw SQL |
| `services.retrieval.hybrid_search` | both legs go through the above |
| `services.context.build_context` | goes through search or hybrid search |
| `TruthRepository.semantic_coverage` | the same conditions |

There is exactly one way to see anything outside this set, and it is `include_retired=True`, used
only by `memory_history` and the debug path. The module docstring says: "Grep for it: if it appears
in a normal retrieval path, that is a bug."

**Diagram 18 — stale-memory filtering**

```
   All memory_revisions rows in the database
   +-------------------------------------------------------+
   |  every project, every status, every revision           |
   +-------------------------------------------------------+
                       |
                       |  STAGE 0  (retrieval/filters.py)
                       |  project_id = ?
                       |  status = 'ACTIVE'
                       |  expires_at IS NULL OR expires_at > now()
                       |  is_current
                       v
   +-------------------------------------------------------+
   |  CANDIDATE SET  —  everything here is currently true   |
   +-------------------------------------------------------+
             |                              |
        lexical ranking             semantic ranking
             |                              |
             +--------- RRF ----------------+
                       |
                       v
                 top N results

   The retired "Redis" memory never enters the candidate set.
   No scoring function — today's or a future one — can bring it back.
```

## 21.6 The tests

`tests/integration/test_stale_memory.py::test_the_retired_fact_leaves_retrieval_entirely` is the
flagship, and the shape of its assertion is the point:

```python
for limit in (1, 2, 5, 10, 50, 100):
    for query in ("queue", "task queue", "redis", None):
        found = await search(db_session, project.id, query=query, limit=limit)
        assert REDIS not in [m.content for m in found.memories], (
            f"the superseded Redis memory surfaced at limit={limit}, query={query!r}. "
            "Suppression must be structural, not a ranking effect."
        )
```

**Why the loop over limits is the whole argument.** Asserting only that the PostgreSQL memory
ranks first would pass even if the Redis memory were merely ranked lower — and would then leak the
moment a caller asked for more results. **Absence at every limit** is what proves suppression is
structural rather than a ranking accident.

Other tests that hold the same line:

- `test_querying_the_stale_term_still_finds_only_the_current_answer` — the hardest case, searching
  the retired memory's own subject.
- `test_a_second_retriever_cannot_resurrect_a_retired_memory` (hybrid suite) — the guarantee had to
  survive adding a whole second retriever.
- `test_hybrid_leaks_no_retired_memories` (eval suite) — `stale_inclusion_rate == 0.0` across the
  full graded corpus with real embeddings.
- Every context-builder test — absent at every token budget too.

## 21.7 Measured, across every strategy

| Strategy | nDCG@10 | Stale memories returned |
|---|---|---|
| Full text, all terms required | 0.478 | **0.000** |
| Full text, any-term fallback | 0.802 | **0.000** |
| Hybrid: FTS + pgvector, RRF | 0.853 | **0.000** |

The last column is the point, not the first three. Retrieval quality changed a great deal across
those three rows. The correctness guarantee did not move at all, because it does not live in the
ranking.

---

# PART 22 — RECIPROCAL RANK FUSION

## 22.1 The problem, concretely

Two retrievers each return a ranked list for the query "redis":

```
  Lexical (full text)          Semantic (pgvector)
  1. A                         1. B
  2. C                         2. A
  3. B                         3. D
```

You need one list. How?

## 22.2 The obvious idea, and why it fails

"Add the scores together."

Look at what the scores actually are:

- `ts_rank_cd` returns something like `0.083`, or `12.4`, or `0.0001`. It is **unbounded**, and it
  depends on the document's length and on how often the term appears *in that document*. Its value
  is meaningful only within one query.
- Cosine distance is in `[0, 2]`, and **smaller is better** — the opposite direction.

Adding `0.083` to `0.21` produces a number with no meaning. The two are not on the same scale, do
not point the same way, and are not comparable across queries.

## 22.3 The second idea, and why it is worse

"Normalise each list to 0–1 first, then add."

This is **min-max normalisation**, and it is query-dependent, which is exactly the flaw. Consider
two queries:

- Query 1 has a perfect match at raw score 0.9 and a poor one at 0.1.
- Query 2 has a mediocre best match at 0.09 and a terrible one at 0.01.

After per-query normalisation, both top results are exactly 1.0. The two queries become
indistinguishable at precisely the moment the difference matters.

## 22.4 The idea that works

**Throw the scores away. Use only the positions.**

Position is already comparable. "First in the lexical list" and "first in the semantic list" mean
the same kind of thing, whatever the underlying numbers were. Nothing needs normalising, because
no magnitudes are being compared.

That is **Reciprocal Rank Fusion**.

## 22.5 The formula, with every symbol defined

```
                          1
   score(d)  =   sum   -------
                  i     k + r_i(d)
```

- **d** — one document (here, one memory).
- **i** — one retriever. Here there are two: `lexical` and `semantic`.
- **r_i(d)** — the **rank** (position) of document `d` in retriever `i`'s list, counting from 1. If
  `d` does not appear in that list at all, the term contributes **nothing** — it is simply omitted
  from the sum.
- **k** — a constant that damps the difference between top positions. Here `k = 60`.
- **sum** — add one term per retriever that returned this document.

"Reciprocal" means `1/x`. "Rank" is the position. "Fusion" is the combining.

## 22.6 Why k = 60

`k = 60` is the value from the original TREC research on rank fusion and is the de facto default.

What it does: without `k` (that is, with `k = 0`), rank 1 scores `1/1 = 1.0` and rank 2 scores
`1/2 = 0.5` — rank 1 is worth **twice** rank 2. That overweights a single retriever's confident
mistake. With `k = 60`, rank 1 scores `1/61 = 0.01639` and rank 2 scores `1/62 = 0.01613` — close
together, so agreement between retrievers matters more than one retriever's certainty.

## 22.7 Working the example by hand

```
  Lexical:   1. A    2. C    3. B
  Semantic:  1. B    2. A    3. D

  k = 60

  A:  lexical rank 1  ->  1/(60+1) = 0.016393
      semantic rank 2 ->  1/(60+2) = 0.016129
      total                          0.032522     <- found by both

  B:  lexical rank 3  ->  1/(60+3) = 0.015873
      semantic rank 1 ->  1/(60+1) = 0.016393
      total                          0.032266     <- found by both

  C:  lexical rank 2  ->  1/(60+2) = 0.016129
      semantic          ->  absent, contributes nothing
      total                          0.016129

  D:  lexical          ->  absent
      semantic rank 3 ->  1/(60+3) = 0.015873
      total                          0.015873

  Fused order:  A (0.032522), B (0.032266), C (0.016129), D (0.015873)
```

Read what happened. **A and B rise clearly above C and D**, because two independent retrievers
agreed on them. B was only third lexically but first semantically, and it still beats C, which was
second on one list and absent from the other. That is exactly the behaviour you want from
combining a keyword view and a meaning view.

**Diagram 17 — RRF**

```
   lexical:   A(1)  C(2)  B(3)
   semantic:  B(1)  A(2)  D(3)

   A:  1/61 + 1/62  = 0.0325   ####################
   B:  1/63 + 1/61  = 0.0323   ###################
   C:  1/62 + —     = 0.0161   ##########
   D:  —    + 1/63  = 0.0159   #########

   agreement wins.  a single first place does not.
```

## 22.8 The implementation

`src/memhub/retrieval/fusion.py`:

```python
RRF_K = 60

def reciprocal_rank_fusion(rankings, *, k=RRF_K, weights=None):
    weights = weights or {}
    scores = {}
    positions = {}

    for name, ranking in rankings.items():
        weight = weights.get(name, 1.0)
        positions[name] = {}
        for index, memory_id in enumerate(ranking):
            rank = index + 1
            positions[name][memory_id] = rank
            scores[memory_id] = scores.get(memory_id, 0.0) + weight / (k + rank)

    fused = [FusedResult(memory_id=mid, score=score,
                         lexical_rank=positions.get("lexical", {}).get(mid),
                         semantic_rank=positions.get("semantic", {}).get(mid))
             for mid, score in scores.items()]
    fused.sort(key=lambda result: (-result.score, str(result.memory_id)))
    return fused
```

Details worth noticing:

- **`weights` defaults to 1.0 for every retriever.** The docstring is explicit about why: weighting
  before measuring is guesswork, and the evaluation harness exists to replace guesses with numbers.
- **Ties break on memory id.** Without that, two documents with equal fused scores come back in
  dictionary insertion order, which is stable within a process and not across them — and unstable
  output cannot be snapshot-tested.
- **A missing document needs no special case.** It simply does not appear in that retriever's
  positions map and contributes nothing. No zero-filling, no skew. This is exactly the right
  behaviour for a memory whose embedding has not been computed yet.

## 22.9 The properties that make RRF the right tool here

| Property | Why it matters here |
|---|---|
| **Scale-free** | nothing needs normalising, because no magnitudes are compared |
| **No per-query normalisation** | a query with only weak matches does not get flattered into looking like one with a perfect match |
| **Robust to one retriever failing** | if the vector index is empty because the outbox is behind, those documents simply do not appear and contribute nothing |
| **Rewards agreement** | a document both retrievers rank highly beats one that either ranks first alone |

## 22.10 Tests

`tests/unit/test_fusion.py`, 12 tests. Pure functions, no database — which is exactly what lets
them be checked against worked examples where the right answer is known by hand.

---

# PART 23 — BACKGROUND EMBEDDINGS

## 23.1 The question

Generating an embedding takes tens to hundreds of milliseconds of CPU work, and it can fail. Why
not just do it during the write, before returning?

## 23.2 Three options, and the reasoning

### Option (a) — inside the write transaction

```
BEGIN
  INSERT memory
  INSERT revision
  embed(content)        <- slow, fallible, holds the transaction open
  INSERT embedding
COMMIT
```

**Latency.** Every write now waits for model inference.

**Locking.** It holds a PostgreSQL transaction — and the `memories` row lock — open across a slow
call. Under contention, one slow embedding call blocks every other writer on that memory.

**Failure.** This is the worst part. A model being unavailable becomes a **write outage**. You can
no longer record a decision because an embedder is down. That coupling is absurd: the memory text
is the product; the vector is an enhancement.

**Rejected.**

### Option (b) — after the transaction, fire and forget

```
BEGIN ... COMMIT              <- the memory is now durable
embed_and_store(...)          <- separately, afterwards
```

This is the classic **dual-write bug**. Crash between the commit and the enqueue — process killed,
machine loses power, an exception in between — and the memory exists forever with **no vector and
nothing that knows to fix it**. There is no record that work is outstanding. It is a silent,
permanent hole in the index.

**Rejected.**

### Option (c) — transactional outbox — chosen

```
BEGIN
  INSERT memory
  INSERT revision
  INSERT embedding_jobs row          <- the outbox row, same transaction
COMMIT

  ... later, separately ...

worker: claim a batch of jobs -> embed -> store vectors -> mark DONE
```

Either both the revision and its job exist, or neither does. There is no window where a memory
exists with no vector and nothing knows to produce one, and no window where a job points at a
revision that was rolled back.

## 23.3 The words

**Worker** — code that does work.

**Background worker** — code that runs separately from request handling, doing deferred work.

**Job** — one unit of that work. Here, one row in `embedding_jobs`.

**Queue** — the collection of pending jobs. Here, the `embedding_jobs` table.

**Outbox** — a table where a row describing follow-up work is written.

**Transactional outbox** — the pattern where that row is written **in the same transaction** as
the change it relates to. The name comes from the idea of an outbox tray: you put the letter in
the tray as part of the same act as writing it, and something else posts it later.

## 23.4 The implementation

**Enqueue** — `TruthRepository.enqueue_embedding`, called from inside `remember` and `revise`:

```sql
INSERT INTO embedding_jobs (memory_id, revision_no, project_id, model)
VALUES (:mid, :rev, :pid, :model)
ON CONFLICT (memory_id, revision_no, model) DO NOTHING
```

`ON CONFLICT DO NOTHING` because a revision only ever needs embedding once per model, and a
re-enqueue after a retry must not create a second job.

**Claim** — `src/memhub/embeddings/worker.py`:

```sql
SELECT j.id, j.memory_id, j.revision_no, j.project_id, j.attempts, r.content
  FROM embedding_jobs j
  JOIN memory_revisions r
    ON r.memory_id = j.memory_id AND r.revision_no = j.revision_no
 WHERE j.state = 'PENDING'
   AND j.model = :model
   AND j.next_attempt_at <= now()
 ORDER BY j.next_attempt_at
   FOR UPDATE OF j SKIP LOCKED
 LIMIT :batch_size
```

**Process** — embed the batch off the event loop (`asyncio.to_thread`, because inference is
CPU-bound and synchronous, and running it inline would block every other request in the process),
insert the vectors, mark the jobs `DONE`. The whole batch is **one transaction**: if anything
raises, the transaction rolls back and the jobs return to `PENDING`, so a crash mid-batch loses no
work and leaves no vector half-written.

**Where the worker runs.** In-process, as an asyncio background task inside each server process
(`_drain_forever` in `src/memhub/mcp/__main__.py`), polling every 2 seconds. The comment explains
the trade: a separate worker process would be more robust and is what a hosted deployment would
want, but for a stdio subprocess it would mean asking the user to run a second thing, which is
worse than the failure it avoids.

Polling rather than `LISTEN`/`NOTIFY`: the queue is small, the latency budget is "before the user
asks their next question", and a notification channel would add a failure mode — a missed
notification means a job sits forever — for a gain nobody would notice.

Any exception in the loop is logged and swallowed. A crashing background task must not take the
server down; the whole reason embedding is asynchronous is that it is allowed to fail.

## 23.5 Failure handling in the worker

```python
MAX_ATTEMPTS = 5
BASE_BACKOFF = dt.timedelta(seconds=2)
```

On an `EmbeddingError`, `_defer` runs. For each job in the batch:

- `attempts + 1`;
- if that reaches 5, set `state='DEAD'` and store `last_error`;
- otherwise set `next_attempt_at = now() + 2s * 2^(attempts-1)` — so 2s, 4s, 8s, 16s.

**Exponential backoff** means each retry waits longer than the last, so a failing dependency is not
hammered. The code notes that jitter (adding randomness so retries do not synchronise) would be
better under many workers; with the two this design has, plain exponential is enough and simpler
to reason about.

**Why bounded at 5.** An embedder that has failed five times is not going to succeed on the sixth,
and a job retrying forever is an infinite log of the same error that hides everything else. A DEAD
job is visible in the table and countable as a metric — never a silent hole in the index.

## 23.6 The consistency model, stated honestly

**Eventual consistency with an explicit, observable, queryable pending state.**

The system is never *silently* inconsistent:

- `embedding_jobs.state` is a fact you can query.
- `semantic_coverage` is computed and reported in every hybrid search response.
- `memhub-admin status` reports `embed_pending` and `embed_dead`.

The trade-off is stated plainly: **a memory written 200 milliseconds ago is findable by full-text
search but may not yet be findable by semantic search.** For this product that window is
irrelevant, and the design says so rather than pretending it does not exist.

## 23.7 Tests

`tests/integration/test_embedding_outbox.py`, 12 tests:

| Test | What it proves |
|---|---|
| `test_a_write_enqueues_a_job_in_the_same_transaction` | the job row is there when the write commits |
| `test_a_rolled_back_write_leaves_no_job` | **the outbox test.** Roll the write back and the job disappears with it. This is the difference between a transactional outbox and the dual-write bug. |
| `test_a_revision_gets_its_own_job` | revising enqueues a job for the new revision |
| `test_drains_the_queue_and_stores_vectors` | the happy path |
| `test_running_again_does_nothing` | idempotent draining |
| `test_two_workers_never_process_the_same_job` | see Part 24 |
| `test_a_broken_embedder_does_not_break_writes` | writes succeed with a permanently failing embedder |
| `test_failure_backs_off_rather_than_spinning` | `next_attempt_at` moves forward |
| `test_repeated_failure_ends_in_dead_not_an_infinite_retry` | `state='DEAD'` after 5 attempts |
| `test_coverage_reports_the_pending_window_honestly` | coverage below 1.0 while jobs are pending |
| `test_partial_coverage_is_reported_as_a_fraction` | the arithmetic is right |
| `test_an_empty_project_is_fully_covered` | 1.0, not a division by zero |

**Diagram 19 — background embedding jobs**

```
   memory_remember("...")
   +--------------------------------------------------+
   | BEGIN                                            |
   |   INSERT memories                                |
   |   INSERT memory_revisions                        |
   |   INSERT embedding_jobs (state='PENDING')  <-- the outbox row
   | COMMIT                                           |
   +--------------------------------------------------+
        both exist, or neither does
                    |
                    |   ... 0-2 seconds later ...
                    v
   worker.run_once()
   +--------------------------------------------------+
   | BEGIN                                            |
   |   SELECT ... FOR UPDATE OF j SKIP LOCKED LIMIT 16|
   |   embed(contents)          <- off the event loop |
   |   INSERT memory_embeddings                       |
   |   UPDATE embedding_jobs SET state='DONE'         |
   | COMMIT                                           |
   +--------------------------------------------------+
        vector and DONE together, or neither

   on EmbeddingError:
        attempts + 1
        attempts < 5  ->  next_attempt_at = now() + 2s * 2^(attempts-1)
        attempts = 5  ->  state = 'DEAD', last_error stored
```

---

# PART 24 — POSTGRESQL AS A BACKGROUND JOB QUEUE

## 24.1 Why PostgreSQL rather than Redis or a message broker

Three reasons.

1. **The transactional outbox requires it.** The whole point is that the job row and the write are
   the same transaction. You cannot have a transaction spanning PostgreSQL and Redis. Putting the
   queue anywhere else reintroduces the dual-write bug the outbox exists to remove.

2. **The guarantees a job queue needs were already there.** Durability, atomicity, ordering,
   visibility rules, and a locking primitive designed for exactly this (`SKIP LOCKED`) are all in
   the database being written to anyway.

3. **One fewer thing to run.** No Redis container, no broker, no separate connection string, no
   second failure domain.

There is a pleasing self-reference here worth mentioning in an interview: **this project uses
PostgreSQL as its durable job queue, which is exactly the decision used as the running example
throughout its own documentation, and exactly the reason Redis is not a dependency.**

## 24.2 What scale makes this reasonable

Count the actual load. One job per memory write, plus one per revision. A developer records
perhaps tens of memories a day. Even a heavy day is a few hundred rows. Batches of 16, polled
every 2 seconds, by at most a handful of worker tasks.

That is nowhere near the scale where a purpose-built broker earns its complexity. `SKIP LOCKED` on
a partial index over a table whose PENDING set is near-empty at steady state is comfortably enough.

## 24.3 When an external queue would become preferable

Be able to name the crossover points:

- **Throughput.** Thousands of jobs per second, where the queue's write load starts competing with
  the application's.
- **Fan-out.** Several different kinds of consumer needing different delivery semantics.
- **Cross-service delivery.** When the producer and consumer are separate services with separate
  databases — at which point the outbox pattern typically stays, with a relay publishing from the
  outbox to the broker.
- **Long retention of processed jobs**, where the table growth becomes an operational problem.
- **Delivery guarantees this does not offer**, such as fan-out to multiple independent consumers or
  ordered partitions.

None of those apply here, and saying so precisely is better than saying "PostgreSQL scales".

## 24.4 The words

**Row lock** — a lock taken on a specific row, held until the transaction commits or rolls back.
Another transaction wanting to modify that row waits.

**FOR UPDATE** — a clause on a `SELECT` that takes an exclusive row lock on every row returned. It
says "I intend to modify these; nobody else may touch them until I finish."

**SKIP LOCKED** — a modifier meaning "if a row I would have returned is already locked by another
transaction, do not wait for it — skip it and return the next one".

That one modifier is what turns a table into a work queue several workers can drain at once.

## 24.5 Two workers, two jobs, worked through

**Without `SKIP LOCKED`:**

```
   Queue: job 1, job 2 (both PENDING)

   Worker A:  SELECT ... FOR UPDATE LIMIT 1   -> gets job 1, locks it
   Worker B:  SELECT ... FOR UPDATE LIMIT 1   -> also matches job 1
                                                 -> BLOCKS, waiting for A
   Worker A:  embeds, commits, releases the lock
   Worker B:  wakes up, re-checks... job 1 is now DONE, so it does not match
              -> B has to run the query again to find job 2

   Result: the two workers run at the speed of one. B spent its time waiting
           for a row it was never going to get.
```

**With `SKIP LOCKED`:**

```
   Queue: job 1, job 2 (both PENDING)

   Worker A:  SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1  -> job 1, locked
   Worker B:  SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1  -> job 1 is locked,
                                                            SKIP it -> job 2
   Both work at the same time, on disjoint jobs. No coordination beyond
   the database. No waiting.
```

**Diagram 20 — SKIP LOCKED**

```
   embedding_jobs
   +----+---------+----------------+
   | id | state   | locked by      |
   +----+---------+----------------+
   |  1 | PENDING | worker A  <----+---- A took this one
   |  2 | PENDING | worker B  <----+---- B skipped 1, took this one
   |  3 | PENDING | —              |
   |  4 | DONE    | —              |
   +----+---------+----------------+

   Without SKIP LOCKED:  B blocks on row 1 and achieves nothing.
   With SKIP LOCKED:     B steps over row 1 and takes row 2.
```

## 24.6 `FOR UPDATE OF j`

Note the exact clause in the query: `FOR UPDATE OF j SKIP LOCKED`, where `j` is the
`embedding_jobs` alias.

The `OF j` restricts the lock to the **job** row only. The query joins to `memory_revisions` to get
the content, and without `OF j` the revision row would be locked too — which would make embedding
contend with revision writes for no reason at all. A worker reading a memory's text must not block
someone revising that memory.

## 24.7 What happens if the worker crashes

Walk through it:

- **Crash before the claim.** Nothing has happened. The jobs are still PENDING.
- **Crash after the claim, before the commit.** The transaction is aborted by the database, which
  releases the row locks. The jobs revert to being unlocked and PENDING. Another worker — or the
  same one after restart — picks them up. **No work is lost and no vector is half-written.**
- **Crash after the commit.** The vectors are stored and the jobs are DONE. Nothing to redo.

There is no state in which a job is marked DONE without its vector existing, or a vector exists
while the job still looks pending, because the vector insert and the `state='DONE'` update are in
the same transaction.

## 24.8 The test

`test_two_workers_never_process_the_same_job` seeds 40 memories, then runs **four** workers with
`batch_size=8` concurrently:

```python
workers = [EmbeddingWorker(sessions, HashEmbedder(), batch_size=8) for _ in range(4)]
outcomes = await asyncio.gather(*(w.drain() for w in workers))

assert sum(o.embedded for o in outcomes) == 40, "a job was processed twice or skipped"
# and
assert vectors == 40
```

`sum(...) == 40` is the assertion that catches both failure modes: more than 40 means a job was
processed twice; fewer means one was skipped.

Underneath, the primary key `(memory_id, revision_no, model)` on `memory_embeddings` would also
reject a double insert — another instance of the same defence-in-depth pattern.

---

# PART 25 — TOKEN-BUDGETED CONTEXT

## 25.1 `memory_search` versus `memory_context`

They answer different questions.

| | `memory_search` | `memory_context` |
|---|---|---|
| Question | "what matches this query?" | "given this much room, what is worth knowing?" |
| Needs a query | effectively yes | no — omit it for a general overview |
| Limit | a count (1–100 results) | a **cost** (100–32000 tokens) |
| Output | a ranked list | a readable brief, plus structured records, plus a budget report |
| Balances types | no | yes, per-type quotas |
| Drops near-duplicates | no | yes, MMR diversity |
| When used | when you need one specific fact | at the start of a session, before you know what you will be asked |

## 25.2 The words

**Token** (model sense) — the unit a language model reads text in, roughly a word fragment.

**Token budget** — a limit on how many tokens the response may cost.

**Context window** — the total amount of text the model can hold at once. The budget exists so a
brief does not eat it.

## 25.3 The problem, concretely

Retrieval offers 60 candidate memories. The budget allows about 2000 tokens. Which do you include?

The ten most *relevant* memories might be five restatements of one decision plus five details of a
task that finished last month. A brief made of those is worse than a shorter one covering four
different things.

So this is a **constrained selection problem**, and it is built as one.

## 25.4 The pipeline, as implemented

`src/memhub/context/builder.py::select`, with `src/memhub/services/context.py` feeding it.

```
 1. CANDIDATES   over-fetch CANDIDATE_POOL = 60 from retrieval
                 (hybrid if a query and an embedder; lexical if a query only;
                  importance-ordered browse if no query)
                 -> the stage-0 filter has already run; nothing retired can be here

 2. SCORE        score = 1 / (index + 1)   -- rank position from retrieval
                 tokens = estimator.estimate(content)

 3. USABLE       spendable = int(budget * (1 - 0.10))       -- 10% safety margin

 4. HARD FILTER  drop anything with tokens > spendable      -> "too_large_alone"

 5. QUOTAS       per type: CONSTRAINT 25%, DECISION 40%, FACT 20%, TASK 15%
                 processed largest share first
                 each type filled by _fill() within its own allowance

 6. REDISTRIBUTE anything unspent is offered to everything not yet chosen
                 and not already rejected as redundant

 7. DROP TALLY   "too_similar", "no_budget_left", "too_large_alone"

 8. ORDER        stable: CONSTRAINT, DECISION, FACT, TASK; then score desc;
                 then memory_id asc

 9. RENDER       grouped markdown brief + structured records + budget report
```

Inside `_fill`, for one allowance:

```
   sort the pool by value_density = score / tokens, ties broken by id
   loop:
     for each candidate that still fits in the remaining allowance:
       novelty = 1 - max(Jaccard similarity with anything already chosen)
       value   = 0.7 * value_density + 0.3 * novelty * value_density
       keep the best
     if the best is >= 0.6 similar to something already chosen:
       record it as redundant, skip it
     else:
       take it, spend its tokens
```

## 25.5 Every mechanism, named and verified

**Quotas** — yes, implemented. `TYPE_QUOTA` in `builder.py`:

```python
MemoryType.CONSTRAINT: 0.25   # violating one breaks the project
MemoryType.DECISION:   0.40   # the largest share; the most useful thing stored
MemoryType.FACT:       0.20
MemoryType.TASK:       0.15   # smallest, and expires fastest
```

Why: so a flood of one kind cannot crowd out the others. Constraints are usually the smallest
group and the most important, so they get a guaranteed share.

**Redistribution** — yes. A project with no open tasks should spend that 15% on decisions rather
than leave it unused, so a second pass offers the whole remaining budget to everything neither
chosen nor already ruled out.

A subtle detail in the code: the set of candidates rejected as near-duplicates is **carried across
passes** rather than recomputed. A candidate rejected during its quota must not be reconsidered
during redistribution — it would be rejected again for the same reason, and the drop counts would
report more rejections than there were candidates.

**MMR (Maximal Marginal Relevance)** — yes, implemented, with `MMR_LAMBDA = 0.7`:

```python
value = MMR_LAMBDA * candidate.value_density + (1 - MMR_LAMBDA) * novelty * (...)
```

MMR balances "how good is this" against "how different is this from what I already have". At
lambda 1.0 it is pure greedy relevance and near-duplicates fill the brief. At 0.0 it selects for
difference alone and returns unrelated trivia. 0.7 leans towards relevance while still refusing a
memory that says what one already selected says.

**Similarity measure — Jaccard overlap of content words, not embeddings.** `similarity()` computes
`|A ∩ B| / |A ∪ B|` over the content words of two memories, with stopwords removed. Two reasons
this is the right choice here: diversity is about *redundant phrasing*, which word overlap captures
well; and making the context builder depend on the embedding pipeline would mean a brief could not
be produced while the outbox was behind.

**`DUPLICATE_SIMILARITY = 0.6`** — above this, two memories are treated as saying the same thing
and the second is dropped as `too_similar`.

**Knapsack, solved greedily** — yes. Each candidate has a **value density**, `score / tokens`. The
fill loop orders by that. A long memory has to earn its space: ranking by score alone would let one
verbose decision consume a third of the budget for the same value as three concise ones.

The docstring is explicit that this is a knapsack problem solved greedily on the ratio, which is
within a known bound, and that exact dynamic programming is not worth it at fewer than 500
candidates.

**Deterministic ordering** — yes, and it is not cosmetic. `_stable_order` sorts by type order, then
score descending, then `memory_id` ascending. A brief whose contents shuffle between identical
calls cannot be snapshot-tested and cannot be safely cached.

## 25.6 Token estimation, honestly

**This cannot be exact, and pretending otherwise would be the mistake.** The budget is spent in
the *client's* model, and the server does not know which model that is. Claude, GPT and Llama
tokenise the same sentence into different counts, and MCP carries no field that would tell us.

Three options were considered:

- **`tiktoken`** — precise-looking and wrong. It is OpenAI's tokeniser; using it to budget for
  Claude produces confident numbers that are systematically off, which is worse than an honest
  approximation because nobody thinks to check them.
- **A hosted token-counting API** — accurate for one vendor, and puts a network call on the read
  path of a feature whose entire purpose is to be fast.
- **A calibrated heuristic** — chosen.

```python
CHARS_PER_TOKEN = 3.2
PER_ITEM_OVERHEAD = 12
SAFETY_MARGIN = 0.10

def estimate(self, text): return math.ceil(len(text) / 3.2) + 12
def usable_budget(budget, margin=0.10): return max(1, int(budget * 0.90))
```

**The contract, stated plainly:**

> **Never exceed the stated budget. May under-fill it by roughly 10%.**

The asymmetry is the whole design. Exceeding the budget corrupts the caller's context window;
under-filling wastes a little of it. Those are not comparable failures, so the estimator is tuned
to fail in the cheap direction.

## 25.7 The calibration, and the correction it forced

This is a good story to tell, because the first value was wrong and the repository says so.
`docs/eval/tokens.md`:

`CHARS_PER_TOKEN` started at **3.6**, chosen as "safely below the ~4.0 usually quoted for English
prose". Measured against a real BPE tokeniser over all 33 hand-written memories in the evaluation
corpus:

| Divisor 3.6 | |
|---|---|
| Mean actual chars/token | 4.17 |
| **Minimum** actual chars/token | **3.25** |
| Mean estimate ÷ actual | 1.18 |
| **Worst estimate ÷ actual** | **0.938** |
| **Samples under-estimated** | **2 of 33** |

The reasoning was sound for prose and wrong for this corpus. Technical writing is full of
identifiers — `FOR UPDATE SKIP LOCKED`, `websearch_to_tsquery`, `PostgreSQL` — and byte-pair
encoding splits those into many short tokens. The densest sample came in at **3.25** characters per
token, below the 3.6 divisor.

Corrected to **3.2**, below the densest observed sample:

| Divisor 3.2 | |
|---|---|
| Mean estimate ÷ actual | 1.323 |
| **Worst estimate ÷ actual** | **1.048** |
| **Samples under-estimated** | **0 of 33** |

The most interesting sentence in that document:

> The 10% safety margin would have absorbed those two cases at the aggregate level, which is
> precisely why this was worth measuring rather than reasoning about: the guarantee would have held
> by accident, through a coupling nobody had written down, and would have broken the first time
> someone tuned the margin.

**The cost, stated.** Over-estimating by about 32% on average means a brief fills roughly two
thirds of the requested budget before the safety margin is even applied. That is the price of the
guarantee. The two ways to narrow it later are named: supply the real tokeniser (`TokenEstimator`
is a `Protocol` exactly so a deployment that knows its client's model can pass the actual counter),
or re-calibrate the divisor per corpus.

## 25.8 The budget report

Every response includes what was spent and why things were left out:

```json
"budget": {"requested": 2000, "estimated_used": 1180, "utilisation": 0.59,
           "estimator": "heuristic(chars/3.2)",
           "considered": 60, "selected": 11,
           "dropped": {"too_similar": 7, "no_budget_left": 42}}
```

Why report it: a caller that asked for 2000 tokens and received 900 needs to know whether the
project has little to say or whether thirty memories were dropped for lack of room. Those call for
opposite responses.

**Diagram 21 — token-budgeted context**

```
   60 candidates                     budget 2000 tokens
        |                                  |
        |                          spendable = 1800 (10% margin held back)
        v
   drop anything that cannot fit alone            -> "too_large_alone"
        |
        v
   +-----------+-----------+---------+---------+
   | CONSTRAINT| DECISION  |  FACT   |  TASK   |    quotas
   |   450     |   720     |  360    |  270    |    (25/40/20/15 %)
   +-----------+-----------+---------+---------+
        each filled greedily by score/token, with MMR novelty,
        rejecting anything >= 0.6 Jaccard-similar to a pick    -> "too_similar"
        |
        v
   redistribute whatever is unspent to everything left         -> "no_budget_left"
        |
        v
   stable order: CONSTRAINT, DECISION, FACT, TASK; score desc; id asc
        |
        v
   grouped markdown brief  +  structured records  +  budget report
```

---

# PART 26 — FAILURE HANDLING

## 26.1 Start from the question

What can go wrong when this Python application talks to PostgreSQL?

- The database is not running, or the port is wrong, so the connection is refused.
- The host name does not resolve.
- Every connection in the pool is busy and none frees up.
- A query is slow and gets cancelled.
- The connection dies **while a statement is in flight**.
- The database restarts underneath us.
- A constraint rejects the write.
- The input was malformed.

Not all of these are the same kind of problem, and — this is the whole point — **the correct thing
for the caller to do next differs for each**.

## 26.2 Why classification matters

Without classification, all of these reach the model as one opaque `OperationalError`. A model that
receives an opaque internal error concludes the tool is broken and stops using it. A model that
receives a named code with a retry hint waits and tries again, which is the correct behaviour for a
restarting database.

## 26.3 The four codes

Verified in `src/memhub/domain/errors.py` and `src/memhub/mcp/mapping.py`.

| Code | What happened | Safe to retry? | Because |
|---|---|---|---|
| `BACKEND_UNAVAILABLE` | the database could not be reached | **yes** | the connection never opened; nothing ran |
| `BACKEND_BUSY` | the connection pool timed out | **yes** | no statement was ever sent |
| `UNKNOWN_OUTCOME` | the connection died mid-flight | **no** | the write may have committed |
| `DEADLINE_EXCEEDED` | `statement_timeout` cancelled the query | **no** | the query was too slow; the server is healthy, so retrying re-runs the same slow query |

Plus the domain errors, which are the caller's fault and not infrastructure: `VALIDATION_FAILED`,
`PROJECT_NOT_FOUND`, `AMBIGUOUS_PROJECT`, `PROJECT_EXISTS`, `MEMORY_NOT_FOUND`,
`IDEMPOTENCY_KEY_REUSED`.

## 26.4 The classifier, in order

`classify_infrastructure_error` in `src/memhub/mcp/mapping.py`:

```python
if isinstance(exc, PoolTimeoutError):
    return BackendBusyError(..., retryable=True)

if isinstance(exc, ConnectionError | socket.gaierror):
    return BackendUnavailableError(..., retryable=True)

if not isinstance(exc, DBAPIError):
    return None                                    # unrecognised: let it propagate

pgcode = getattr(getattr(exc, "orig", None), "sqlstate", None)

if pgcode == QUERY_CANCELED:                       # 57014
    return DeadlineExceededError(..., retryable=False)

if exc.connection_invalidated:                     # <-- the load-bearing branch
    return UnknownOutcomeError(..., retryable=False)

if pgcode in CONNECTION_LOST or isinstance(exc.orig, ConnectionError | socket.gaierror):
    return BackendUnavailableError(..., retryable=True)

return None
```

Two design decisions in there worth naming:

**Returning `None` for anything unrecognised.** An unrecognised exception is a bug, and it
propagates. Dressing an unfamiliar failure up as a known one would be worse than an opaque error,
because it would arrive with confident and possibly wrong advice about whether retrying is safe.
Tested by `test_an_unfamiliar_error_is_not_classified`.

**The branch order.** See 26.6.

## 26.5 UNKNOWN_OUTCOME in detail

**TERM: UNKNOWN_OUTCOME**

**Simple definition.** The connection to the database died while a statement was in flight, so
whether the write committed is genuinely unknown.

**Why the problem exists.** Committing and being told about the commit are two different events. A
transaction can commit on the server, and the acknowledgement travelling back can be lost. From
this side, that looks exactly like the transaction never having run.

**Concrete example.**

```
   Client sends memory_remember("The job queue runs on PostgreSQL.")
        |
   Server: BEGIN ... INSERT ... COMMIT   <- PostgreSQL has durably committed
        |
   *** the connection dies ***           <- the acknowledgement is lost
        |
   The application never learns the result.
```

Now: did it happen? **Maybe.** There is no way to tell from this side. The acknowledgement that
would have said which is precisely what was lost.

**Without this classification.** The failure is reported as a generic error, or worse, as
"retryable". A blind retry then either does nothing wrong (if the write failed) or **writes the
memory twice** (if it succeeded). You cannot know which.

**With it.** The response does not claim the write failed. It says the outcome is unknown and names
the two ways to resolve it. The actual message:

> "the connection to the database was lost while the request was in flight. The write may or may
> not have committed. Replay the same request with its idempotency key to find out, or re-read
> before retrying - retrying without a key risks writing twice."

**How it connects back to idempotency.** This is the payoff of Part 12. If the original request
carried a `client_request_id`, the client can simply send the same request again:

- if the write did land, the key is `COMPLETED` and the original response is replayed;
- if it did not, the key is free and the write happens once.

Either way, exactly one memory exists. **Idempotency is what turns an unrecoverable ambiguity into
a recoverable one.**

**Relevant source file.** `src/memhub/domain/errors.py` (`UnknownOutcomeError`),
`src/memhub/mcp/mapping.py` (`classify_infrastructure_error`).

**Relevant test.** `test_a_lost_connection_mid_flight_is_an_unknown_outcome`,
`test_invalidation_outranks_the_sqlstate`, `test_it_names_both_ways_out`.

**Different from.** `BACKEND_UNAVAILABLE`, where the connection never opened, so nothing ran and
retrying is unconditionally safe.

**20-second interview explanation.** "The genuinely hard failure is when the connection dies
mid-flight. The transaction either committed just before the drop or it did not, and the
acknowledgement that would have told me is exactly what was lost. So I do not guess. I return
UNKNOWN_OUTCOME, explicitly not marked retryable, and the message names the two ways to resolve it:
replay the idempotency key, or re-read. Retrying blindly is how you get duplicate writes."

**Diagram 22 — UNKNOWN_OUTCOME**

```
   client                          server                 PostgreSQL
     |  remember(..., key=R)  ->     |                        |
     |                               |  BEGIN INSERT COMMIT ->|
     |                               |                        |  committed, durable
     |                               |<-- ack ----------------|
     |         X  connection dies    |
     |                               |
     |  ???  did it commit?          |
     |
     |  send the SAME request with key R
     |         ->  key R is COMPLETED  ->  replay the stored response
     |         ->  key R is absent     ->  do the write, once
     |
     |  either way: exactly one memory exists
```

## 26.6 The branch order, and why it is mutation-tested

In a real mid-transaction disconnect, **two signals are present at once**: SQLSTATE `08006`
(connection failure) appears, *and* SQLAlchemy sets `DBAPIError.connection_invalidated`.

`connection_invalidated` is set when a connection that **had already been established** dies. A
connection that never opened at all leaves it `False`. That is exactly the line between "nothing
ran" and "something may have run".

So the order matters absolutely:

```
   check connection_invalidated  FIRST   -> UNKNOWN_OUTCOME (not safe to retry)
   check the SQLSTATE            SECOND  -> BACKEND_UNAVAILABLE (safe to retry)
```

Reversed, every mid-flight disconnect would be reported as safely retryable. That is how duplicate
writes happen.

`docs/failure-modes.md` records that inverting the order was tried and made **three tests fail**.
`test_invalidation_outranks_the_sqlstate` pins it.

## 26.7 Degradation rather than failure

Two paths degrade instead of erroring.

**The embedder is down.** `hybrid_search` catches `EmbeddingError`, leaves the semantic ranking
empty, and sets `degraded = "lexical_only: <reason>"`. RRF over one list is just that list's order,
so no special case is needed.

**The semantic query is cancelled by `statement_timeout`.** The semantic leg is the expensive one —
an HNSW probe over the whole project — so it is the leg that hits the timeout first. When it does,
the lexical results are already in hand, and half an answer beats an error. Only SQLSTATE `57014`
degrades; any other `DBAPIError` is re-raised.

There is a subtlety written into that `except` block, and it is worth reading:

> "Nothing after this point issues another statement, which matters: PostgreSQL has aborted the
> transaction, so any further query would fail with 25P02. It holds because `missing` below is only
> ever populated by semantic-only hits, and there are none when the semantic leg produced nothing."

That is exactly the kind of reasoning a later refactor breaks without realising, which is why it is
a comment in the code rather than only in a document.

`test_an_unrelated_database_error_is_not_swallowed` exists because catching every `DBAPIError`
there would turn a genuine bug in the vector query — a bad cast, a missing column — into a search
that silently returns lexical results forever while reporting itself as merely degraded. That is a
worse outcome than a crash, since nothing would ever surface it.

## 26.8 A correction the repository made about itself

`docs/architecture.md` §9 said the degraded marker would be `fts_only`. The implementation uses
`lexical_only: <reason>`. `docs/failure-modes.md` records the correction explicitly.

More significantly, that same document records that three of the codes §9 promised —
`BACKEND_UNAVAILABLE`, `BACKEND_BUSY`, `UNKNOWN_OUTCOME` — **did not exist anywhere in the code**
until Milestone 9. Driver failures reached the model as opaque internal errors. Writing the
failure-modes document is what found the gap:

> "the prose had been true when it was written and had quietly stopped being true, and nothing else
> would have noticed."

## 26.9 The timeouts

Two, and they do different jobs:

**`pool_timeout = 2.0` seconds.** How long to wait for a free connection. Bounded deliberately.
Unbounded queueing converts a slow backend into an unbounded latency tail: requests pile up
holding memory, and by the time the backlog drains the callers have all given up anyway. Refusing
quickly at least tells the caller what happened while it is still true.

**`statement_timeout = 5000` milliseconds**, applied **server-side** through
`connect_args={"server_settings": {...}}`. A runaway query is killed by PostgreSQL itself.
Relying on an application-side timeout leaves the backend still burning CPU on a query nobody is
waiting for any more. `test_the_configured_timeout_reaches_the_backend` reads
`pg_settings` to confirm it actually arrived.

Also `pool_pre_ping=True`, which checks a pooled connection is alive before handing it out, turning
"the database restarted" from an error into a transparent reconnect.

## 26.10 Schema drift

`src/memhub/persistence/schema.py::verify` runs at startup and refuses to start in **both**
directions:

- **Database behind the code** — "run `alembic upgrade head`". Without this, the server starts
  happily and fails on the first query touching a missing column, which the client reports as the
  tool being broken. Nobody looks at migrations.
- **Database ahead of the code** — refuse, and say to upgrade the code rather than downgrade the
  database. This is the more dangerous direction and the one easy to get wrong: a newer schema
  mostly works, until this process writes a row that newer constraints were added to prevent. The
  failure then surfaces later, somewhere else, as corrupt data rather than an error.

Tests: `test_an_unmigrated_database_is_refused_with_the_remedy`,
`test_an_unknown_revision_is_refused_as_too_new`, `test_a_migrated_database_verifies`.

---

# PART 27 — PROJECT ISOLATION

## 27.1 The requirement

Memory belonging to **Project A** must not appear while operating inside **Project B**. Not
"should usually not". Must not.

## 27.2 Tracing the project scope through every layer

**Diagram — project scope from input to schema**

```
 MCP input        every memory tool takes project_id as a REQUIRED argument
      |           (verified in tests/protocol/manifest.json: project_id is in
      |            the input properties of all six memory_* tools)
      v
 Handler          _parse_uuid(project_id, "project_id")
      |
      v
 Service          every service function's first positional parameter after the
      |           session is project_id
      v
 Repository       "Every method takes project_id as its first parameter. Not as
      |           an optional filter - as a required argument, so there is no
      |           signature in this class that can express a cross-project read."
      v
 SQL              WHERE m.project_id = :project_id
      |           applied by current_revisions() in EVERY mode, including
      |           include_retired=True
      v
 Schema           UNIQUE (id, project_id) on memories
                  composite FK (memory_id, project_id) on memory_revisions,
                    memory_dedup_keys, memory_attestations
                  composite FK (superseded_by_id, project_id) on memories itself
                  PK (project_id, client_request_id) on idempotency_keys
                  PK (project_id, hash_version, content_hash) on dedup keys
```

Four independent layers. You would have to defeat all four.

## 27.3 The strongest one

```python
ForeignKeyConstraint(
    ["superseded_by_id", "project_id"],
    ["memories.id", "memories.project_id"],
)
```

A memory can only be superseded by a memory in the same project — **and that is not a rule the
application enforces, it is a state the database cannot represent**.

`test_cross_project_supersession_is_unrepresentable` proves it by writing the forbidden `UPDATE`
in raw SQL, bypassing the service layer entirely, and asserting the `IntegrityError` names
`fk_memories_superseded_by_id_project_id_memories`.

## 27.4 What a caller sees when it tries

Two different behaviours, both deliberate.

**Attempting to supersede across a project** returns the target in `not_superseded`, rather than
raising. The compare-and-set is project-scoped, so the attempt simply matches zero rows. The caller
gets a clear answer.
`test_supersession_cannot_cross_a_project` also asserts the foreign memory is untouched in its own
project.

**Attempting to read a memory from another project** raises `MEMORY_NOT_FOUND`. And the docstring
for that error is careful:

> "Deliberately indistinguishable from 'exists but belongs to another project'. Project isolation
> is a boundary, so a lookup outside the caller's project must not confirm that an id exists
> elsewhere."

That is a small, correct security decision: not leaking existence.

## 27.5 Isolation in the retrieval paths

- `current_revisions(project_id)` applies `Memory.project_id == project_id` **before** the
  `include_retired` branch, so even the debug path is scoped.
- `semantic.search_by_vector` carries `AND m.project_id = :pid` in its raw SQL.
- `test_hybrid_respects_project_isolation` asserts the vector leg does not leak either.
- The evaluation corpus deliberately contains a **second project** as a trap — memories that would
  be plausible answers if isolation leaked. Those appear in the graded queries' `forbidden` lists,
  and `stale_inclusion_rate` is computed over **all** queries including those traps, precisely
  because a cross-project leak is exactly as serious as a stale-memory leak.

## 27.6 Isolation of the other keys

- **Idempotency keys** are `(project_id, client_request_id)`. Keys are caller-generated, so a client
  using a counter rather than a UUID would otherwise have its second project silently replay the
  first project's memories. `test_keys_are_scoped_to_a_project` covers it.
- **Dedup keys** are `(project_id, hash_version, content_hash)`. Project A and Project B may each
  hold "The job queue runs on Redis." without interfering.

## 27.7 The one place isolation is deliberately relaxed, and why it is still safe

The resource URIs `memory://memories/{memory_id}` and `.../history` carry no project. The handler
resolves the owning project first (`_owning_project`) and then uses it as the scope for the actual
read.

The comment explains why that is safe: a memory id is an unguessable UUID, and the project it
resolves to is then used as the scope, so a caller still cannot reach across a project boundary —
they can only read a memory whose id they already hold.

## 27.8 The mis-resolution problem, which is the subtler half

Isolation is not only about queries. It is also about **not silently forking the corpus**.

`services/projects.use_project` resolves every hint independently and requires them to agree:

- Two hints pointing at different projects → `AMBIGUOUS_PROJECT`, listing both.
- A named slug that does not exist, while a git remote points at some *other* project →
  `AMBIGUOUS_PROJECT`, because returning the hint's project would hand the caller a namespace they
  did not ask for and silently write another project's memories into it.
- Nothing matches and `create=false` → `PROJECT_NOT_FOUND` with an instruction, never a silent
  create.

And `project_aliases` has `UNIQUE (kind, value_norm)` **globally**, so an alias value can never
resolve to two projects. Ambiguity is impossible by construction rather than by careful query
writing.

---

# PART 28 — PROVENANCE

## 28.1 What provenance means

**Provenance** — where a piece of information came from.

For a shared memory system this is not decoration. A memory only means something because someone
decided it was worth keeping, and the answer to "why does this system contain this sentence?" has
to exist.

## 28.2 What is actually persisted

Verified against `src/memhub/persistence/models.py`. No invented columns.

**On `memory_revisions` — per revision, so provenance is per version, not per memory:**

| Column | Type | What it holds |
|---|---|---|
| `author_client` | text, NOT NULL | the client name the caller supplied, e.g. `claude-desktop`, `cursor`, or `unknown` |
| `author_kind` | text, NOT NULL, CHECK in (`agent`, `human_confirmed`, `import`) | whether a human explicitly asked for this to be remembered |
| `source` | text, nullable | free text, e.g. `"architecture discussion"` |
| `created_at` | timestamptz, NOT NULL, default `now()` | the database clock |
| `change_reason` | text, nullable | why a revision was made |

Note that `author_client` and `author_kind` are **per revision**. Revision 1 can be authored by
`claude-desktop` and revision 2 by `cursor`, and both are preserved. That is asserted in
`test_history_is_preserved_not_overwritten`.

**On `memories` — lifecycle provenance:**

`created_at`, `updated_at`, `superseded_at`, `superseded_by_id`, `deleted_at` — all timestamps from
`now()`.

**In `memory_attestations` — corroboration:**

`client_name`, `times_seen`, `first_seen_at`, `last_seen_at`, keyed `(memory_id, client_name)`.

This answers a question no other column can: **which distinct clients independently asserted this
fact?** "Claude Desktop and Cursor both recorded this" is a stronger statement than "one client
recorded it twice", and keying on client name is what keeps those apart.

**In `audit_events` — the action trail:**

`at`, `action`, `outcome`, `actor_client`, `request_id`, and a JSONB `detail`. **Never the
content** — only identifiers, outcomes, and sizes, so the log can be read freely without exposing
what the memories say.

## 28.3 `author_kind` replaces a memory type

Worth knowing, because it shows a modelling decision. The original specification had an
`OBSERVATION` memory type. It was cut, and `author_kind` covers it instead: "the agent noticed
this" versus "a human confirmed this" is a statement about **provenance and trust**, not about what
*kind of thing* the memory is.

## 28.4 Where provenance surfaces

- **`MemoryOut`** — every search and read result carries `author_client`, `author_kind`, `source`.
- **`memory_history`** — every revision with its own author, plus attestations, plus the audit
  trail.
- **The context brief** — `render_brief` adds a provenance line under each memory:

```
- The job queue runs on PostgreSQL. Redis was removed.
  _confirmed by the user; from architecture discussion; recorded by cursor_
```

The rendering module's docstring gives the reason: "Two clients independently recorded this" and
"an agent noticed this once" are different claims, and a brief that flattens them invites the
reader to treat them alike.

## 28.5 Tests

- `test_two_sessions_share_state_over_stdio` asserts `author_client`, `author_kind`, and `source`
  all survive the process boundary.
- `test_history_still_shows_the_retired_fact` asserts the retired memory's original author is still
  visible: `record.revisions[0].author_client == "claude-desktop"`.
- `test_deduplication_is_reported_as_corroboration` covers the attestation path.
- `test_the_audit_record_outlives_the_memory` and `test_earlier_audit_detail_is_redacted` cover the
  audit trail's behaviour under purge.

## 28.6 What is NOT stored as provenance

Be precise here, because it is easy to overclaim:

- No user identity. There is no user table, no authentication, no notion of *who* the human was.
- No conversation reference. Nothing links a memory back to the exchange that produced it.
- No file or line reference.
- `author_client` is **caller-supplied and unverified.** A client can send any string. It defaults
  to `"unknown"`. The metrics layer collapses anything outside an allow-list to `"other"` to avoid
  a cardinality explosion, but the stored value is whatever was sent. This is fine for the
  single-user local deployment; it would not be evidence of anything in a multi-tenant one.

---

# PART 29 — TESTING STRATEGY

## 29.1 What is actually there

**Counted directly from the repository:** 319 test functions across 30 test modules, in seven
directories. `README.md` reports 369 tests, which is a *collected* count — several modules use
`@pytest.mark.parametrize`, and one decorated function produces several collected tests. The two
numbers are consistent with each other; 319 is the number of `def test_` definitions.

I could not run `pytest` during this review, because it requires a live PostgreSQL instance, so I
am reporting the counts I could verify statically rather than a pass/fail result.

| Directory | Modules | Test functions | Needs PostgreSQL |
|---|---|---|---|
| `tests/unit/` | 10 | 133 | no |
| `tests/integration/` | 10 | 109 | yes |
| `tests/protocol/` | 3 | 36 | yes |
| `tests/failure/` | 1 | 19 | yes |
| `tests/concurrency/` | 2 | 12 | yes, with a large pool |
| `tests/eval/` | 2 | 7 | yes; 3 also need a real model |
| `tests/perf/` | 2 | 3 | yes, deselected by default |

## 29.2 The test harness

`tests/conftest.py`. Two layers of isolation:

**Template databases.** Migrations run **once per session** into a template database. Each test
module then does `CREATE DATABASE ... TEMPLATE`, which PostgreSQL implements as a file copy.
Running the full migration chain per module would grow linearly with the number of migrations;
cloning does not.

**Transaction rollback.** Within a module, each test gets a session wrapped in a transaction that
is rolled back afterwards. Fast, and adequate for everything that is not concurrent.

**The stated limitation.** The rollback fixture **cannot** be used by concurrency tests. Two
writers inside one transaction are not two writers. Those tests take the module database directly
and manage their own transactions.

**A CI safeguard worth noting.** Locally, a missing database *skips* integration tests with an
actionable message. In CI, `MEMHUB_REQUIRE_DB=1` turns that skip into a hard failure, so a broken
service container can never be mistaken for a green build.

## 29.3 Layer by layer

### Unit tests — `tests/unit/`

**Definition.** Test one piece in isolation, with no database and no network.

**Why necessary.** They are fast, they are deterministic, and they can be checked against worked
examples where you know the right answer by hand.

**Files and what each covers:**

| File | Tests | Subject |
|---|---|---|
| `test_eval_metrics.py` | 25 | nDCG, recall, precision, MRR, stale inclusion, against hand-computed examples |
| `test_validation.py` | 25 | slug, content, tags, importance, expiry rules |
| `test_error_classification.py` | 16 | the four failure codes and the branch order |
| `test_metrics.py` | 13 | the metrics registry and its forbidden-label rule |
| `test_context_builder.py` | 13 | quotas, MMR, greedy fill, deterministic ordering |
| `test_fusion.py` | 12 | RRF arithmetic and tie-breaking |
| `test_config.py` | 9 | settings validation |
| `test_logging.py` | 7 | JSON format, and that nothing reaches stdout |
| `test_normalize.py` | 7 | content normalisation, git-remote and path normalisation |
| `test_filter_sql.py` | 6 | the compiled SQL of the stage-0 filter |

**Mocked:** nothing, really — these modules are pure functions. **Real:** the logic itself.

**A representative one.** `test_filter_sql.py::test_is_current_is_a_bare_boolean` compiles the
stage-0 filter to a SQL string and asserts it does not contain `IS TRUE`. See Part 17.8.

### Integration tests — `tests/integration/`

**Definition.** Several pieces together, against real PostgreSQL.

**Why necessary.** Everything interesting here is a database behaviour.

| File | Tests | Subject |
|---|---|---|
| `test_memories.py` | 18 | remember, revise, forget, search, history, at the service layer |
| `test_lexical_search.py` | 14 | full-text matching, ranking, the any-term fallback |
| `test_embedding_outbox.py` | 12 | enqueue, drain, SKIP LOCKED, backoff, DEAD, coverage |
| `test_projects.py` | 12 | resolution, ambiguity, aliases, creation |
| `test_context.py` | 11 | budget adherence, quotas, diversity, stale exclusion at every budget |
| `test_stale_memory.py` | 11 | supersession, forget, dedup-key release |
| `test_invariants.py` | 10 | constraints, and the eight-query invariant suite |
| `test_database.py` | 9 | engine, pool, timeouts, connectivity |
| `test_hybrid_search.py` | 8 | fusion, hydration, coverage, degradation, isolation |
| `test_migrations.py` | 4 | upgrade, downgrade, and ORM-vs-schema drift |

**Mocked:** the embedder, sometimes, using the deterministic fake or a deliberately `BrokenEmbedder`.
**Real:** the database, always. There is no mocked database anywhere in the suite.

**A representative one.** `test_the_retired_fact_leaves_retrieval_entirely` — Part 21.6.

### Concurrency tests — `tests/concurrency/`

**Definition.** Several writers at the same time, on genuinely separate connections.

**Why necessary.** A lost update cannot be demonstrated with one writer.

**What is real:** everything — real PostgreSQL, real separate connections, real barriers, real row
locks.

**A representative one.** `test_exactly_one_of_fifty_writers_wins` — Part 11.

**And the fixture that makes it honest.** `assert_backends_are_distinct` proves the pool really
hands out 50 separate backends. Without it the test would pass for the wrong reason.

### Protocol tests — `tests/protocol/`

**Definition.** Drive the MCP interface and check the contract.

Two kinds:

**In-process** (`test_mcp_tools.py`, 31 tests) — build a server against the test database and drive
it with an in-process `Client`. Fast enough to cover the whole tool surface: outcome
discriminators, tool errors versus protocol errors, resource URIs, cross-project isolation,
conflicts, dedup, history, the stale-memory demo through MCP.

**Real subprocess** (`test_stdio_transport.py`, 2 tests) — spawn `python -m memhub.mcp` as an
actual subprocess and talk over stdin/stdout. Slower, marked `stdio`, and worth it because it is
the only way to prove three things the in-process tests cannot:

1. the console script and its entry point actually work;
2. **stdout carries nothing but JSON-RPC** — any log line or stray `print` would break the framing
   and the handshake would fail;
3. two *separate sessions*, the second started after the first has fully exited, share state
   through PostgreSQL and nothing else.

**Golden manifest** (`test_manifest_snapshot.py`, 3 tests) — see Part 30.

### Failure tests — `tests/failure/`

**Definition.** Deliberately cause a failure and check the response.

`test_failure_modes.py`, 19 tests, grouped: `TestDatabaseUnavailable`, `TestPartialWrites`,
`TestSchemaDrift`, `TestOperatorPurge`, `TestRetention`.

**A representative one.** `test_purge_clears_every_derived_table` — content is copied into
embeddings, dedup keys, and attestations, and a purge that missed one of them would leave a leaked
credential recoverable while reporting success. A partial erasure is not an erasure.

**`docs/failure-modes.md`** maps every row of the architecture's failure matrix to the test that
covers it, and is candid about the three rows where no test can and a structural argument has to
do instead. Those three are named explicitly: the server being killed mid-request, the client
disconnecting mid-call, and an embedding being partially written. In each case the document
explains that a test there would be testing PostgreSQL's atomicity rather than this system, and
points at `test_a_rolled_back_write_leaves_no_job` as the place the underlying claim is actually
checked.

### Evaluation tests — `tests/eval/`

**Definition.** Measure retrieval quality against a graded dataset, and fail the build on a
regression.

`test_retrieval_quality.py` (4 tests) runs the lexical path with no model, and is part of the
default suite. `test_hybrid_quality.py` (3 tests) is marked `real_embeddings` and deselected by
default, because it needs a real model download and CPU inference.

**A representative one.** `test_quality_has_not_regressed` compares the current run against
`eval/dataset/baseline.json` with a tolerance of 0.02 and fails the build if a metric drops
further.

### Performance tests — `tests/perf/`

**Definition.** Measure latency and how cost grows with corpus size.

Marked `perf` and **deselected by default** — and the reason in `pyproject.toml` is worth reading:

> "The latency benchmark measures a query; running it alongside 280 other tests measures contention
> with them instead, and produced a p95 that failed in the full suite and passed in isolation. A
> benchmark that shares a machine with the rest of the suite is not measuring what it claims."

## 29.4 Why PostgreSQL-dependent correctness is tested with real PostgreSQL

The list of guarantees this project makes that **do not exist in SQLite**:

- partial unique indexes;
- `EvalPlanQual` re-check semantics under READ COMMITTED;
- `FOR UPDATE SKIP LOCKED`;
- `FOR SHARE` blocking until a transaction resolves;
- `ON CONFLICT DO NOTHING` waiting on an uncommitted insert;
- GIN indexes, `tsvector`, `websearch_to_tsquery`, `ts_rank_cd`;
- HNSW and pgvector's distance operators;
- composite foreign keys used to make a state unrepresentable;
- `EXPLAIN ANALYZE` output.

A test that passed on SQLite would prove nothing about the system actually being shipped. The
conftest docstring says exactly this.

**Diagram 23 — testing**

```
                      no database          real PostgreSQL
                      -----------          ---------------
   unit/         133 tests
     pure logic, hand-checked

   integration/                            99 tests
     services against the real schema

   concurrency/                            12 tests, own connections,
                                           barrier-gated, pool asserted

   protocol/                               34 in-process + 2 real subprocess

   failure/                                19 tests, injected failures

   eval/                                   4 lexical (+3 with a real model)

   perf/                                   3 tests, deselected by default
```

---

# PART 30 — MUTATION TESTING AND NEGATIVE VERIFICATION

## 30.1 The idea

**Mutation testing** — deliberately break a mechanism and confirm that a test notices.

**Why it matters.** A passing test does not prove it was checking anything. A test can pass because
the code is right, or because the assertion is vacuous, or because the test never reaches the code
path it claims to cover. Breaking the mechanism and watching the test fail is the only real
evidence.

## 30.2 What the repository reports

Three mutation experiments are recorded, in `README.md` and `docs/failure-modes.md`.

**1. The stage-0 filter.** Removing the `status = 'ACTIVE'` condition from
`retrieval/filters.py::current_revisions` makes **five** of the stale-memory tests fail.

What that proves: those tests genuinely depend on the structural filter, not on ranking happening
to bury the retired memory.

**2. The compare-and-set predicate.** Dropping `AND current_revision_no = :expected_revision` from
`cas_revise.sql` makes the concurrency tests fail.

What that proves: the version predicate is load-bearing. Without the experiment, you could not
distinguish "the CAS works" from "nothing ever collides in this test".

**3. The failure classifier's branch order.** Inverting the `connection_invalidated` check and the
SQLSTATE check makes **three** tests fail.

What that proves: the ordering is not incidental. Reversed, every mid-flight disconnect would be
reported as safely retryable, which is how duplicate writes happen.

## 30.3 The finding that came out of one of them

This is the best part, and it is visible in the code.

`test_losers_are_told_what_beat_them` contains this:

```python
    # Assert the population before iterating it. Without this, the loop below
    # passes vacuously when there are no losers at all - which is exactly what
    # an experiment removing the version predicate from the CAS produced.
    assert len(losers) == CONCURRENCY - 1

    for loser in losers:
        assert loser.expected_revision == 1
        ...
```

The mutation experiment revealed a **vacuous test**. With the CAS predicate removed, every writer
succeeded, so there were **zero** losers, so the `for` loop body never executed, so every assertion
inside it passed trivially — and the test was green while the mechanism it was meant to protect was
completely absent.

The fix is one line: assert the population size before iterating it.

That is the single most useful thing mutation testing does — not confirming that good tests are
good, but finding tests that were never checking anything.

**Diagram 24 — the mutation experiment**

```
   1. mechanism intact          ->  tests pass
   2. break the mechanism       ->  tests MUST fail
   3. tests still pass?         ->  the tests were not checking it. Fix the tests.
   4. restore the mechanism     ->  tests pass again

   Step 3 is the whole point. Steps 1 and 4 prove nothing on their own.
```

## 30.4 The other negative-verification technique: the golden manifest

`tests/protocol/test_manifest_snapshot.py` captures everything a client can observe about the
interface *before* calling it — tool names, titles, **descriptions**, input property names,
resource URIs, and the server instructions — and compares it against the committed
`manifest.json`.

**Why this matters, and it is not obvious.** Tool descriptions are not documentation. They are the
**prompt that steers the model**. The description is what decides whether the model stores a
credential, whether it passes `supersedes` instead of recording a contradicting fact, and whether
it treats a conflict as recoverable.

Change one word and system behaviour changes, with **no logic change and no other test failing**.
The snapshot turns that into a visible diff in code review, which is where a change to the model's
instructions belongs.

Regenerating it is deliberate: `MEMHUB_UPDATE_MANIFEST=1 pytest tests/protocol/test_manifest_snapshot.py`,
then read the diff before committing.

There is a related test, `test_descriptions_carry_the_operating_contract`, which asserts specific
load-bearing sentences are present — the instruction never to store credentials, and the
instruction to state a rejected alternative inside a decision rather than as a standalone fact.

**A caveat, and it is a real one.** The snapshot pins the descriptions to whatever is committed. It
does not check that the descriptions are *accurate*. As Part 5.5 records, `memory_search`'s
description still says "The optional query is a substring match today", which stopped being true
when full-text search shipped. The snapshot faithfully preserved a stale sentence. That is the
mechanism working exactly as designed, and it is also the limit of what it can do.

## 30.5 The evaluation baseline as a regression gate

The same idea, applied to quality. `eval/dataset/baseline.json` holds committed metrics.
`test_quality_has_not_regressed` fails the build if any of them drops by more than 0.02.

The tolerance is not zero, deliberately: the ranking depends on a recency prior computed against
the database clock, so scores move fractionally between runs. Zero tolerance would make the test
flaky; a large tolerance would let a real regression through unnoticed.

And `test_no_query_returns_a_retired_or_foreign_memory` asserts `stale_inclusion_rate == 0.0` with
**no tolerance at all**, because that is a correctness metric rather than a quality one.

---

# PART 31 — RETRIEVAL EVALUATION

## 31.1 The metrics, from first principles

Suppose a query has three genuinely relevant memories, `M1`, `M2`, `M3`, and the system returns ten
results.

### Precision@10

**Definition.** Of the results returned, what fraction are relevant.

```
   returned:  [M1, X, X, M2, X, X, X, X, X, X]
   relevant among them: 2
   Precision@10 = 2 / 10 = 0.20
```

**What it tells you.** How much of what you got is worth having. Low precision means noise.

**Why it matters here specifically.** It is what a token budget actually spends. At `k=5` with a
2000-token budget, an irrelevant result does not merely add noise — it **displaces** something
useful.

### Recall@10

**Definition.** Of all the relevant memories that exist, what fraction appeared in the top 10.

```
   relevant in the corpus:  M1, M2, M3   (3)
   found in the top 10:     M1, M2       (2)
   Recall@10 = 2 / 3 = 0.667
```

**What it tells you.** Whether you found them at all. Low recall means the system is blind to
things it should have surfaced.

**Precision and recall pull against each other.** Return everything and recall is perfect while
precision is terrible. Return one very safe result and precision is perfect while recall is
terrible.

### nDCG@10

Build it up in four steps.

**Step 1 — graded relevance.** Not every relevant result is equally relevant. This project grades
0, 1, or 2:

```
  2 = directly answers the question
  1 = useful context a good result set would include
  absent = irrelevant
```

**Step 2 — gain.** Convert a grade to a number. This project uses the **exponential** form:

```
   gain = 2^relevance - 1

   relevance 2  ->  gain 3
   relevance 1  ->  gain 1
   relevance 0  ->  gain 0
```

Why exponential rather than the raw grade: it makes the difference between "directly answers" and
"merely useful" 3 versus 1, rather than 2 versus 1. A memory that answers the question and one that
merely relates to it are not two-thirds as different as the raw grades suggest.

**Step 3 — discount by position.** A relevant result at position 1 is worth more than the same
result at position 9:

```
   DCG = sum over positions p of   gain(p) / log2(p + 2)
```

(The `+2` is because positions are counted from 0 in the implementation, so the first position
divides by `log2(2) = 1`.)

**Step 4 — normalise.** Compute the DCG of the *ideal* ranking — all the relevant memories sorted
best-first — and divide:

```
   nDCG = DCG(actual) / DCG(ideal)
```

Which puts the result in `[0, 1]`, where 1.0 means "perfect ordering".

**Worked example.**

```
   Judgments:   M1 = 2,  M2 = 1,  M3 = 1
   Returned:    [M2, X, M1, ...]

   Ideal ranking:  [M1(2), M2(1), M3(1)]
     DCG_ideal = 3/log2(2) + 1/log2(3) + 1/log2(4)
               = 3/1.000 + 1/1.585 + 1/2.000
               = 3.000 + 0.631 + 0.500 = 4.131

   Actual:  position 0 = M2 (gain 1), position 1 = X (gain 0), position 2 = M1 (gain 3)
     DCG_actual = 1/log2(2) + 0/log2(3) + 3/log2(4)
                = 1.000 + 0 + 1.500 = 2.500

   nDCG@10 = 2.500 / 4.131 = 0.605
```

**Why nDCG is the primary metric here.** MRR credits only the position of the *first* relevant
result. These queries usually have several relevant memories at different degrees of relevance — a
decision that directly answers the question, plus a constraint that qualifies it — and MRR would
score "perfect answer first, everything else missing" identically to "perfect answer first,
everything else present". MRR is still computed and reported, because it is cheap and occasionally
illuminating, but it is not what a change is judged on.

### stale_inclusion_rate

**Definition.** The fraction of queries where a memory that must never be returned appeared in the
top k.

**Target: exactly 0.**

This is the metric the project exists for, and it is reported **separately** on purpose. It is not
a quality metric to be traded off against the others. A retrieval change that improved nDCG by 0.03
while surfacing one retired memory is a **regression**, not an improvement, and averaging it into a
quality score would hide that.

It is computed over **all** queries, including the unanswerable ones and the cross-project traps,
because a leak there is exactly as serious.

### empty_for_unanswerable

**Definition.** The fraction of unanswerable queries that correctly returned nothing. Target 1.0.

A query the corpus cannot answer should produce silence, not ten confident-looking irrelevant
results.

## 31.2 The dataset, verified

From `eval/dataset/` and `tests/eval/dataset.py`:

- **200 memories total** (`TOTAL_CORPUS_SIZE = 200`).
- A hand-written core in `memories.yaml`, padded with deterministically generated **distractors**
  (`DISTRACTOR_SEED = 20260601`) to reach 200. Distractors exist so queries face genuine
  competition; nothing judges them, and a distractor appearing in a top-10 result simply costs
  precision, which is the correct penalty.
- **34 queries, of which 31 are answerable.** The other three have no relevant memory at all and
  exist to test whether the system knows when to say nothing.
- **Graded relevance** per query: `relevant: {m102: 2, m103: 1}`.
- **A `forbidden` list** per query, naming memories that must never appear.
- **Two projects.** A second project holds memories that would be plausible answers if isolation
  leaked.
- **Deliberate difficulty**, stated in the corpus file's own comments: superseded pairs where the
  retired memory matches the query wording *better* than its replacement; near-misses that share
  vocabulary but answer a different question; and cross-project traps.

The seeding applies supersession **through the real `remember(supersedes=...)` path**, not by
writing a status directly — otherwise the evaluation would be measuring a state the system cannot
actually produce.

**The judgments were written before any strategy was measured against them.** That ordering is the
methodological point of Milestone 6.

## 31.3 The results

| Strategy | nDCG@10 | Recall@10 | Precision@10 | MRR | Stale | Empty for unanswerable |
|---|---|---|---|---|---|---|
| Full text, all terms required | 0.478 | 0.468 | 0.484 | 0.484 | **0.000** | 0.667 |
| Full text, any-term fallback | 0.802 | 0.817 | 0.691 | 0.823 | **0.000** | 0.667 |
| Hybrid: FTS + pgvector, RRF, distance ≤ 0.35 | **0.853** | **0.828** | 0.671 | 0.914 | **0.000** | 0.333 |

The committed baseline (`eval/dataset/baseline.json`) is the middle row, because that is the
strategy CI runs — the hybrid numbers were measured locally with a real model.

**The weakest queries**, recorded rather than hidden:

| Query | nDCG (full text) | Note |
|---|---|---|
| `q13` "deadlock prevention" | 0.000 | the matching memory describes it without using the phrase; hybrid does not fix this either |
| `q22` "jwt" | 0.000 | the Snowball stemmer never matches `JWTs`; hybrid takes this to **1.000** |
| `q31` "what is being worked on right now" | 0.000 | |
| `q29` "how is the server deployed" | 0.174 | |
| `q11` "how are concurrent updates handled" | 0.562 | |

## 31.4 What the evaluation demonstrates

- The retrieval is competent: nDCG 0.853 with hybrid on a graded 200-memory corpus.
- Hybrid genuinely beats full text on ranking and recall, and the delta is published.
- The distance threshold was chosen by measurement, and the sweep that produced it is committed.
- **A retired memory reached a caller zero times**, at every strategy, every token budget, and every
  query — including one built specifically to defeat a similarity-only system.
- Quality is gated: a regression fails the build rather than going unnoticed.

## 31.5 What the evaluation cannot prove

Be able to say all of this unprompted.

- **200 memories is small.** Retrieval behaviour at 100,000 memories with genuinely competing
  content is not measured by this.
- **The corpus is synthetic and written by one person.** It reflects one author's idea of what
  project memories look like and what a query for them looks like.
- **The judgments are one person's opinion.** There is no second annotator and therefore no
  inter-annotator agreement figure.
- **34 queries is a small sample.** A 0.05 nDCG difference on 31 answerable queries is not a
  statistically robust result, and the document does not claim it is.
- **The distractors are template-generated**, so they are less confusing than real near-miss content
  would be.
- **The hybrid numbers were measured locally, once**, with one embedding model, not in CI.
- **It says nothing about whether the *right things were recorded*.** It measures retrieval over a
  given corpus, not the quality of what a model chose to remember.

## 31.6 The stale-memory result, separately

This deserves its own statement because it is a different kind of claim.

`stale_inclusion_rate = 0.000` across all three strategies, all budgets, all queries.

**Why that number is not surprising, and why that is the point.** It is not the result of good
ranking. It is the result of the retired memory never being in the candidate set. The number would
be 0.000 even if the ranking were terrible, and it stayed 0.000 when the ranking changed
dramatically between the three strategies. A guarantee that survives a change of retrieval strategy
is a structural guarantee.

The hardest single case is query `q02`, literally `"redis"`, where the retired memory is about
Redis and mentions it twice and the current one mentions it only to say it was removed. Its note in
`queries.yaml` says: "Lexical relevance alone would rank the retired one first, which is why
suppression has to be structural rather than a ranking effect."

---

# PART 32 — PERFORMANCE

## 32.1 The words

**Latency** — how long an operation takes.

**Server latency** — PostgreSQL's own execution time, from `EXPLAIN ANALYZE`. This is the part the
design controls.

**Client latency** — the time measured from Python, which includes the network round trip and, on
this development machine, Docker Desktop's port forwarding.

**p50 / p95** — the median and the 95th percentile. p95 means "95% of calls were at least this
fast", and it is a better summary of user experience than an average, which one slow outlier can
distort.

**Cold** — the first call after a bulk load, reading from disk rather than from cache.

**Query plan** — PostgreSQL's chosen strategy for a query, visible with `EXPLAIN`.

**Sequential scan** — reading the table row by row.

**Index scan** — using an index to jump to the relevant rows.

## 32.2 The scaling benchmark

`docs/perf/scaling.txt`, produced by `tests/perf/test_scaling.py` at three corpus sizes:

```
  1,000 memories   matched=5     server=0.36ms   client= 5.34ms
 10,000 memories   matched=50    server=0.53ms   client=11.00ms
100,000 memories   matched=500   server=0.67ms   client=22.80ms
```

**Read it carefully.** The corpus grew **100×**. Server-side query time grew **1.9×**.

**The property that matters is sub-linear growth**: cost grows with the size of the *answer*, not
the size of the *table*. That is the property an index exists to provide.

**The gap between server and client time is Docker Desktop's port forwarding on the development
machine, not query cost.** The benchmark reports both separately rather than folding the overhead
into one misleading number.

The test asserts `SERVER_BUDGET_MS = 50.0` at every scale, and the fixture comment explains why
that assertion is meaningful: a query whose cost is linear in the corpus would pass comfortably at
1k and fail at 100k.

## 32.3 The latency benchmark

`docs/perf/search_latency.txt`, at 10,000 memories:

```
quiescence         matched=  50  server= 3.90ms  cold= 21.92ms  p50=16.67ms  p95=22.95ms
advisory locks     matched= 685  server=11.52ms  cold= 21.53ms  p50=24.07ms  p95=42.93ms
postgresql         matched= 685  server= 7.17ms  cold= 26.67ms  p50=18.70ms  p95=23.73ms
outbox table       matched= 738  server= 8.75ms  cold= 16.65ms  p50=23.07ms  p95=35.28ms
retry              matched= 614  server= 6.25ms  cold= 20.97ms  p50=15.59ms  p95=28.21ms
```

Budgets: server under 50 ms, client warm p95 under 150 ms. Both are met comfortably.

## 32.4 The query plan, and a lesson about asserting on plans

`docs/perf/scaling_plan.txt` records the `EXPLAIN ANALYZE` at 100,000 memories. At that size,
PostgreSQL chooses a **sequential scan** over the GIN index — and it is **right to**.

```
Limit  (cost=0.42..252.77 rows=10) (actual time=0.082..0.814 rows=10 loops=1)
  ->  Nested Loop  (actual time=0.080..0.809 rows=10 loops=1)
        ->  Seq Scan on memory_revisions r  (actual time=0.059..0.713 rows=10 loops=1)
              Filter: (is_current AND (content_tsv @@ '''quiescenc'''::tsquery)
                       AND (project_id = '...'::uuid))
              Rows Removed by Filter: 1563
        ->  Index Scan using pk_memories on memories m  (actual time=0.008..0.008 rows=1 loops=10)
Planning Time: 0.528 ms
Execution Time: 0.850 ms
```

**Why the sequential scan is correct here.** With `LIMIT 10`, the scan stops as soon as it has ten
matches. Note `Rows Removed by Filter: 1563` — a small fraction of a 100,000-row table. An index
lookup plus the heap fetches to get the rows would cost more than simply reading until satisfied.
The index earns its keep on queries that must examine *every* match, not on ones that stop early.

**And the lesson recorded in the file itself:**

> "Recorded, not asserted on. Two earlier versions of this benchmark did assert that the GIN index
> appears in the plan, and both were wrong for the same reason: PostgreSQL keeps finding cheaper
> ways to answer the query than the one the test expected."

That is a genuinely good engineering story. The right assertion is on the **property** (cost grows
with the answer, not the corpus), not on the **mechanism** (which access path the planner picks).

There is a related detail in `MemoryRepository.explain_search`, which takes a `force_index` flag
that sets `enable_seqscan = off`. That is not how the query runs in production; it is a way to ask
a different question: *can* the planner reach the index at all? A plan showing a sequential scan is
ambiguous — either the index is unreachable (a predicate written in a form that defeats
partial-index proving, as in Part 17.8), or it is reachable and the planner correctly judged a scan
cheaper. Only that flag can tell them apart.

## 32.5 Methodology and limitations

**Methodology.** A combinatorially generated corpus with a fixed seed (`20260101`). Bulk-loaded in
chunks of 5,000. Server time from `EXPLAIN ANALYZE`; client time measured around the call. Cold
time is the first call after loading. Benchmarks are deselected from the default suite so they do
not measure contention with 300 other tests.

**Limitations, stated plainly:**

- **One machine.** A local Docker Desktop PostgreSQL instance on a development laptop.
- **One query shape.** A selective full-text query with `LIMIT 10`.
- **No concurrent load.** These are single-query measurements, not a throughput test.
- **No vector-search benchmark.** The committed numbers are for the lexical path. HNSW performance
  at 100,000 vectors is not measured here.
- **No write-throughput benchmark.**
- **Generated content**, which is more uniform than real project knowledge.

The README states the boundary precisely, and this is the sentence to reuse:

> "This measures one selective query at three corpus sizes on one machine — it demonstrates
> sub-linear growth for that query shape, not a general scalability claim."

**Do not say "it scales to 100,000 memories."** Say "at 100,000 memories, one selective query's
server-side time grew 1.9× while the corpus grew 100×, which shows cost tracking the answer rather
than the table."

---

# PART 33 — TECH STACK

Every dependency below was read from `pyproject.toml` and confirmed by its use in the code.

## Python 3.12

**What.** The language. `requires-python = ">=3.12"`.

**Why here.** The codebase uses 3.12 features directly: `StrEnum` (`domain/enums.py`), PEP 695
generic syntax (`def domain_errors[**P, R](...)` in `mcp/mapping.py`, `async def run_together[T]`
in the concurrency conftest), `X | Y` union syntax throughout, and `match` statements
(`embeddings/factory.py`).

**Alternative.** Any earlier Python, with more verbose typing.

**Tradeoff.** Requires a recent interpreter. Acceptable for a developer tool.

## MCP SDK (`mcp>=2.1.1`)

**What.** The official Model Context Protocol SDK for Python.

**Why here.** It handles the JSON-RPC framing, the stdio transport, tool and resource registration,
and — importantly — derives each tool's input and output JSON Schema by introspecting the handler's
type annotations.

**Where.** `src/memhub/mcp/server.py` (`MCPServer`, `@server.tool`, `@server.resource`),
`__main__.py` (`run_stdio_async`), and the tests (`mcp.Client`, `StdioServerParameters`).

**A note in `pyproject.toml` worth reading:** *"mcp 2.1.1 verified against PyPI at Milestone 1, not
assumed: v2 renamed FastMCP to MCPServer, so code written from v1 tutorials does not run here."*
That is the right instinct — checking the current API rather than copying a tutorial.

**Tradeoff.** Depending on the SDK's introspection means annotations become load-bearing, which is
why `server.py` avoids `from __future__ import annotations` and `@domain_errors` uses
`functools.wraps`.

## PostgreSQL 16 + pgvector

**What.** The database, and an extension adding vector types and indexes.

**Why here.** Every guarantee this project makes is a PostgreSQL feature: partial unique indexes,
composite foreign keys, `EvalPlanQual` re-check semantics, `FOR UPDATE SKIP LOCKED`, `FOR SHARE`,
GIN + `tsvector`, HNSW, `EXPLAIN ANALYZE`, and server-side `now()`.

**Where.** Everywhere below the service layer. Image: `pgvector/pgvector:pg16` in
`docker-compose.yml`, on host port 5435 to avoid colliding with a locally installed PostgreSQL, and
started with `max_connections=200` for the 50-writer test.

### PostgreSQL vs SQLite

This is the strongest honest alternative, because this is a local tool. SQLite with `FTS5` and
`sqlite-vec` would be *simpler*: zero infrastructure, a single file, no Docker, no connection pool.
If the only goal were "a personal tool that works", SQLite would probably win.

The reasons for PostgreSQL, stated without dressing them up as scale:

1. **Multi-process concurrent writers are SQLite's weakest axis.** Two stdio server processes both
   writing is exactly the case where WAL-mode SQLite serialises writers and surfaces `SQLITE_BUSY`.
   The concurrency story becomes "retry until the lock frees" rather than "the database evaluated
   my predicate atomically and told me I lost".
2. **The correctness mechanisms this project demonstrates are PostgreSQL features** — the list
   above.
3. **It does not foreclose a remote phase.** A shared HTTP server needs a real server database.

**The honest trade:** paying operational complexity to buy a substrate where the interesting
invariants are expressible and testable.

### PostgreSQL vs Redis

Redis is fast and has good primitives for queues. It is the wrong choice here for one decisive
reason: **you cannot have a transaction spanning PostgreSQL and Redis.** The transactional outbox
requires the job row and the memory write to be the same transaction. Putting the queue in Redis
reintroduces the dual-write bug the outbox exists to remove. Redis also offers no durability
guarantee comparable to a committed PostgreSQL transaction, and no referential integrity.

### pgvector vs a separate vector database

Pinecone, Qdrant, Chroma. Three reasons against, in increasing importance:

1. **Nearest-neighbour search over a corpus containing both "Redis is the queue" and "PostgreSQL is
   the queue" returns both**, ranked by similarity, and similarity has no opinion about which is
   true. This is precisely the failure mode this project exists to prevent.
2. **Vector stores have no transactions.** Metadata (status, revision, supersession) and the vector
   would live in two systems, and the only way to guarantee "the vector index never returns a
   retired memory" is a distributed transaction across two stores, or acceptance of a permanent
   inconsistency window.
3. **No compare-and-set, no unique constraints, no referential integrity.**

Keeping the vectors in the same database means the filter and the ANN search commit together and
are read under one snapshot. The trade: pgvector's ANN implementation is less specialised than a
dedicated engine's, and at very large scale a purpose-built vector database would win on raw
retrieval performance.

## SQLAlchemy 2.x (async)

**What.** A Python toolkit for defining database models and building queries.

**Why here.** Typed model definitions that double as the schema definition Alembic diffs against;
composable `Select` objects, which is what makes the stage-0 filter reusable as a *function*
returning a query fragment; and it is fully typed, which matters under `mypy --strict`.

**Where.** `persistence/models.py`, all repositories, `retrieval/filters.py`.

**Note.** The project does **not** use the ORM for everything. Six statements are hand-written SQL
in `persistence/sql/*.sql`, deliberately, because their correctness depends on exact PostgreSQL
semantics that a query builder would hide.

**Tradeoff.** A large dependency, and its async layer has sharp edges (the event-loop scoping issue
documented in `pyproject.toml`'s pytest configuration).

## asyncpg

**What.** A fast, async-native PostgreSQL driver.

**Why here.** The whole persistence layer is async; `config.py` refuses any URL whose driver is not
`postgresql+asyncpg`.

**Where.** Under SQLAlchemy for application code, and used directly in `tests/conftest.py` for
administrative statements (`CREATE DATABASE`, `DROP DATABASE`) which cannot run inside a
transaction block.

**Tradeoff.** It ships no `py.typed` marker, so `pyproject.toml` carries a narrow mypy exemption —
with a comment explaining that application code reaches PostgreSQL through SQLAlchemy, so the
exemption does not weaken checking of `src/`.

## Alembic

**What.** The schema migration tool for SQLAlchemy.

**Why here.** Schema changes need to be versioned, ordered, reviewable, and reversible.

**Where.** `migrations/`, six versions `0001`–`0006`; `persistence/schema.py` reads the migration
head at startup to refuse a mismatched database.

**Notable.** `0001` is deliberately **empty**. Its job was to prove the migration pipeline — that
Alembic is wired to the async engine, that upgrade and downgrade round-trip, that
`alembic_version` is created — before any DDL existed to complicate it.

**Tested.** `test_migrations.py` runs upgrade → downgrade → upgrade, and includes a **drift test**:
`alembic revision --autogenerate` against the migrated schema must produce an empty diff. That
single test catches the whole class of "someone changed the model and forgot the migration".

## fastembed + BAAI/bge-small-en-v1.5

**What.** An ONNX-based embedding runtime, and a small English embedding model producing 384-dim
vectors.

**Why here.** Local, private, reproducible, and free of an external API. ONNX rather than torch: a
~50 MB model instead of a multi-gigabyte install.

**Where.** `embeddings/local.py`, imported lazily inside `_load()`.

**Optional.** `pip install -e ".[local-embeddings]"`. The default adapter is `none`.

**Alternative.** A hosted embedding API — better quality, at the cost of a network dependency, an
API key, and sending every private project memory to a third party.

**Tradeoff.** A small model. `q13` "deadlock prevention" still scores 0.000, and the results
document says 384 dimensions of a small local model do not close that particular gap.

## Pydantic and pydantic-settings

**What.** Runtime data validation and typed settings.

**Why here.** `pydantic-settings` gives typed, validated configuration from the environment, and
`config.py`'s rule is that nothing in the codebase reads `os.environ` directly. Pydantic models in
`mcp/schemas.py` become the tools' declared output schemas.

## Docker and Docker Compose

**What.** Container runtime and a one-file service definition.

**Why here.** So `docker compose up -d --wait` gives you PostgreSQL 16 with pgvector already
installed, at a known port, with the settings the tests need.

**Tradeoff, and it is measured.** Docker Desktop's port forwarding is the entire gap between server
time (0.36–0.67 ms) and client time (5–23 ms) in the benchmarks. The benchmark reports both
separately rather than hiding it.

## pytest, pytest-asyncio

**What.** The test framework, and its async support.

**Why here.** 319 test functions, almost all of them async.

**Notable configuration.** `asyncio_mode = "auto"`, and both loop scopes set to `"session"` — with
a comment explaining that function-scoped loops make the second test in a module reach for a pooled
connection created in an already-closed loop, failing with "Event loop is closed", which is a
confusing symptom of a scope mismatch rather than a bug in the code under test.

Also `filterwarnings = ["error", ...]` — warnings are failures, with exactly one narrowly scoped
exception for Alembic's inability to diff a generated column's expression.

## ruff and mypy

**What.** A linter/formatter, and a static type checker.

**Why here.** `mypy` is in `strict` mode over both `src` and `tests`.

**The load-bearing lint rule:** `T20`, flake8-print, which **bans `print()`**. The `pyproject.toml`
comment says why: *"stdout is the JSON-RPC channel. print() is banned."* `demo.py` is exempted,
because it is a script a human runs in a terminal, not a subprocess whose stdout is parsed as
protocol.

---

# PART 34 — REPOSITORY STRUCTURE AND THE FILES TO READ

## 34.1 The folders

| Folder | Purpose | What belongs here | What must not | Who calls it | What it calls |
|---|---|---|---|---|---|
| `domain/` | pure types and rules | enums, errors, frozen result types, validation, normalisation, per-type policy | any I/O; any import from another memhub package | services, mcp, persistence | nothing internal |
| `services/` | business logic and transactions | orchestration, idempotency, dedup, supersession, audit, metrics | SQL; any awareness of MCP | mcp handlers, CLI, most tests | domain, persistence, retrieval, context |
| `persistence/` | database access | ORM models, repositories, engine, hand-written SQL, schema check | policy decisions; anything unscoped by project | services | domain, retrieval/filters |
| `retrieval/` | reading | the stage-0 filter, lexical, semantic, fusion, ranking | any mutation | repositories, services | domain, persistence models |
| `embeddings/` | the embedding boundary | the port, adapters, the outbox worker | being required for a write to succeed | mcp entry point, services, tests | persistence, observability |
| `context/` | selection under a budget | token estimation, quotas, MMR, greedy fill, rendering | retrieval or SQL | services/context | domain |
| `mcp/` | the protocol boundary | tool and resource registration, output schemas, error mapping, entry point | SQL or business logic; handlers stay thin | the MCP client | services |
| `cli/` | operator commands | purge, gc, status | anything reachable over MCP | a human | persistence, services |
| `observability/` | logs and metrics | JSON logging to stderr, in-process metric registry | writing to stdout; unbounded metric labels | everything | nothing |
| `migrations/` | schema versions | Alembic revisions | application logic | Alembic | — |
| `tests/` | the proof | seven categories | mocked databases | pytest | everything |
| `docs/` | design and evidence | architecture, failure modes, client setup, eval results, perf artifacts | — | humans | — |
| `eval/dataset/` | the graded corpus | memories, queries, baseline, history | — | tests/eval | — |

## 34.2 The twenty files to read, in order

### 1. `src/memhub/retrieval/filters.py` (78 lines)
**Role.** The stage-0 filter. The single definition of "which memories are visible".
**Key function.** `current_revisions(project_id, *, include_retired=False)`.
**Called by.** Every retrieval path.
**Why important.** This is the product. Four conditions, each load-bearing.
**Interview concept.** Structural exclusion over ranking-based suppression.

### 2. `src/memhub/persistence/sql/cas_revise.sql` (56 lines, mostly comment)
**Role.** The compare-and-set statement.
**Why important.** The entire concurrency mechanism is one `UPDATE`. The comment is the correctness
argument.
**Interview concept.** Optimistic concurrency, `EvalPlanQual`, READ COMMITTED chosen for its
conflict semantics.

### 3. `src/memhub/services/memories.py` (604 lines)
**Role.** The write path.
**Key functions.** `remember`, `revise`, `forget`, `history`, `search`, `_apply_supersession`,
`_deduplicate`.
**Why important.** Every ordering decision in the system is here — idempotency before CAS, savepoint
around dedup, supersession in the same transaction.
**Interview concept.** Transaction design and ordering as correctness.

### 4. `src/memhub/persistence/models.py` (563 lines)
**Role.** The schema, as typed models with all constraints and indexes.
**Why important.** Nine of fourteen invariants live here.
**Interview concept.** Enforcing invariants in the schema rather than the application.

### 5. `src/memhub/mcp/server.py` (582 lines)
**Role.** The seven tools, three resources, and the tool descriptions.
**Why important.** Shows how thin the protocol layer is, and that descriptions are production
surface.
**Interview concept.** Tool descriptions as the prompt that steers the model.

### 6. `src/memhub/services/idempotency.py` (218 lines)
**Role.** The claim/wait/replay protocol.
**Key functions.** `claim`, `complete`, `fingerprint`, `purge_expired`.
**Interview concept.** Race-free idempotency; `FOR SHARE` as a wait primitive.

### 7. `src/memhub/persistence/repositories/truth.py` (336 lines)
**Role.** Dedup, attestation, supersession, forget, lineage, the outbox enqueue, coverage.
**Why important.** "Which facts are currently true?" is answered here.

### 8. `src/memhub/context/builder.py` (264 lines)
**Role.** Selection under a token budget.
**Key functions.** `select`, `_fill`, `_stable_order`, `similarity`.
**Interview concept.** Quotas, MMR, greedy knapsack, deterministic output.

### 9. `src/memhub/embeddings/worker.py` (213 lines)
**Role.** The outbox worker.
**Key.** `CLAIM_JOBS` with `FOR UPDATE OF j SKIP LOCKED`, `run_once`, `_defer`, `drain`.
**Interview concept.** PostgreSQL as a job queue; backoff and DEAD.

### 10. `src/memhub/mcp/mapping.py` (182 lines)
**Role.** Turning exceptions into results the model can act on.
**Key function.** `classify_infrastructure_error`, and the `domain_errors` decorator.
**Interview concept.** `UNKNOWN_OUTCOME`, and the branch order that is mutation-tested.

### 11. `src/memhub/retrieval/fusion.py` (92 lines)
**Role.** Reciprocal Rank Fusion.
**Interview concept.** Why rank space rather than score space.

### 12. `src/memhub/retrieval/semantic.py` (130 lines)
**Role.** Vector search.
**Interview concept.** Filtered ANN, iterative scans, and why a distance threshold is required.

### 13. `src/memhub/retrieval/lexical.py` (119 lines)
**Role.** Full-text matching, scoring, and the any-term fallback.
**Interview concept.** `websearch_to_tsquery` vs `to_tsquery`; `ts_rank_cd`; unbounded scores.

### 14. `src/memhub/persistence/repositories/memories.py` (303 lines)
**Role.** Create, get, search, count, `compare_and_set`, `explain_search`.
**Interview concept.** Every method requires a project scope, by signature.

### 15. `src/memhub/services/retrieval.py` (149 lines)
**Role.** Hybrid search: run both legs, fuse, hydrate, report coverage, degrade.
**Interview concept.** Degradation as a first-class outcome.

### 16. `src/memhub/context/tokens.py` (122 lines)
**Role.** The token estimator and the safety margin.
**Interview concept.** Being honest about an unavoidable approximation, with a measured error bound.

### 17. `src/memhub/domain/normalize.py` (100 lines)
**Role.** Content normalisation, hashing, git-remote and workspace-path normalisation.
**Interview concept.** Deduplication keys, and why the normaliser is deliberately conservative.

### 18. `src/memhub/services/projects.py` (194 lines)
**Role.** Project resolution.
**Interview concept.** Never guess, never auto-create; ambiguity as an error.

### 19. `src/memhub/persistence/schema.py` (99 lines)
**Role.** Refuse to start against a schema this build does not expect, in either direction.
**Interview concept.** Failing loudly at startup beats failing quietly at query time.

### 20. `src/memhub/cli/admin.py` (283 lines)
**Role.** `purge`, `gc`, `status`.
**Interview concept.** Irreversible operations do not belong in a model's tool surface.

**And the five tests to read**, because they are where the guarantees are:

```
tests/concurrency/test_compare_and_set.py     the 50-writer proof
tests/integration/test_stale_memory.py        the thesis, executed
tests/integration/test_invariants.py          the schema, proved deployed
tests/protocol/test_stdio_transport.py        two processes, one database
tests/integration/test_embedding_outbox.py    the outbox and SKIP LOCKED
```

---

# PART 35 — RECONSTRUCTING THE PROJECT PHASE BY PHASE

## 35.0 How this was reconstructed

Not from the README's milestone table alone. From `git log` (11 substantive commits, each with a
detailed message and a file list), cross-checked against the six migrations, the test directories,
and the docs each commit added.

Git history and the README agree, which is itself worth noting.

**Diagram 25 — phase evolution**

```
  0  skeleton         docker, alembic 0001 (empty), logging, harness, CI
  |
  1  thin slice       0002: projects, memories, revisions.  3 tools over stdio.
  |                   -> proves MCP works before building on assumptions
  |
  2  correctness      0003: idempotency_keys, audit_events. CAS. metrics.
  |                   -> 50-way concurrency and idempotency proofs
  |
  3  truth            0004: dedup keys, attestations. supersedes, forget, history.
  |                   -> the thesis: stale memory leaves retrieval
  |
  4  clients          docs/clients.md, resources, golden manifest snapshot
  |
  5  lexical          0005: content_tsv, GIN. ts_rank_cd + priors.
  |
  6  EVALUATION       200 memories, 34 graded queries, committed baseline
  |                   -> the ruler is built BEFORE the thing it measures
  |
  7  semantic         0006: pgvector, HNSW, embedding outbox. RRF.
  |                   -> measured against the baseline from phase 6
  |
  8  context          quotas, MMR, knapsack, token estimator
  |
  9  failure + perf   error classification, operator CLI, schema check,
                      1k/10k/100k benchmarks
```

---

## PHASE 0 — SKELETON
*Commit: "Set up project skeleton, database and test harness"*

**What existed before.** Nothing.

**Problem.** You cannot build anything until you can run PostgreSQL reproducibly, migrate it, test
against it, and see what the server is doing.

**Why this was the next problem.** Everything else depends on it.

**What was added.** Docker Compose with `pgvector/pgvector:pg16` on port 5435; typed settings with
validation; JSON logging to **stderr**; async Alembic with an intentionally **empty** baseline
migration; a template-database test harness; CI.

**Simple explanation.** Build the workbench before the furniture.

**New concepts.** Connection pool, bounded pool timeout, server-side statement timeout, structured
logging, template-database test isolation, constraint naming conventions for drift detection.

**Important files.** `docker-compose.yml`, `config.py`, `observability/logging.py`,
`persistence/engine.py`, `migrations/env.py`, `tests/conftest.py`.

**Schema changes.** `0001_baseline.py` — deliberately empty. Its job was to prove the pipeline
before any DDL existed to complicate it.

**Tests added.** 31, including one that asserts nothing reaches stdout, and a barrier-gated pool
test that pre-empts the concurrency work to come.

**Why before the next phase.** The pgvector image was chosen on day one so that Phase 7 would need
no infrastructure change — but the extension was deliberately *not* created until then.

**Interview takeaway.** The empty migration is the detail worth mentioning. It proves the pipeline
in isolation from the modelling.

---

## PHASE 1 — END-TO-END THIN SLICE
*Commit: "Add project and memory persistence with MCP tools"*

**What existed before.** Infrastructure only.

**Problem.** The MCP layer holds the most unknowns — it is the part you do not control.

**Why this was the next problem.** The architecture document is explicit: validate the part you do
not control in week one, before building three phases of backend on assumptions about it.

**What was added.** `projects`, `project_aliases`, `memories`, `memory_revisions`; the domain layer;
three tools (`project_use`, `memory_remember`, `memory_search`); the stdio entry point; the stage-0
filter as a function from the very beginning.

**New concepts.** Immutable revision log behind a mutable lifecycle row; project identity vs
resolution aliases; content hashing from day one (because `content_hash` is `NOT NULL` on an
append-only table, adding it later would need a backfill).

**Schema changes.** `0002` — four tables, the composite foreign keys, and the partial unique index
`uq_memory_revisions_memory_id`.

**Tests added.** Up to 148 passing, including the invariant suite and the real-subprocess stdio
test.

**Exit criterion, met.** A real MCP client writes a memory and reads it back in a new session.

**Interview takeaway.** De-risk the unknown interface first. Also: the stage-0 filter existed before
there was any ranking to protect, which is why it is one function everything else composes on.

**Between phases 1 and 2** there is a documentation-only commit: *"Document why files and git are
not sufficient"*, prompted by reviewing an existing public MCP memory server built on markdown files
in a git repository. It is worth reading, because it is the cheapest alternative and the one most
likely to be raised in an interview. The answer: git solves *distribution* while leaving both of
this project's actual problems untouched — it has the wrong conflict model (a three-way text merge
is a syntactic operation and "which of these two statements is true?" is a semantic question), and
it cannot filter at read time (a retired line stays in the file until a human deletes it).

---

## PHASE 2 — CORRECTNESS CORE
*Commit: "Add compare-and-set updates and idempotent writes"*

**What existed before.** Memories could be created and read, but not changed.

**Problem.** Two clients revising the same memory. And: a client retrying after a dropped
connection.

**Why now.** Idempotency changes the signature of every write. Retrofitting it means rewriting the
write API and every test that calls it.

**What was added.** `memory_revise` with `expected_revision`; the CAS SQL; the idempotency claim
protocol with `FOR SHARE`; the audit log; the metrics registry.

**New concepts.** Lost update, optimistic concurrency, CAS, READ COMMITTED, `EvalPlanQual`, `FOR
SHARE`, request fingerprints, bounded-cardinality metric labels.

**Schema changes.** `0003` — `idempotency_keys`, `audit_events`.

**Tests added.** 184 total; the 50-way concurrency and 50-way idempotency suites, plus the
connection-pool fixture that makes them honest.

**Interview takeaway.** Observability arrived here, not at the end, because you cannot debug a
50-way concurrency test without it.

---

## PHASE 3 — TRUTH MAINTENANCE
*Commit: "Add deduplication, supersession and memory history"*

**What existed before.** Facts could be created and revised, but never retired.

**Problem.** The thesis of the project: a decision gets replaced, and the old one must stop being
returned while remaining auditable.

**Why now.** The architecture document is blunt: *"Stale-memory suppression is the thesis of the
project. It cannot be a late add-on."* The original specification had it at Phase 8; it was pulled
forward to 3.

**What was added.** `supersedes` folded into `memory_remember`; `memory_forget` as a tombstone;
`memory_history`; deduplication keyed on content hash; attestations.

**New concepts.** Revision vs supersession as distinct relations with different cardinalities;
dedup vs idempotency; tombstoning; dedup-key release on retirement.

**Schema changes.** `0004` — `memory_dedup_keys`, `memory_attestations`.

**Tests added.** 201 total, including `test_stale_memory.py` with the every-limit loop.

**Interview takeaway.** Supersession is an argument to `remember`, not its own tool, because
retiring the old fact and asserting the new one must be one transaction.

---

## PHASE 4 — CLIENT INTEGRATION
*Commit: "Add client integration docs, resources and a manifest snapshot"*

**Problem.** Configuration is where an MCP server silently fails to appear.

**What was added.** `docs/clients.md`, verified against current official documentation rather than
copied from tutorials — recording the absolute-path requirement, the schema difference between
Claude Desktop and Cursor, and where each writes its logs. Three read-only resources. The golden
manifest snapshot.

**Interview takeaway.** Tool descriptions are production surface, so they belong under snapshot
test. A wording change alters model behaviour with no logic change and no other test failing.

---

## PHASE 5 — LEXICAL RETRIEVAL
*Commit: "Add full-text retrieval with relevance ranking"*

**Problem.** Search was exact/substring matching. Real questions do not work that way.

**What was added.** `content_tsv` as a generated column, partial GIN indexes, `ts_rank_cd` scoring,
and the three multiplicative ranking priors.

**Schema changes.** `0005`.

**Two findings recorded where they matter**, and both are in the commit message:

1. **The `IS TRUE` trap.** A currency predicate written as `is_current IS TRUE` makes the partial
   index unusable at any corpus size. Guarded by a test on the compiled SQL.
2. **The stemmer gap.** The English stemmer misses `JWTs` → `jwt`. Pinned as a failing case that
   motivates semantic retrieval.

**Interview takeaway.** Finding #2 is why Phase 7 exists, and it was recorded as a failing case
*before* the fix was built.

---

## PHASE 6 — EVALUATION HARNESS
*Commit: "Add retrieval evaluation harness with a committed baseline"*

**This is the most important sequencing decision in the project.**

**Problem.** You are about to add semantic search and claim it is better. Better than what,
measured how?

**Why the harness came BEFORE vectors.** The architecture document states the reasoning: *"If you
build hybrid retrieval and then build the measurement, you will unconsciously fit the metric to the
conclusion. Build the ruler first, then prove hybrid beats FTS."*

The judgments were written before any strategy was measured against them, so there was no
opportunity to grade whatever the system happened to return.

**What was added.** A 200-memory corpus, 34 graded queries with `forbidden` lists, nDCG / recall /
precision / MRR / stale inclusion, and a committed baseline with a CI regression gate.

**And the harness found a real defect on its first run.** PostgreSQL joins bare query terms with
AND, so natural-language questions returned nothing at all — "migration rules" and "connection pool
size" scored zero. The any-term fallback took nDCG from **0.478 to 0.802** and recall from **0.468
to 0.817**, while stale inclusion stayed at exactly zero.

**Interview takeaway.** This is the strongest single answer to "how do you know it works". Build the
ruler before the thing you want to measure — and be able to point at a defect the ruler found.

---

## PHASE 7 — SEMANTIC AND HYBRID
*Commit: "Add vector search and rank fusion over an embedding outbox"*

**What existed before.** Full text, measured, with its weaknesses recorded.

**Problem.** Queries whose wording does not match the memory's wording. `q22` "jwt" scored 0.000.

**What was added.** `CREATE EXTENSION vector`; `memory_embeddings` with an HNSW index;
`embedding_jobs` as a transactional outbox; the `EmbeddingPort` protocol with a local adapter and a
deterministic fake; RRF; iterative index scans; the distance threshold.

**Schema changes.** `0006`.

**Measured against the committed baseline.** nDCG 0.803 → 0.853, recall 0.817 → 0.828, stale
inclusion unchanged at zero.

**And the finding that makes the phase interesting.** Without a distance threshold, nDCG reached
0.881 while precision collapsed to 0.113. The threshold was chosen by sweeping against the corpus,
and the sweep is committed.

**Interview takeaway.** "I measured, and the first result was misleading in a way that looked like
success" is a much better story than "I added pgvector".

---

## PHASE 8 — CONTEXT BUILDER
*Commit: "Add budgeted context assembly"*

**Problem.** Search answers "what matches". A session start needs "given this much room, what is
worth knowing".

**Why after retrieval.** It selects *from* retrieval's output. It could not be built first.

**What was added.** `memory_context`; quotas with redistribution; MMR diversity; greedy knapsack
fill; the token estimator; deterministic ordering; the budget report.

**Schema changes.** None. This phase is pure logic on top of what already existed.

**And the calibration correction.** `CHARS_PER_TOKEN` started at 3.6 and under-estimated 2 of 33
real memories, because technical writing full of identifiers tokenises far more densely than prose.
Corrected to 3.2; none under-count and the worst case is 4.8% over.

**Interview takeaway.** Suppression of retired memories was verified at *every* budget, not just at
one — the same shape of assertion as the every-limit loop in Phase 3.

---

## PHASE 9 — FAILURE HANDLING AND SCALE
*Commit: "Add failure handling, operator commands and scaling benchmarks"*

**Problem.** The architecture had a failure matrix. A matrix is a claim until something checks it.

**What was added.** The four infrastructure error codes at the MCP boundary; degradation to
lexical-only when the semantic leg is cancelled, not only when the embedder fails; the operator CLI
(`purge`, `gc`, `status`), deliberately not over MCP; the startup schema check refusing in both
directions; benchmarks at 1k, 10k, and 100k.

**And what writing `docs/failure-modes.md` found.** Three of the error codes the architecture
promised — `BACKEND_UNAVAILABLE`, `BACKEND_BUSY`, `UNKNOWN_OUTCOME` — **did not exist anywhere in
the code**. And the deadline row claimed a degradation path that only triggered when the *embedder*
failed, not when a query was cancelled, which is the likelier cause.

**Interview takeaway.** This is the best answer to "what would you do differently". The prose had
been true when written and had quietly stopped being true, and nothing else would have noticed.
Mapping every claim to the test that proves it is what surfaced the gap.

---

## 35.10 The sequencing lessons, extracted

1. **De-risk the interface you do not control, first.** MCP in Phase 1, not Phase 4.
2. **Anything that changes every write signature goes early.** Idempotency in Phase 2.
3. **The thesis cannot be a late add-on.** Supersession moved from Phase 8 to Phase 3.
4. **Build the ruler before the thing you measure.** Evaluation in Phase 6, vectors in Phase 7.
5. **Observability is a debugging tool, not a garnish.** Phase 2, not Phase 9.
6. **Testing is not a phase.** Every milestone shipped its own tests.
7. **Write the document that maps claims to evidence.** It is what finds the claims that quietly
   stopped being true.

---

# PART 36 — CURRENT LIMITATIONS

Every item below was verified by searching the implementation, not taken from the README.

## 36.1 No authentication or authorization

**What is missing.** There is no user model, no credential check, no per-caller permissions.
Verified: no auth code anywhere in `src/memhub/`. Any process that can reach the configured
`MEMHUB_DATABASE_URL` can read and write any project.

**Why out of scope.** V1 is single-user and local. The clients are on the same machine as the
database. Attempting an identity and permission model in a local stdio V1 would be ceremony around
a boundary that does not exist yet.

**When needed.** The moment a shared server serves multiple untrusted callers — that is, the moment
the HTTP transport exists.

**How it could be added.** It arrives with that transport, not before: a caller identity carried in
the transport, an authorization check at the MCP boundary, and PostgreSQL row-level security or an
explicit tenant column on every table. It is design work, not a flag.

## 36.2 No metrics exporter

**What is missing.** `src/memhub/observability/metrics.py` is an in-process registry. Nothing
scrapes it and nothing receives it. Verified: no OTLP exporter, no Prometheus endpoint, no push
code.

**Why out of scope.** The architecture explains the real obstacle: the server is a **short-lived
subprocess**, so pull-based Prometheus scraping cannot find it, and a metrics endpoint on a random
port is a poor fit. The intended answer is OTLP push to a local collector, which was scoped and not
built.

**When needed.** Any deployment where someone is on call, or where you want to see conflict rates
and embedding backlog over time.

**How it could be added.** An OTLP push exporter behind the existing registry. The label discipline
is already correct, which is the part that is expensive to retrofit.

## 36.3 No distributed tracing

**What is missing.** `docs/architecture.md` §11.3 specifies a root span per tool call with child
spans for `db.transaction`, `db.cas_update`, `retrieval.fts`, and so on. Verified: there is no
tracing module and no OpenTelemetry dependency.

**How it could be added.** OpenTelemetry spans in the `domain_errors` decorator and around the
repository calls. The `request_id` propagation the design calls for is partially there —
`audit_events.request_id` exists as a column, though the MCP handlers do not currently populate it.

## 36.4 stdio only; no Streamable HTTP

**What is missing.** One transport. Each client gets its own process.

**Why out of scope.** stdio is what Claude Desktop and Cursor use for local servers, and it is what
makes the concurrency work real.

**When needed.** A long-lived server handling many concurrent client connections; a hosted
deployment; sharing memories between people.

**How it could be added.** The architecture is explicit that the service layer does not change —
`memhub.mcp` gains a second entry point. But the *phase* is real work, because it brings
authentication, per-caller authorization, and rate limiting with it.

## 36.5 Retention is manual

**What is missing.** Nothing schedules `memhub-admin gc`. Verified: no cron, no scheduler, no
startup sweep. `docs/architecture.md` §6.2 says a bounded delete "runs at startup and hourly" — it
does not.

**What `gc` does when invoked.** Deletes expired idempotency keys and embedding jobs that have been
`DEAD` for more than 30 days, both in bounded batches. It never removes a memory, which is asserted
by `test_garbage_collection_never_removes_a_memory`.

**When needed.** Any long-running deployment. Idempotency keys accumulate at one row per keyed
write.

**How it could be added.** An external cron calling `memhub-admin gc`, or a periodic asyncio task
alongside the embedding worker.

## 36.6 No secret screening

**What is missing.** `docs/architecture.md` §10.2 specifies write-time screening — pattern rules for
AWS keys, GitHub tokens, private-key headers, JWT shapes, `KEY=value` lines, plus a Shannon-entropy
heuristic — with a default action of **reject**, and an `acknowledge_sensitive` override recorded in
the audit log.

**None of that exists.** Verified by searching for `entropy`, `acknowledge_sensitive`,
`secret_rejection`, and screening logic: no matches in `src/memhub/`.

**What does exist.** The tool descriptions instruct the model never to record credentials, and
`test_descriptions_carry_the_operating_contract` asserts that instruction is present. The
`memory_forget` description points at the operator purge for the case where one was recorded
anyway, and `memhub-admin purge` genuinely erases content from every derived table.

**This is the largest single gap between the architecture document and the implementation.** A
prompt instruction is not a control. Say so plainly if asked.

**How it could be added.** A pure function in `domain/` applied in `validate_content`, plus the
`acknowledge_sensitive` argument and an audit outcome for it. The `memhub_secret_rejections_total`
metric name is already reserved in the architecture.

## 36.7 Search does not report its own coverage

**What is missing.** `SearchResult` carries `match_strategy`, `semantic_coverage`, and `degraded`.
`SearchOut` — the MCP output schema — carries none of them. So a client calling `memory_search` is
not told whether the query was widened, how much of the corpus was vector-indexed, or whether the
semantic leg failed. `memory_context` does report the last two.

**Why it matters.** The architecture's sketched signature for `memory_search` includes
`semantic_coverage` and `degraded` precisely so that "search is never silently partial". Through
MCP, today, it can be.

**How it could be added.** Three fields on `SearchOut`, and a manifest snapshot regeneration.

## 36.8 Ranking priors are not applied to fused results

**What is missing.** `docs/architecture.md` §7.4 describes RRF followed by multiplicative priors,
including an attestation prior, plus a `why` block per result showing component ranks. The
implementation applies priors inside the lexical leg only; there is no attestation prior and no
`why` block.

**Why it matters less than it sounds.** The lexical ranking feeding fusion is already
prior-adjusted. But the semantic leg's contribution is not, and a caller cannot see why a result
ranked where it did.

## 36.9 Two tool descriptions are stale

Verified against `tests/protocol/manifest.json`:

- `memory_search`: *"The optional query is a substring match today; relevance ranking arrives in a
  later version."* Wrong since Phase 5.
- `memory_revise`: *"If the fact has been replaced by a different one, that is supersession, not
  revision - support for it arrives in a later version."* Wrong since Phase 3.

**Why this matters more than an ordinary stale comment.** Tool descriptions are the prompt that
steers the model. A model reading the second sentence may conclude supersession is unavailable and
record a contradicting fact instead — which is exactly the behaviour the `memory_remember`
description warns against.

**How it could be fixed.** Edit the descriptions, regenerate the manifest with
`MEMHUB_UPDATE_MANIFEST=1`, review the diff.

## 36.10 Ranking weights are untuned

`W_IMPORTANCE = 0.5` and `W_RECENCY = 0.3` were set before the evaluation harness existed and were
deliberately not fitted afterwards. The module docstring says so and explains that tuning before
measuring would mean fitting numbers to intuition. They are named constants so tuning later is a
single-file change — but it has not been done.

## 36.11 No high availability, backups, or replication

One PostgreSQL container with a named Docker volume. No replica, no backup schedule, no
point-in-time recovery. Appropriate for a local developer tool, and worth naming rather than
implying otherwise.

## 36.12 Semantic near-duplicate surfacing is not implemented

`docs/architecture.md` §7.5 describes the `remember` response including a `similar: [...]` list so
the model can decide whether to revise or supersede instead. Verified: `RememberOut` has no
`similar` field. The design decision it embodies — surface evidence, let the model decide, never
merge automatically on a similarity score — still holds; the surfacing was not built.

## 36.13 Honest summary

The implementation is ahead of the documentation in some places (the failure model, the any-term
fallback, the distance threshold) and behind it in others (secret screening, tracing, the metrics
exporter, the `why` block, semantic near-duplicate surfacing, retention scheduling). The README's
"Not built" section names three of these. This part names all of them.

---

# PART 37 — VERIFYING YOUR FOUR RESUME BULLETS

Each bullet is broken into its claims, each claim is marked, and each has evidence.

**Verdict key:** ✅ accurate · ⚠️ partially accurate or ambiguous · ❌ overstated

---

## Bullet 1

> "Built an MCP-based shared memory service for Claude Desktop and Cursor, enabling persistent
> project context across sessions with project isolation, provenance tracking, immutable revision
> history, and memory supersession."

**Overall verdict: accurate.**

| Phrase | Simple meaning | Verdict | Evidence |
|---|---|---|---|
| MCP-based | speaks the Model Context Protocol | ✅ | `mcp>=2.1.1`; `MCPServer` in `mcp/server.py`; 7 tools + 3 resources; `tests/protocol/manifest.json` |
| shared memory service | several clients read and write one store | ✅ | `test_two_sessions_share_state_over_stdio`; `test_a_second_client_sees_the_first_clients_memory` |
| for Claude Desktop and Cursor | configured and verified for both | ✅ | `docs/clients.md`, verified against current official docs; `docs/perf` and tests use both client names |
| persistent project context across sessions | survives process exit | ✅ | the stdio test starts a second process after the first has fully exited |
| project isolation | project A's memories never appear in project B | ✅ | required `project_id` at every layer; composite FKs; `test_cross_project_supersession_is_unrepresentable`; `test_projects_are_isolated_across_the_protocol` |
| provenance tracking | who recorded it, and how | ✅ | `author_client`, `author_kind`, `source`, `created_at` per revision; `memory_attestations`; `audit_events` |
| immutable revision history | old versions are kept, never overwritten | ✅ | `memory_revisions` PK `(memory_id, revision_no)`; `test_history_is_preserved_not_overwritten` |
| memory supersession | one memory retires another, atomically | ✅ | `cas_supersede.sql`; `memories.superseded_by_id`; `test_the_retired_fact_leaves_retrieval_entirely` |

**One nuance to be ready for.** "For Claude Desktop and Cursor" is accurate as *configured and
documented*. If asked "did you run it in both?", the honest answer is that the repository contains
verified configuration for both and an automated test that drives the real stdio transport with two
independent sessions; I could not verify from the repository that a manual two-client session was
recorded.

---

## Bullet 2

> "Added safe concurrent updates, versioned memory, and idempotent writes using PostgreSQL
> transactions and database constraints to prevent stale clients from overwriting newer information
> and avoid duplicate memory records."

**Overall verdict: accurate.**

| Phrase | Simple meaning | Verdict | Evidence |
|---|---|---|---|
| safe concurrent updates | two writers cannot silently overwrite each other | ✅ | `cas_revise.sql`; `test_exactly_one_of_fifty_writers_wins` (1 winner, 49 conflicts) |
| versioned memory | every change is a numbered version | ✅ | `current_revision_no`; append-only `memory_revisions`; `test_a_second_round_advances_by_exactly_one` |
| idempotent writes | a retry does not write twice | ✅ | `idempotency_keys`; `claim_idempotency.sql` + `wait_idempotency.sql`; `test_fifty_concurrent_retries_create_one_memory` |
| PostgreSQL transactions | one transaction per operation | ✅ | `session_scope`; `test_a_failed_write_leaves_nothing_behind` |
| database constraints | rules enforced by the schema | ✅ | 9 of 14 invariants; `tests/integration/test_invariants.py` writes raw SQL to prove each |
| prevent stale clients from overwriting newer information | the lost-update guarantee | ✅ | the CAS predicate; mutation-tested |
| avoid duplicate memory records | no duplicates from retries or from two clients | ✅ | idempotency for retries, `memory_dedup_keys` for cross-client duplicates — **two mechanisms, and you should name both** |

**The nuance that makes this bullet strong in an interview.** The phrase "avoid duplicate memory
records" covers **two different mechanisms**. If you say only "idempotency", you have described
half of it. Say: "Two mechanisms — an idempotency key for one client retrying the same request, and
a content hash under a unique constraint for two different clients asserting the same fact. They
solve different problems and return different outcomes."

---

## Bullet 3

> "Built hybrid keyword and semantic search using PostgreSQL full-text search and pgvector, with
> context filtering to exclude superseded, deleted, and expired memory and return only relevant
> project information."

**Overall verdict: accurate, with one wording note.**

| Phrase | Simple meaning | Verdict | Evidence |
|---|---|---|---|
| hybrid keyword and semantic search | both, combined | ✅ | `services/retrieval.py::hybrid_search`; `retrieval/fusion.py` |
| PostgreSQL full-text search | tsvector, GIN, ts_rank_cd | ✅ | `content_tsv` generated column; partial GIN index; `retrieval/lexical.py` |
| pgvector | vector storage and ANN search | ✅ | `vector(384)`; HNSW with `vector_cosine_ops`; `retrieval/semantic.py` |
| context filtering | excluding invalid memories | ⚠️ wording | The mechanism is real and strong, but "filtering" undersells it and "context" is ambiguous — `memory_context` is a *different* feature. Prefer **"structural exclusion before ranking"**. |
| exclude superseded, deleted, and expired | all three | ✅ | `filters.py::current_revisions`: `status='ACTIVE'` covers superseded and deleted; `expires_at IS NULL OR expires_at > now()` covers expired |
| return only relevant project information | scoped and current | ✅ | `stale_inclusion_rate = 0.000` across all strategies; every retrieval path is project-scoped |

**The improvement worth making.** Replace "with context filtering to exclude..." with **"with a
structural filter that excludes superseded, deleted, and expired memories before any ranking
runs"**. Two reasons: it removes the collision with `memory_context`, and it states the property
that actually distinguishes the design — exclusion happens *before* scoring, so no ranking function
can resurrect a retired memory.

**A number you can attach.** "Measured at nDCG@10 0.853 with hybrid retrieval on a 200-memory graded
corpus, and a stale-memory return rate of exactly zero across every strategy and every budget."

---

## Bullet 4

> "Added background embedding generation, retry and failure handling, token-budgeted context
> retrieval, database migrations, and automated tests covering MCP tools, concurrency, retrieval,
> and persistence workflows."

**Overall verdict: accurate.**

| Phrase | Simple meaning | Verdict | Evidence |
|---|---|---|---|
| background embedding generation | vectors are produced outside the write path | ✅ | `embedding_jobs` outbox; `EmbeddingWorker`; `_drain_forever` as an asyncio task |
| retry | failed jobs are retried with backoff | ✅ | `MAX_ATTEMPTS = 5`, exponential from 2s, then `DEAD`; `test_failure_backs_off_rather_than_spinning` |
| failure handling | driver failures classified into actionable codes | ✅ | four codes in `mcp/mapping.py`; `docs/failure-modes.md`; degradation to lexical-only |
| token-budgeted context retrieval | a brief that fits a token budget | ✅ | `memory_context`; quotas, MMR, greedy knapsack; `docs/eval/tokens.md` |
| database migrations | versioned schema changes | ✅ | six Alembic migrations; a drift test; a startup schema check refusing in both directions |
| automated tests covering MCP tools | protocol-level tests | ✅ | 36 tests, including a real subprocess and a golden manifest |
| ... concurrency | real concurrent writers | ✅ | 12 tests, 50 writers, distinct backends asserted |
| ... retrieval | search quality | ✅ | lexical, hybrid, and a graded evaluation with a CI gate |
| ... persistence | database behaviour | ✅ | 109 integration tests against real PostgreSQL |

**One thing to be careful about.** "Retry and failure handling" is accurate but easy to overstate in
conversation. There is **no automatic client-side retry** in this system. What exists is: retry with
backoff *for embedding jobs*, and *classification* of database failures into codes that tell the
caller whether retrying is safe. Do not say "the system retries failed writes" — it does not, and
`UNKNOWN_OUTCOME` exists precisely because a blind retry would be unsafe.

---

## The one-line summary of the four bullets

**All four are accurate.** The only changes worth making are wording improvements, not corrections:

1. Bullet 3: "context filtering" → "a structural filter applied before ranking".
2. Bullet 2: when speaking, name *both* duplicate-avoidance mechanisms.
3. Bullet 4: when speaking, be precise that retry applies to embedding jobs, and that database
   failures are *classified* rather than automatically retried.
4. Any bullet: attach a measured number. The strongest are "1 winner and 49 conflicts,
   deterministically", "nDCG 0.803 → 0.853", and "stale-memory return rate exactly zero".

## Interview questions each bullet invites, with 20-second answers

**Bullet 1 → "What does MCP actually do here?"**
"It is the interface. It lets Claude Desktop and Cursor discover and call seven tools over stdio.
It carries the tool arguments in and the results out. Everything that makes the system correct —
versioning, conflict handling, retirement, retrieval — is underneath it, in the service layer and
in PostgreSQL. MCP does not store anything and never sees a conversation."

**Bullet 1 → "How is 'shared' actually achieved if the clients never talk?"**
"Each client spawns its own copy of the server as a subprocess. The two processes share no memory
and no cache. PostgreSQL is the only thing both touch, so it is the shared state — and that is
exactly why the concurrency control is a real requirement rather than decoration."

**Bullet 2 → "What is a stale client and how do you stop it?"**
"A client acting on a revision that has since changed. Every update must state the revision it
read. The database checks that condition and performs the write in one statement, so if the
revision has moved on, zero rows change and the writer is told it lost, with the winning revision
attached. That is compare-and-set. Fifty concurrent writers give exactly one winner and 49
conflicts, deterministically."

**Bullet 3 → "Why not just use a vector database?"**
"Because similarity has no opinion about what is currently true. A vector search over a corpus
containing both 'the queue runs on Redis' and 'the queue runs on PostgreSQL' returns both, and the
retired one usually matches better. Keeping the vectors in PostgreSQL means the lifecycle filter
and the vector scan are the same query, so 'a superseded memory is never returned' is one guarantee
rather than two systems that have to agree."

**Bullet 4 → "Why is embedding in the background?"**
"Because a failing embedding model must never fail a write. Inline, one slow inference call holds a
row lock and blocks every other writer, and a model outage becomes a write outage. Doing it after
the commit instead is the dual-write bug — crash in between and the memory exists with no vector and
nothing knows. So the job row is inserted in the same transaction as the memory: either both exist
or neither does. A worker then claims jobs with FOR UPDATE SKIP LOCKED."

---

# PART 38 — INTERVIEW EXPLANATIONS AT FIVE LENGTHS

Every version starts with the **problem**, never with a technology name. If the first thing out of
your mouth is "PostgreSQL and pgvector", you have described a stack rather than a system.

## 38.1 The 15-second version

> "If you settle a design decision while talking to one AI coding tool, a different tool has no way
> to know about it — it never saw that conversation. I built a shared, versioned memory store that
> several AI clients can read and write through one protocol, where two clients writing at once are
> resolved safely and a decision that gets replaced stops being returned as if it were still true."

## 38.2 The 30-second version

> "Developers use more than one AI coding tool, and those tools do not share what you told them.
> The obvious fix is one shared place to record project knowledge — but sharing it correctly is the
> hard part, because now two clients can write to the same fact at the same time, and knowledge
> goes out of date.
>
> So I built a PostgreSQL-backed memory service that several MCP clients reach over one protocol.
> Updates use compare-and-set, so a client working from an out-of-date version is refused and told
> what beat it, rather than silently overwriting. When a decision is replaced, the old one is
> retired in the same transaction as the new one, and it is excluded from retrieval *before* any
> ranking runs — so no search, keyword or semantic, can bring it back. It stays fully readable in
> the audit history."

## 38.3 The 60-second version

> "Say you tell one AI tool on Monday that the job queue runs on Redis. On Tuesday you open a
> different tool. It never saw that conversation, so it starts from nothing. The fix sounds simple:
> one shared place to write project knowledge down.
>
> Giving one assistant a memory is simple — a text file. Sharing it across several clients is where
> it gets hard, for three reasons. Two clients can update the same fact at the same time. A decision
> can be reversed six months later, and the reversal has to actually take effect. And if the old
> and the new version are both left retrievable, a search returns the wrong one — because the
> retired phrasing usually matches the query *better* than the current answer does.
>
> So this is a PostgreSQL-backed store with a versioned memory model. Each client spawns its own
> copy of the server, so the two processes share nothing but the database — which is what makes the
> concurrency work a requirement rather than decoration. Writes are compare-and-set: you say which
> revision you read, and if it has moved on you get a conflict carrying the winning version, so you
> can merge and retry in one round trip. Retiring a fact and asserting its replacement happen in one
> transaction, and retired memories are removed from the candidate set *before* ranking, so
> retrieval cannot resurface them at any result limit or token budget.
>
> Retrieval is hybrid — PostgreSQL full-text plus pgvector — combined by rank fusion rather than by
> adding incomparable scores. I built the evaluation harness before the semantic half, so the
> improvement is a measured number rather than a claim: nDCG went from 0.803 to 0.853, and a
> retired memory reached a caller zero times across every strategy."

## 38.4 The 2-minute version

Start with everything in the 60-second version, then add these four points:

> "**On concurrency, specifically.** The update is a single SQL statement: increment the revision
> where the memory id matches, the project matches, the current revision equals what the caller
> read, and the status is still ACTIVE. Zero rows updated means someone else got there first. The
> reason that is airtight is a PostgreSQL behaviour under READ COMMITTED called EvalPlanQual: when a
> blocked update wakes up, the engine walks to the newest committed version of the row and
> re-evaluates the WHERE clause against it. So the read and the write are the same statement and
> there is no window to lose in. I chose READ COMMITTED deliberately over SERIALIZABLE — SERIALIZABLE
> is also correct, but it turns a clean, informative refusal into a retry loop, and with fifty
> writers you get forty-nine retry storms instead of forty-nine clean conflicts. There is a test
> that launches fifty writers from a barrier so they collide for real, and a fixture that first
> asserts the connection pool can supply fifty distinct database backends — otherwise a fifty-way
> test against a ten-connection pool just measures five sequential waves and passes for the wrong
> reason.
>
> **On duplicates.** Two separate mechanisms, because they are two separate problems. Idempotency is
> one client retrying the same request after a dropped connection, keyed on a caller-supplied
> request id; the retry replays the stored response. Deduplication is two different clients
> independently asserting the same fact, keyed on a normalised content hash under a unique
> constraint; that returns the existing memory and records the second assertion as corroborating
> evidence rather than a duplicate.
>
> **On embeddings.** They are generated in the background through a transactional outbox — the job
> row is inserted in the same transaction as the memory, so either both exist or neither does. A
> worker claims jobs with FOR UPDATE SKIP LOCKED, which lets workers in both client processes drain
> the same queue without contending or double-processing. That means a failing embedding model
> degrades semantic search instead of blocking writes, and every search reports what fraction of the
> corpus is actually vector-indexed.
>
> **On failure.** Driver failures are classified into codes that say what to do next, and the
> interesting one is UNKNOWN_OUTCOME: the connection died mid-flight, so the write may or may not
> have committed, and the acknowledgement that would have said which is exactly what was lost. So
> the system does not claim the write failed — it names the two ways to find out: replay the
> idempotency key, or re-read. That branch is mutation-tested, because reporting a mid-flight
> disconnect as safely retryable is how you get duplicate writes."

## 38.5 The 5-minute deep version

Use the 2-minute version, then take questions in whatever order the interviewer wants. These are
the five threads worth having ready:

**Thread 1 — Why this is not just a database table.**
"A table gets you storage. It does not get you: a fact being *retired by* another fact,
compare-and-set on a logical unit, provenance per version, a read path that answers 'give me the
most useful 2000 tokens', or a machine-callable protocol so the agent can participate. Every
property that makes this interesting is absent from a notes table."

**Thread 2 — Why this is not just a vector database.**
"Three things. Nearest-neighbour search over a corpus that contains both 'the queue runs on Redis'
and 'the queue runs on PostgreSQL' returns both, and similarity has no opinion about which is true —
which is precisely the failure mode this exists to prevent. Vector stores have no transactions, so
the lifecycle metadata and the vector would live in two systems, and the only way to guarantee the
index never returns a retired memory would be a distributed transaction or a permanent inconsistency
window. And no compare-and-set, no unique constraints, no referential integrity. Keeping the vectors
in the same database means the filter and the ANN search commit together and are read under one
snapshot."

**Thread 3 — Why the retrieval evaluation came before the vectors.**
"If you build hybrid retrieval and *then* build the measurement, you will unconsciously fit the
metric to the conclusion. So the harness came first: 200 memories, 34 graded queries, judgments
written before any strategy ran. That ordering paid off twice. It found a real defect on its first
run — PostgreSQL joins bare query terms with AND, so ordinary natural-language questions returned
nothing, and adding an any-term fallback took nDCG from 0.478 to 0.802. And when the semantic half
arrived, the improvement was a measured delta against a committed baseline instead of an assertion."

**Thread 4 — The measurement that was misleading in a way that looked like success.**
"The first hybrid implementation had no distance threshold. nDCG went up, to 0.881. Precision
collapsed from 0.691 to 0.113, and every unanswerable query started returning ten results — because
approximate nearest-neighbour search returns the k closest vectors whether or not anything is
actually close. Reporting the nDCG improvement and stopping there would have been true and badly
misleading. I swept the threshold against the corpus and settled on 0.35, which is the knee: it
keeps most of the ranking gain and recovers nearly all of the precision. The sweep table is
committed, including the row showing what it costs — hybrid is meaningfully worse at recognising a
question it cannot answer."

**Thread 5 — What I would do differently, and what is missing.**
"Three things. There is no authentication — any process that can reach the database can read and
write any project, which is fine for a single-user local deployment and a real gap the moment it is
shared. The metrics registry has correct label discipline but nothing scrapes it, so there is no
operational visibility outside the structured logs. And the architecture specifies write-time secret
screening — pattern rules plus an entropy heuristic, with a default action of reject — which is not
implemented; today the only control is an instruction in the tool description, and an instruction is
not a control. I found the third gap by writing a document that maps every claimed failure behaviour
to the test that proves it, which is also how I discovered three error codes the architecture
promised had never been implemented at all."

---

# PART 39 — SIXTY-PLUS INTERVIEW QUESTIONS

Format for each: **Q**, simple answer, deep answer, implementation evidence, likely follow-up, what
not to say.

---

## Project basics

**1. What does this project do?**
*Simple:* It gives several AI coding clients one shared, versioned place to record and retrieve
project knowledge, and it makes sure a fact that has been replaced can never come back as if it were
still true.
*Deep:* It is a multi-writer, versioned knowledge store with two hard properties: write correctness
under concurrency across independent processes, and read consistency over a corpus that contradicts
itself over time.
*Evidence:* 7 MCP tools; `memories` + `memory_revisions`; `cas_revise.sql`; `filters.py`.
*Follow-up:* "What is the hard part?"
*Don't say:* "It gives AI a memory." That describes a text file.

**2. Why isn't this just a database table?**
*Simple:* A table stores things. It has no idea that one row replaces another.
*Deep:* A notes table plus `LIKE` gives you none of: retirement of one fact by another,
compare-and-set on a logical unit, provenance per version, a budgeted read path, or a
machine-callable protocol. Every property that makes this interesting is absent.
*Evidence:* `superseded_by_id`, `cas_revise.sql`, `memory_context`, the MCP tool surface.
*Follow-up:* "So what is the minimum you would need?"
*Don't say:* "A table would be slow." Speed is not the argument.

**3. Why isn't this just a vector database?**
*Simple:* Similarity has no opinion about what is currently true.
*Deep:* Three reasons — retired and current facts are both returned and the retired one often
matches better; vector stores have no transactions, so lifecycle metadata and vectors live in two
systems that can disagree; and there is no CAS, no unique constraints, no referential integrity.
*Evidence:* `q02` in `queries.yaml`; `semantic.search_by_vector` composing the same stage-0 filter;
`stale_inclusion_rate = 0.000`.
*Follow-up:* "What does pgvector give up compared to a dedicated engine?"
*Don't say:* "Vector databases are bad." They are good at their job; their job is not truth
maintenance.

**4. Why isn't this chat history?**
*Simple:* It stores what a client explicitly decided to record, not conversations.
*Deep:* An MCP server receives the arguments of tool calls and nothing else. It has no access to a
transcript. And even with one, ingesting everything destroys the corpus — precision collapses under
noise. A curated corpus beats an exhaustive one.
*Evidence:* `SERVER_INSTRUCTIONS`; `test_server_instructions_do_not_overclaim`.
*Follow-up:* "Could a client paste a transcript into `content`?"
*Answer:* "Yes, and that would be the client's decision, visible in the call. The server never
obtains it independently."
*Don't say:* "It syncs your chats." That is the one sentence the project's own architecture document
says must never be written.

**5. Does your server read Claude conversations?**
*Simple:* No. It receives tool-call arguments only.
*Deep:* As above.
*Don't say:* Anything that implies otherwise.

**6. How does information get stored?**
*Simple:* The model calls `memory_remember` with the text and a type. Nothing is automatic.
*Deep:* The tool descriptions steer that decision — record durable decisions, constraints, facts,
and current work; not conversational chatter; never credentials.
*Evidence:* `mcp/server.py` tool descriptions; `test_descriptions_carry_the_operating_contract`.
*Follow-up:* "What stops it recording rubbish?"
*Answer:* "Nothing structural. That is a real limitation. What exists is the tool description, plus
deduplication and expiry to limit the damage."

**7. Who decides what is remembered?**
*Simple:* The client's model, guided by the user and the tool descriptions.
*Deep:* That is also what makes supersession meaningful — you can retire an asserted fact, but you
cannot retire an overheard one. Explicit writes give provenance and a defensible answer to "why does
your service have this sentence in it".

---

## MCP

**8. Why MCP?**
*Simple:* It is the standard way for an AI application to call tools in a separate program, so I did
not need a custom integration per client.
*Deep:* It also shaped the architecture: stdio means one server process per client, which means no
shared memory, which is why the database has to be the synchronisation primitive.
*Don't say:* "Because it is trendy."

**9. What does MCP actually do here?**
*Simple:* It advertises seven tools, carries their arguments in and results out over stdio as
JSON-RPC.
*Deep:* Plus schema derivation from type annotations, and the distinction between a protocol error
and a tool error.
*Evidence:* `mcp/server.py`, `mcp/schemas.py`, `mcp/mapping.py`.

**10. What is MCP *not* responsible for?**
Storage, versioning, concurrency, ranking, budgets, and deciding what is true. It is the interface,
not the engine.

**11. What is stdio, and why does it matter?**
*Simple:* The client starts the server as a subprocess and talks over its standard input and output.
*Deep:* It matters because each client starts its *own* subprocess. Two clients means two OS
processes with no shared memory, so all coordination has to be in the database.
*Follow-up:* "What is the biggest operational risk with stdio?"
*Answer:* "stdout is the protocol channel. One stray print corrupts the stream. Logs go to stderr, a
lint rule bans print, and a test spawns the real subprocess to prove stdout stays clean."

**12. Do Claude and Cursor communicate directly?**
No. There is no channel between them and none between the two server processes. Verified by reading
the code: nothing opens a socket or pipe between them.

**13. Does each client start a separate memhub process?**
Yes. That is what stdio means.

**14. Do those processes share Python memory?**
No. Separate operating-system processes.

**15. Where does the shared state live?**
PostgreSQL. It is the only component that can see both writers, which is why every invariant is
enforced there.

**16. How does Claude Desktop connect?**
Through `claude_desktop_config.json`, with an **absolute path** to the server executable and
`MEMHUB_DATABASE_URL` in `env`. `docs/clients.md` records that the absolute path is the single most
common cause of failure, because neither client runs the server from your project directory.

**17. How does Cursor connect?**
Through `~/.cursor/mcp.json` or a project-scoped `.cursor/mcp.json`. Its schema differs from Claude
Desktop's in taking an explicit `type`.

**18. Why are conflicts tool results rather than protocol errors?**
*Simple:* A protocol error means the request was invalid or the server broke. A conflict means the
request was fine and the domain said no.
*Deep:* Returning it as a protocol error hides it from the model, which then cannot recover. It
comes back as an ordinary result with `outcome: "conflict"` plus the winning revision, content, and
author — everything needed to merge and retry in one round trip.
*Evidence:* `ReviseOut`; `test_conflict_is_a_result_not_an_error`.
*Follow-up:* "The architecture said `isError: true` with a structured payload. What changed?"
*Answer:* "The SDK's `ToolError` carries a message string only — `is_error` and `structured_content`
are mutually exclusive in practice. It is recorded as an amendment in the architecture document and
in `mcp/mapping.py`. It is arguably better, because it forces the discriminator into the success
schema, so every caller has to acknowledge that a write can resolve in more than one way."

**19. Why are tool descriptions under snapshot test?**
Because the description is the prompt that steers the model. Changing one word changes system
behaviour with no logic change and no other test failing. The snapshot turns that into a visible
diff in review.

**20. What is a resource, and why is `memory_history` a tool instead?**
*Simple:* Resources are identity-addressed and read-only; tools are model-invoked.
*Deep:* The rule is: identity-addressed and side-effect-free with a stable URI → resource; takes a
query the model composes, or changes state → tool. `memory_history` fits the resource rule, but
clients differ in whether the *model* can pull a resource on demand, so anything the model must
reach autonomously has to be a tool. It is exposed both ways.

---

## Architecture

**21. Walk me through the layers.**
MCP handlers (validate, call one service, map the result) → services (transactions, invariants,
orchestration) → repositories (SQL, always project-scoped) → PostgreSQL. Beside that: `domain/`
(pure types, no I/O), `retrieval/` (read-only), `context/` (selection), `embeddings/` (the port and
the worker), `cli/` (operator commands, not over MCP), `observability/`.

**22. What rule keeps the layers honest?**
`domain/` imports nothing from the other packages. Services never import from `mcp`. No SQL outside
`persistence/`. Handlers stay thin — validate, call, map.

**23. Why is the server stateless?**
Because it is a short-lived subprocess. It holds no authoritative state, so a restart is a
non-event, and no cache can be wrong.

**24. Why PostgreSQL rather than SQLite?**
Multi-process concurrent writers are SQLite's weakest axis — WAL-mode SQLite serialises writers and
surfaces `SQLITE_BUSY`, so the story becomes "retry until the lock frees" rather than "the database
evaluated my predicate atomically and told me I lost". And the correctness mechanisms here are
PostgreSQL features: partial unique indexes, composite FKs, `SKIP LOCKED`, `EvalPlanQual`, GIN,
HNSW.
*Be honest:* SQLite would be simpler, and if the only goal were "a personal tool that works" it
would probably win.

**25. Why not markdown files in git?**
*Simple:* Git solves distribution and leaves both real problems untouched.
*Deep:* Git has the wrong conflict model — a three-way text merge either succeeds and leaves two
contradictory sentences side by side, or fails and hands a human conflict markers. Neither answers
"which of these is currently true", because that is a semantic question and a merge is a syntactic
operation. And git only adjudicates at push time, whereas the damage happens at write time. Worse,
it cannot filter at read time: a retired line stays in the file until a human deletes it, and until
then every retrieval returns it as live.
*Evidence:* `docs/architecture.md` §1.3, added in its own commit.

---

## Database and lifecycle

**26. What is a memory?**
One logical statement with an identity: a row in `memories` holding type, status, importance,
expiry, and a pointer to the current revision. It never holds content.

**27. What is a revision?**
One version of that statement's text: a row in `memory_revisions`, keyed `(memory_id, revision_no)`,
append-only.

**28. Revision vs supersession?**
Revision is the same fact refined — one memory, many versions, guarded by CAS on the revision
number. Supersession is a *different* memory retiring this one — many old, one new, with its own
author and timestamp, guarded by CAS on the status. Forcing the replacement to be "revision 2" would
mean lying about who wrote it and when, and could not express one memory retiring three.

**29. Why preserve old revisions?**
Because someone will ask why a decision changed. Retirement removes a memory from retrieval; it must
not remove it from the record, or the system is just deleting inconvenient history.

**30. What makes information stale?**
It was true when written and is not current now. Either another memory superseded it, it was
forgotten, or it expired.

**31. Why is EXPIRED not a status?**
Because a stored status needs a sweeper to become true, and between expiry and sweep the column
would lie. `expires_at IS NULL OR expires_at > now()` is always correct with zero moving parts.

**32. How do you prevent stale information from returning?**
One filter, in one function, that every retrieval path composes on: project, `status='ACTIVE'`, not
expired, current revision only. Retired memories never enter the candidate set.

**33. Why not just rank stale memories lower?**
Four reasons. It leaks the moment someone asks for more results. The penalty would have to beat an
unbounded, corpus-dependent score. It couples correctness to tuning, so every future ranking change
risks resurrecting a retired memory. And it cannot be stated as a guarantee.
*Evidence:* the every-limit loop in `test_the_retired_fact_leaves_retrieval_entirely`.

**34. What does one row in each table represent?**
`projects` — a namespace. `memories` — a logical fact's identity and lifecycle. `memory_revisions` —
one version of its text. `memory_dedup_keys` — one distinct active fact. `memory_attestations` — one
client's assertion of one fact. `idempotency_keys` — one logical request and its answer.
`audit_events` — one thing that happened. `memory_embeddings` — one vector for one revision under
one model. `embedding_jobs` — one pending piece of embedding work.

**35. Why is `project_id` duplicated onto `memory_revisions`?**
Two reasons: it makes the composite foreign key possible, which is what pins isolation; and it lets
project-scoped scans avoid a join.

**36. Why does `audit_events` have no foreign key?**
Because an audit row must survive the thing it describes. An operator purge destroys a memory and
every revision of it, and the record that the purge happened has to outlive that. A CASCADE would
erase the evidence along with the subject.

---

## Concurrency

**37. What is concurrency here, concretely?**
Two independent server processes, started by two different clients, writing to the same database at
the same time with no coordination between them.

**38. What is a race condition?**
A bug whose outcome depends on the timing of concurrent operations.

**39. What is a lost update?**
Two writers read the same value, both write based on it, and the second silently erases the first's
change — and nobody is told.

**40. What is CAS?**
Compare-and-set: write only if the current value still equals what you read. Here the compare and
the set are one SQL statement, so nothing can slip in between.

**41. What is `expected_revision`?**
The caller's claim about which revision it read. It is the compare half.

**42. Why optimistic concurrency rather than locking?**
Conflicts are rare; the loser gets an informative refusal instead of a wait; and holding a lock
across an MCP round trip — while a language model decides what to write — would be dangerous.
*Tradeoff:* under heavy contention, 49 of 50 writers do work that is then refused.

**43. Why READ COMMITTED?**
Because of the conflict semantics it gives. Under SERIALIZABLE the same collision raises `40001` and
the application must retry — also correct, but it converts a clean domain outcome into a retry loop,
and the test assertion stops being exact. READ COMMITTED plus explicit CAS gives 1 success and 49
conflicts, deterministically, with no retries.

**44. Explain why the CAS is correct at the storage-engine level.**
The `UPDATE` takes a row-level exclusive lock, so two matching transactions cannot both proceed. The
second blocks. When the first commits, the second wakes and — this is the crux, and it is specific
to READ COMMITTED — PostgreSQL does not proceed with the row version it originally found. It walks
to the newest committed version and re-evaluates the `WHERE` clause against it. That is
`EvalPlanQual`. The predicate now fails, the row is skipped, and the update reports zero rows. The
read and the write are the same statement, so there is no window.

**45. What is MVCC?**
Multi-Version Concurrency Control. An update writes a *new version* of the row rather than
overwriting it, and each transaction sees the version appropriate to its snapshot. Readers never
block writers and writers never block readers.

**46. What happens if the CAS is skipped by a bug?**
The invariant still holds. `PRIMARY KEY (memory_id, revision_no)` rejects a second revision 5, and
`UNIQUE (memory_id) WHERE is_current` rejects a second current revision. Both raise `23505`.

**47. How do you avoid deadlocks?**
Every write path takes the `memories` row lock first, before touching revisions, and supersession
locks its targets in ascending UUID order. A single consistent lock order across all writers means
no cycle can form.

**48. What does the concurrency test prove, and what does it not?**
*Proves:* lost updates cannot happen through revise; the conflict outcome is deterministic; losers
get enough to recover; revision numbers stay gapless; history is appended; invariants still hold.
*Does not prove:* throughput, scalability, behaviour under SERIALIZABLE, anything about network
partitions — and it is 50 concurrent *callers*, not 50 MCP clients.

**49. What is the connection-pool trap?**
A 50-way test against a 10-connection pool measures five sequential waves of ten and passes for the
wrong reason. The fixture asserts `pool_size > concurrency` and then proves it at runtime by
collecting 50 distinct `pg_backend_pid()` values behind a barrier.

---

## Idempotency and deduplication

**50. What does idempotency mean?**
Doing it twice has the same effect as doing it once.

**51. Why do clients retry?**
Because a dropped connection is indistinguishable from a failed request. The client knows what it
sent and not what happened.

**52. Walk me through the protocol.**
`INSERT ... ON CONFLICT DO NOTHING` to claim the key. One row back means we own it: do the write,
then set `COMPLETED` with the stored response, all in one transaction. Zero rows means someone else
owns it: `SELECT ... FOR SHARE` blocks until their transaction resolves. A row comes back means they
committed — verify the fingerprint and replay their response. No row means they rolled back and the
key is free, so loop, bounded to three attempts.

**53. Why `FOR SHARE`?**
It takes a shared row lock that blocks until the owning transaction finishes. That blocking is how a
duplicate waits for the original without polling or sleeping.

**54. What is the request fingerprint for?**
So a key reused with a *different* payload fails loudly instead of silently returning a result for a
request the caller never made. It is SHA-256 over canonical JSON of the meaningful arguments,
deliberately excluding incidental fields like `source` so a cosmetically different retry still
replays.

**55. Idempotency vs deduplication?**
Idempotency: one client retrying the same request, keyed on a caller-supplied request id, answered
by replaying the stored response. Deduplication: two different clients asserting the same fact, keyed
on a normalised content hash, answered by returning the existing memory and recording an attestation.
They solve different problems, and neither covers the other's case.

**56. Why does `revise` need an idempotency key if CAS already prevents duplicates?**
It does not need one for correctness. Without a key, a retry of an already-applied revise fails the
version check and is reported as a *conflict* — telling the caller it lost a race it actually won.
The key turns that into a clean replay. That is why the claim happens before the CAS. Knowing when
idempotency is required for correctness versus for ergonomics is the actual insight.

**57. Why is the content hash SHA-256 rather than something fast?**
Because it backs a uniqueness constraint, and a collision would silently merge two distinct project
facts. Thirty-two bytes per revision is not a cost worth optimising against that.

**58. What happens to a dedup key when a memory is retired?**
It is deleted. Otherwise retiring "The job queue runs on Redis." would permanently block anyone from
storing that sentence again, including legitimately if the decision were reversed.

---

## Transactions and constraints

**59. What is a transaction and why does supersession need one?**
All-or-nothing. Without it, a crash between creating the replacement and retiring the old one leaves
both ACTIVE and contradicting each other, with nothing that knows to repair it.

**60. Why enforce rules in the database rather than in Python?**
A Python check can be bypassed by a new code path, a script, or a migration. It is not atomic —
check-then-insert races. A constraint documents itself in the schema. And it catches the case you
did not think of. The pattern is: the constraint is the guarantee, the validator is the good error
message.

**61. Name the constraint you are proudest of.**
The composite foreign key `(superseded_by_id, project_id) → (id, project_id)` on `memories`. It makes
cross-project supersession **unrepresentable**, not merely disallowed. It only works because
`memories` also carries `UNIQUE (id, project_id)`.

**62. What is a savepoint and where is one used?**
A marker inside a transaction you can roll back *to*, undoing only what came after it. `remember`
uses one around the speculative insert for deduplication, so a dedup hit undoes only that insert and
leaves the enclosing transaction — including an idempotency claim — intact.

---

## Retrieval

**63. What is full-text search here?**
A generated `tsvector` column, a partial GIN index on current revisions,
`websearch_to_tsquery('english', ...)` for parsing, and `ts_rank_cd` for scoring.

**64. Why `websearch_to_tsquery` rather than `to_tsquery`?**
Because a model composes the query and will write things like `"task queue" -redis`. `to_tsquery`
raises a syntax error on anything that is not a strict boolean expression, turning an ordinary
question into a tool failure.

**65. Why `ts_rank_cd`?**
Cover density rewards query terms appearing near each other. Memories are one or two sentences, so
proximity is close to the whole signal.

**66. What was the any-term fallback for?**
`websearch_to_tsquery` joins bare terms with AND, so natural-language questions demanded every word
and returned nothing. Retrying with any-term matching when the strict query finds nothing took nDCG
from 0.478 to 0.802. Precision is preserved for queries that match strictly, and queries using
explicit syntax are never rewritten.

**67. What is an embedding?**
A list of numbers produced by a model, representing text's meaning, such that similar meanings get
similar directions. Here 384 numbers, unit-normalised.

**68. What is cosine distance?**
`1 - cosine similarity`, between 0 and 2. Smaller means more similar. The `<=>` operator in pgvector.

**69. What is HNSW?**
Hierarchical Navigable Small World — a graph-based approximate nearest-neighbour index. Layers of
links let a search jump across the space coarsely and then refine. Fast, good recall, no need for
the whole dataset up front.

**70. Why is a distance threshold necessary?**
Because ANN returns the k closest vectors whether or not anything is close. Without one, precision
collapsed from 0.691 to 0.113 and every unanswerable query returned ten results.

**71. How did you choose 0.35?**
By sweeping it against the graded corpus. It is the knee: between 0.45 and 0.35 precision climbs
from 0.144 to 0.671 while nDCG gives up 0.023; below 0.30 the threshold starts excluding genuine
matches and the numbers converge on lexical-only.

**72. What is the filtered-ANN problem?**
An HNSW scan collects `ef_search` neighbours and *then* the filter is applied, so a restrictive
filter can leave only a handful surviving and the query silently under-returns. pgvector 0.8's
iterative scans continue until enough rows survive; it is enabled per query with `SET LOCAL`, because
it costs more when the filter is not selective.

**73. Why hybrid?**
Because the two retrievers fail on different queries. `jwt` went from 0.000 to 1.000 because the
stemmer never matches `JWTs`. Overall, nDCG 0.803 → 0.853.

**74. What is RRF?**
Combining ranked lists by position rather than score: `sum over retrievers of 1/(k + rank)`, with
`k = 60`.

**75. Why not combine raw scores?**
`ts_rank_cd` is unbounded and corpus-dependent; cosine distance is bounded and points the other way.
Adding them is meaningless. And per-query min-max normalisation is worse — it scales a mediocre best
match to 1.0 identically to a perfect one, so the two queries become indistinguishable at exactly the
moment the difference matters.

**76. Why k = 60?**
It damps the gap between top positions. Without it rank 1 is worth twice rank 2, which overweights a
single retriever's confident mistake. 60 is the value from the original TREC work.

**77. What happens if one retriever returns nothing?**
Those documents simply do not appear in that list and contribute nothing. RRF over one list is just
that list's order. No zero-filling, no special case — which is exactly right for a memory whose
embedding has not been computed yet.

---

## Background work

**78. Why background embeddings?**
Because a failing model must never fail a write. Inline, one slow inference call holds a row lock
and blocks every other writer, and a model outage becomes a write outage.

**79. What is an outbox?**
A table where a row describing follow-up work is written **in the same transaction** as the change it
relates to, so either both exist or neither does.

**80. What is the dual-write bug?**
Writing to two systems without a transaction spanning them. Commit the memory, crash before
enqueuing, and the memory exists forever with no vector and nothing that knows to fix it.

**81. Why PostgreSQL for jobs?**
Because the outbox requires the job row and the write to be one transaction, which rules out any
other store. And the durability, atomicity, and locking a queue needs were already in the database
being written to anyway.

**82. What does SKIP LOCKED mean?**
Skip over rows another transaction has locked instead of waiting for them. That is what turns a table
into a queue several workers can drain at once.

**83. Why `FOR UPDATE OF j`?**
So only the job row is locked. The query joins to `memory_revisions` for the content, and locking
that too would make embedding contend with revision writes for no reason.

**84. What happens if the worker crashes?**
Before the claim: nothing happened. After the claim but before the commit: the transaction aborts,
the locks release, the jobs revert to PENDING and another worker takes them. After the commit: the
vectors are stored and the jobs are DONE. There is no state where a job is DONE without its vector.

**85. What happens if the embedder is permanently broken?**
Jobs back off exponentially from 2 seconds and are marked `DEAD` after five attempts, with the error
stored. Writes keep working, search keeps working, and `semantic_coverage` reports how much of the
corpus is missing.

**86. When would you move to a real queue?**
Thousands of jobs per second; fan-out to several kinds of consumer; cross-service delivery where
producer and consumer have separate databases; or delivery guarantees this does not offer. None
apply at this scale.

---

## Context and budgets

**87. What is token-budgeted context?**
Returning the most useful brief that fits a stated number of tokens, rather than a list of matches.

**88. How is it different from search?**
Search answers "what matches this query". This answers "given this much room, what is worth
knowing". The limit is a cost, not a count.

**89. What is MMR?**
Maximal Marginal Relevance — balancing how good an item is against how different it is from what is
already selected, so a brief is not three restatements of one decision. Lambda is 0.7.

**90. Why Jaccard word overlap rather than embeddings for diversity?**
Because diversity here is about redundant *phrasing*, which word overlap captures well — and because
making the context builder depend on the embedding pipeline would mean a brief could not be produced
while the outbox was behind.

**91. Why is this a knapsack problem?**
Items with a cost (tokens) and a value (score), under a budget. The fill is greedy on value per
token, which is within a known bound and is what the size of the problem warrants; exact dynamic
programming is not worth it at fewer than 500 candidates.

**92. How do you count tokens if you don't know the client's model?**
You cannot, exactly. So the estimator is a calibrated heuristic biased to over-count, and the
contract is stated: never exceed the budget, may under-fill by roughly 10%. Using `tiktoken` would
look more precise and be less honest, because it is the wrong tokeniser for Claude.

**93. Tell me about the calibration.**
The divisor started at 3.6 and under-estimated 2 of 33 real memories, because technical writing full
of identifiers tokenises far more densely than prose — the densest sample was 3.25 characters per
token. Corrected to 3.2, none under-count and the worst case is 4.8% over. The cost is roughly 32%
over-estimation on average, which is the price of the guarantee.

**94. Why is the output ordering deterministic?**
Because a brief whose contents shuffle between identical calls cannot be snapshot-tested and cannot
be safely cached. The sort is type order, then score, then memory id.

---

## Failure handling

**95. What are the failure codes?**
`BACKEND_UNAVAILABLE` and `BACKEND_BUSY` — safe to retry, nothing ran. `DEADLINE_EXCEEDED` — the
query was too slow; the server is healthy, so retrying re-runs the same slow query. `UNKNOWN_OUTCOME`
— not safe to retry.

**96. What is UNKNOWN_OUTCOME?**
The connection died while a statement was in flight. The transaction either committed just before
the drop or it did not, and the acknowledgement that would have said which is exactly what was lost.

**97. What if COMMIT succeeds but the connection disappears?**
That is precisely this case. The response does not claim the write failed. It names the two ways to
resolve it: replay the idempotency key, which returns the stored response if the write landed, or
re-read. Retrying without a key risks writing twice.

**98. How do you distinguish it from "the database is down"?**
`DBAPIError.connection_invalidated` is set when a connection that had already been established dies;
a connection that never opened leaves it False. That is exactly the line between "nothing ran" and
"something may have run". The check must come *before* the SQLSTATE check, because in a real
mid-flight drop both signals are present. Inverting the order makes three tests fail.

**99. What happens when the semantic leg times out?**
The search degrades to lexical-only and says so, because the lexical results are already in hand and
half an answer beats an error. Only SQLSTATE `57014` degrades; any other database error is re-raised,
because swallowing all of them would turn a genuine bug in the vector query into a search that
silently returns lexical results forever.

**100. Why does the server refuse to start on a schema mismatch?**
Because the alternative failure is quiet. A behind database fails on the first missing column, which
the client reports as the tool being broken and nobody looks at migrations. An *ahead* database is
worse — it mostly works, until this process writes a row that newer constraints were added to
prevent, and the failure surfaces later as corrupt data.

---

## Testing, measurement, and production

**101. How did you test concurrency?**
Fifty coroutines, each on its own connection in its own transaction, released together by a barrier,
against real PostgreSQL — with a fixture that first proves the pool hands out fifty distinct
backends. Exactly one succeeds and 49 get conflicts carrying the winning revision, then the invariant
suite runs.

**102. Why real PostgreSQL rather than mocks?**
Because every guarantee is a PostgreSQL behaviour: partial unique indexes, `EvalPlanQual`,
`SKIP LOCKED`, `FOR SHARE`, `ON CONFLICT` waiting on an uncommitted insert, GIN, HNSW. A test passing
on SQLite would prove nothing about the system being shipped.

**103. How do you keep a real-database test suite fast?**
Migrations run once into a template database; each module clones it with
`CREATE DATABASE ... TEMPLATE`, which PostgreSQL implements as a file copy. Within a module, each
test runs in a transaction that is rolled back — except concurrency tests, which cannot use that,
because two writers inside one transaction are not two writers.

**104. What is mutation testing and what did it find?**
Deliberately breaking a mechanism to confirm a test notices. Removing the status condition from the
stage-0 filter fails five tests; dropping the CAS predicate fails the concurrency tests; inverting
the failure classifier's branch order fails three. And one of those experiments found a **vacuous
test**: with the CAS predicate removed there were zero losers, so a loop over losers never executed
and every assertion inside it passed trivially. The fix was to assert the population size before
iterating it.

**105. What are precision, recall, and nDCG?**
Precision@10 is the fraction of returned results that are relevant. Recall@10 is the fraction of all
relevant memories that appeared. nDCG@10 is a ranking measure using graded relevance, discounted by
position and normalised against the ideal ordering, so it rewards putting the best answers near the
top.

**106. Why nDCG rather than MRR?**
MRR credits only the first relevant result. These queries usually have several relevant memories at
different grades, and MRR would score "perfect answer first, everything else missing" identically to
"perfect answer first, everything else present".

**107. How did you evaluate retrieval?**
200 memories, 34 graded queries with `forbidden` lists, judgments written before any strategy ran,
metrics gated against a committed baseline with a 0.02 tolerance so a regression fails the build.

**108. What can your evaluation not prove?**
200 memories is small; the corpus is synthetic and single-author; the judgments are one person's
opinion with no inter-annotator agreement; 34 queries is a small sample, so a 0.05 nDCG delta is not
statistically robust; hybrid was measured locally once with one model; and it says nothing about
whether the right things were recorded in the first place.

**109. What do your performance numbers show?**
At 1k, 10k, and 100k memories, one selective full-text query's server-side time went from 0.36 ms to
0.67 ms — the corpus grew 100× and the query time grew 1.9×. That shows cost tracking the answer
rather than the table, for that query shape, on one machine. It is not a general scalability claim.

**110. Why does the plan show a sequential scan at 100k rows?**
Because with `LIMIT 10` the scan stops as soon as it has ten matches — `Rows Removed by Filter` is a
small fraction of the table — so index access plus heap fetches would cost more. Two earlier versions
of the benchmark asserted the GIN index would appear in the plan, and both were wrong. The right
assertion is on the property, not the mechanism.

**111. What would you improve for production?**
Authentication and per-caller authorization, which arrive with the HTTP transport. A metrics
exporter — the label discipline is right but nothing scrapes it. Scheduled retention. And the secret
screening the architecture specifies, which is not implemented; today the only control is an
instruction in a tool description, and an instruction is not a control.

**112. What is the biggest gap between your design document and your code?**
Secret screening. The architecture specifies pattern rules plus an entropy heuristic with a default
action of reject and an audited override. None of it exists. I also found, while writing the
failure-modes document, that three error codes the architecture promised had never been implemented
at all — which is the argument for writing that kind of document: the prose had been true when
written and had quietly stopped being true.

**113. If you rebuilt it, what would you change?**
Fix the two stale tool descriptions immediately, because they are the prompt that steers the model
and one of them tells it supersession is unavailable. Forward `semantic_coverage`, `degraded`, and
`match_strategy` into the search response, so search cannot be silently partial through MCP. And
schedule the garbage collector rather than leaving it manual.

---

# PART 40 — IMPORTANT CODE, EXPLAINED LINE BY LINE

Only the snippets that carry a guarantee. Each one gets: the code, a line-by-line reading, what
breaks if a line is removed, and which project concept it implements.

---

## 40.1 The compare-and-set

`src/memhub/persistence/sql/cas_revise.sql`

```sql
UPDATE memories
   SET current_revision_no = current_revision_no + 1,
       updated_at          = now()
 WHERE id                  = :memory_id
   AND project_id          = :project_id
   AND current_revision_no = :expected_revision
   AND status              = 'ACTIVE'
RETURNING current_revision_no AS new_revision;
```

**Line 1 — `UPDATE memories`.** The identity row, not the content row. It is the single point every
writer must pass through, so it is where they can be adjudicated.

**Line 2 — `SET current_revision_no = current_revision_no + 1`.** The increment is computed by the
database from the row's own value, not from a number Python calculated. *Remove it and* the revision
never advances, so every subsequent writer with the same `expected_revision` also succeeds — the
lost update, with extra steps.

**Line 3 — `SET updated_at = now()`.** The database clock. *Replace it with an application
timestamp and* two server processes with skewed clocks can disagree about the order of events.

**Line 4 — `WHERE id = :memory_id`.** Which memory. *Remove it and* every memory in the table is
updated.

**Line 5 — `AND project_id = :project_id`.** Namespace isolation in the write path. *Remove it and*
a caller holding an id from another project could modify it. This is also why a cross-project
supersession attempt matches zero rows instead of raising.

**Line 6 — `AND current_revision_no = :expected_revision`.** **The compare.** This is the entire
mechanism. *Remove it and* every writer succeeds and the last one wins silently. Mutation-tested:
removing it makes the concurrency tests fail, which is the only evidence it was ever doing anything.

**Line 7 — `AND status = 'ACTIVE'`.** Two jobs. It stops a revise from resurrecting a tombstoned or
superseded memory. And it makes revise, forget, and supersede serialise on the same row lock rather
than racing around each other. *Remove it and* you can revise a memory another client is retiring.

**Line 8 — `RETURNING current_revision_no AS new_revision`.** The winner gets the new number in the
same round trip. And **zero rows returned is the conflict signal** — no exception, no error code.

**Implements:** optimistic concurrency, lost-update prevention, `EvalPlanQual` under READ COMMITTED.

---

## 40.2 The stage-0 filter

`src/memhub/retrieval/filters.py`

```python
stmt = stmt.where(Memory.project_id == project_id)

if include_retired:
    return stmt

return stmt.where(
    Memory.status == MemoryStatus.ACTIVE.value,
    or_(Memory.expires_at.is_(None), Memory.expires_at > func.now()),
    MemoryRevision.is_current,
)
```

**Line 1 — the project scope, applied before the branch.** *Remove it and* a debug path leaks across
projects. Its position above the `if` is deliberate: isolation is not negotiable, even for
`include_retired`.

**Line 3 — the escape hatch.** Used by `memory_history` and one debug path only. *Widen its usage
and* the guarantee disappears; the module docstring says to grep for it.

**Line 6 — `status == 'ACTIVE'`.** Excludes SUPERSEDED and DELETED. *Remove it and* five stale-memory
tests fail — verified by mutation.

**Line 7 — the expiry predicate.** Evaluated at read time, against `now()` on the server. *Store an
`EXPIRED` status instead and* the column lies between expiry and sweep. *Use a Python timestamp
instead and* two processes disagree about what has expired.

**Line 8 — `MemoryRevision.is_current`, written bare.** *Write it as `.is_(True)` and* PostgreSQL can
no longer prove the query's predicate implies the partial index's predicate, so the GIN index becomes
unusable — silently, with correct results and green tests, until the corpus is large enough for
anyone to notice. Guarded by `tests/unit/test_filter_sql.py`.

**Implements:** structural stale-memory exclusion, project isolation, derived expiry.

---

## 40.3 The dedup claim

`src/memhub/persistence/sql/claim_dedup_key.sql`

```sql
INSERT INTO memory_dedup_keys (project_id, hash_version, content_hash, memory_id)
VALUES (:project_id, :hash_version, :content_hash, :memory_id)
ON CONFLICT (project_id, hash_version, content_hash) DO NOTHING
RETURNING memory_id;
```

**Line 1–2 — one statement, not two.** *Replace with "SELECT to check, then INSERT" and* two clients
asserting the same fact at the same instant both pass the SELECT and both create a memory.

**Line 3 — `ON CONFLICT ... DO NOTHING`.** The primary key adjudicates atomically. The three columns
named are exactly the primary key.

**Line 4 — `RETURNING memory_id`.** Zero rows means someone else holds it. The caller then reads the
holder and records an attestation instead of creating a duplicate.

**Implements:** deduplication as a real constraint rather than a check.

---

## 40.4 The idempotency claim and wait

`claim_idempotency.sql`

```sql
INSERT INTO idempotency_keys
    (project_id, client_request_id, operation, request_fingerprint, state)
VALUES (:project_id, :client_request_id, :operation, :request_fingerprint, 'IN_PROGRESS')
ON CONFLICT (project_id, client_request_id) DO NOTHING
RETURNING client_request_id;
```

**The subtlety.** Under READ COMMITTED, if a *concurrent uncommitted* transaction has already
inserted this key, `ON CONFLICT DO NOTHING` does not return immediately — it **waits** for that
transaction to finish, then returns zero rows. So zero rows already means "the other writer has
resolved", not "might still be in flight".

`wait_idempotency.sql`

```sql
SELECT state, response, request_fingerprint, operation
  FROM idempotency_keys
 WHERE project_id = :project_id AND client_request_id = :client_request_id
   FOR SHARE;
```

**`FOR SHARE`.** A shared row lock that blocks until the owning transaction commits or rolls back.
*Remove it and* you would need a polling loop with sleeps, which is slower and racier.

**One row back** — the owner committed; replay its stored response. **No rows** — the owner rolled
back, which rolled its INSERT back too; the key is free, so loop.

**Implements:** race-free idempotency, and the recovery path for `UNKNOWN_OUTCOME`.

---

## 40.5 The savepoint around deduplication

`src/memhub/services/memories.py`

```python
savepoint = await session.begin_nested()

memory, revision = await repo.create(project_id, ...)

claimed = await truth.claim_dedup_key(
    project_id, content_hash=digest, hash_version=HASH_VERSION, memory_id=memory.id
)

if claimed is None:
    await savepoint.rollback()
    holder = await truth.dedup_key_holder(...)
    return await _deduplicate(session, project_id, holder, ...)

await savepoint.commit()
```

**Line 1 — the savepoint.** *Remove it and* the only way to undo the speculative insert is rolling
back the whole transaction, which would discard the idempotency claim made moments earlier and
break any caller managing its own transaction.

**Line 3 — the speculative insert.** It has to happen first, because the dedup key carries a foreign
key to the memory. The key cannot be claimed for a memory that does not exist yet.

**Line 9 — `rollback()` to the savepoint.** Undoes only the memory and revision inserts. The memory
that was created and rejected leaves no trace.

**Line 11 — read the holder.** Zero rows from the claim tells you the key is taken but not by whom.

**Implements:** deduplication without check-then-insert, inside a caller-owned transaction.

---

## 40.6 The supersession statement

`src/memhub/persistence/sql/cas_supersede.sql`

```sql
UPDATE memories
   SET status           = 'SUPERSEDED',
       superseded_at    = now(),
       superseded_by_id = :winner_id,
       updated_at       = now()
 WHERE id         = :memory_id
   AND project_id = :project_id
   AND status     = 'ACTIVE'
RETURNING id;
```

**Lines 2–3 must be set together.** A `CHECK` constraint enforces
`(status = 'SUPERSEDED') = (superseded_at IS NOT NULL)`. *Set one without the other and* the write is
rejected.

**Line 4 — `superseded_by_id`.** Another `CHECK` requires it to be non-null whenever the status is
SUPERSEDED, and the composite foreign key requires it to name a memory **in the same project**.

**Line 8 — `AND status = 'ACTIVE'`.** The compare-and-set. Two clients retiring the same memory
concurrently produce one winner and one "already retired", not two conflicting supersession records.

**Line 9 — `RETURNING id`.** Zero rows means already retired, already deleted, or in another project.
All three are reported back in `not_superseded` rather than silently skipped.

**And the caller detail that prevents deadlocks:** `_apply_supersession` iterates
`sorted(set(targets))` — ascending UUID order, the same lock order every other write path uses.

**Implements:** supersession as an atomic act, project isolation, deadlock freedom.

---

## 40.7 The outbox enqueue

`src/memhub/persistence/repositories/truth.py`

```python
await self._session.execute(
    text(
        "INSERT INTO embedding_jobs (memory_id, revision_no, project_id, model) "
        "VALUES (:mid, :rev, :pid, :model) "
        "ON CONFLICT (memory_id, revision_no, model) DO NOTHING"
    ),
    {"mid": memory_id, "rev": revision_no, "pid": project_id, "model": model},
)
```

**The whole point is what is *not* here: there is no commit.** It runs in the caller's transaction,
alongside the memory and the revision. *Move it outside and* it becomes the dual-write bug.

**`ON CONFLICT DO NOTHING`.** A revision needs embedding once per model; a re-enqueue after a retry
must not create a second job.

**Implements:** the transactional outbox.

---

## 40.8 The SKIP LOCKED claim

`src/memhub/embeddings/worker.py`

```sql
SELECT j.id, j.memory_id, j.revision_no, j.project_id, j.attempts, r.content
  FROM embedding_jobs j
  JOIN memory_revisions r
    ON r.memory_id = j.memory_id AND r.revision_no = j.revision_no
 WHERE j.state = 'PENDING'
   AND j.model = :model
   AND j.next_attempt_at <= now()
 ORDER BY j.next_attempt_at
   FOR UPDATE OF j SKIP LOCKED
 LIMIT :batch_size
```

**Line 5 — `state = 'PENDING'`.** Matches the partial index
`ix_embedding_jobs_next_attempt_at WHERE state = 'PENDING'`, which is near-empty at steady state.

**Line 6 — `model = :model`.** A worker only claims work for the model it can actually produce.
*Remove it and* a worker with one model would claim and fail jobs meant for another.

**Line 7 — `next_attempt_at <= now()`.** This is what implements backoff. A deferred job is invisible
until its time comes.

**Line 9 — `FOR UPDATE OF j`.** Locks the **job** row only. *Drop the `OF j` and* the joined revision
is locked too, making embedding contend with revision writes for no reason.

**Line 9 — `SKIP LOCKED`.** *Remove it and* a second worker blocks on the first's rows and the pair
runs at the speed of one.

**Line 10 — `LIMIT :batch_size`.** Bounds how much one worker holds locked at a time.

**Implements:** PostgreSQL as a durable job queue drained by several workers.

---

## 40.9 The failure classifier's branch order

`src/memhub/mcp/mapping.py`

```python
if pgcode == QUERY_CANCELED:                       # 57014
    return DeadlineExceededError(..., retryable=False)

if exc.connection_invalidated:
    return UnknownOutcomeError(
        "the connection to the database was lost while the request was in "
        "flight. The write may or may not have committed. Replay the same "
        "request with its idempotency key to find out, or re-read before "
        "retrying - retrying without a key risks writing twice.",
        retryable=False,
    )

if pgcode in CONNECTION_LOST or isinstance(exc.orig, ConnectionError | socket.gaierror):
    return BackendUnavailableError(..., retryable=True)

return None
```

**Line 1 — the deadline check first.** A cancelled statement is a healthy server, and reporting it
as an outage would send the caller into a retry loop re-running the same slow query.

**Line 4 — `connection_invalidated` before the SQLSTATE.** **This ordering is the load-bearing
line.** In a real mid-transaction drop, *both* signals are present: SQLSTATE `08006` appears either
way, and `connection_invalidated` distinguishes a connection that had been established and then died
from one that never opened. *Swap lines 4 and 13 and* every mid-flight disconnect is reported as
safely retryable, which is how duplicate writes happen. Inverting it makes three tests fail.

**Lines 5–10 — the message.** It does not say the write failed. It names the two ways to resolve the
ambiguity. That is the whole product decision behind `UNKNOWN_OUTCOME`.

**Line 16 — `return None` for anything unrecognised.** *Guess instead and* an unfamiliar failure
arrives dressed as a known one, carrying confident and possibly wrong advice about whether retrying
is safe.

**Implements:** actionable failure classification, and the link back to idempotency.

---

## 40.10 Reciprocal Rank Fusion

`src/memhub/retrieval/fusion.py`

```python
for name, ranking in rankings.items():
    weight = weights.get(name, 1.0)
    positions[name] = {}
    for index, memory_id in enumerate(ranking):
        rank = index + 1
        positions[name][memory_id] = rank
        scores[memory_id] = scores.get(memory_id, 0.0) + weight / (k + rank)

fused.sort(key=lambda result: (-result.score, str(result.memory_id)))
```

**Line 4–5 — `rank = index + 1`.** Ranks count from 1, so the best document divides by `k + 1`.

**Line 7 — the accumulation.** `scores.get(memory_id, 0.0)` is what makes a missing document need no
special case: it simply never gets a term from that retriever. *Zero-fill instead and* you would skew
every score by a constant that depends on how many retrievers ran.

**`k + rank` in the denominator.** *Set `k = 0` and* rank 1 is worth twice rank 2, which overweights
one retriever's confident mistake.

**Line 9 — the tiebreak on `str(memory_id)`.** *Remove it and* documents with equal fused scores come
back in dictionary insertion order, which is stable within a process and not across them — and
unstable output cannot be snapshot-tested.

**Implements:** combining incomparable score scales without inventing a shared one.

---

## 40.11 The greedy fill with MMR

`src/memhub/context/builder.py`

```python
remaining.sort(key=lambda c: (-c.value_density, str(c.memory.memory_id)))
while remaining:
    best, best_value = None, float("-inf")
    for candidate in remaining:
        if spent + candidate.tokens > allowance:
            continue
        novelty = 1.0 - max(
            (similarity(candidate.memory.content, text) for text in chosen_text),
            default=0.0,
        )
        value = MMR_LAMBDA * candidate.value_density + (1 - MMR_LAMBDA) * novelty * (
            candidate.value_density or 1e-9
        )
        if value > best_value:
            best, best_value = candidate, value

    if best is None:
        break

    remaining.remove(best)
    redundant = any(
        similarity(best.memory.content, text) >= DUPLICATE_SIMILARITY for text in chosen_text
    )
    if redundant:
        rejected.add(best.memory.memory_id)
        continue

    taken.append(best)
    chosen_text.append(best.memory.content)
    spent += best.tokens
```

**Line 1 — sort by value density, tiebreak on id.** Score per token, so a long memory has to earn its
space. The tiebreak is what makes the output deterministic.

**Line 5 — the budget check.** *Remove it and* the budget is exceeded, which is a correctness bug
rather than a quality one.

**Lines 7–10 — novelty.** `1 - (the highest similarity to anything already chosen)`. A candidate
unlike everything selected scores 1.0; a near-duplicate scores near 0.

**Line 11 — the MMR blend.** 0.7 relevance, 0.3 novelty. *Set lambda to 1.0 and* near-duplicates fill
the brief. *Set it to 0.0 and* the brief becomes unrelated trivia.

**Line 17 — `if best is None: break`.** Nothing left that fits. Without it, the loop never ends.

**Lines 21–25 — the hard duplicate rejection.** MMR *discourages* duplicates; this **forbids** them
above 0.6 Jaccard similarity. Note that the rejected id is recorded and returned, so a later
redistribution pass does not reconsider it — otherwise the drop tally would count the same rejection
twice.

**Implements:** knapsack fill, diversity, and the drop accounting the budget report exposes.

---

## 40.12 The transaction scope

`src/memhub/persistence/engine.py`

```python
@asynccontextmanager
async def session_scope(factory):
    async with factory() as session:
        try:
            yield session
        except BaseException:
            await session.rollback()
            raise
        else:
            await session.commit()
```

**`except BaseException`, not `except Exception`.** *Narrow it and* an `asyncio.CancelledError` — a
cancelled tool call, a client disconnecting — would skip the rollback and leave the transaction in an
undefined state.

**`raise` after the rollback.** The error still propagates to `@domain_errors` for classification.

**`else: commit()`.** Commit only when the body completed without raising.

**Why this one small function matters.** It is the transaction boundary for every MCP tool call.
Everything atomic in this system is atomic because of these nine lines.

**Implements:** one transaction per operation.

---

# GLOSSARY

Alphabetical, self-contained.

**ACTIVE** — the memory lifecycle state meaning current and retrievable.

**Alembic** — the schema migration tool used here; six migrations, `0001`–`0006`.

**Ambiguous failure** — one where you cannot tell whether the operation happened. See
UNKNOWN_OUTCOME.

**Append-only** — new rows are added; existing rows are not rewritten or removed.

**Approximate nearest neighbour (ANN)** — a vector search that finds probably-closest vectors much
faster than checking every one, trading a little accuracy for a lot of speed.

**asyncpg** — the async PostgreSQL driver used underneath SQLAlchemy.

**Atomic** — all-or-nothing.

**Attestation** — a record that a particular client asserted a particular fact.

**Audit event** — a durable row recording what happened to a memory, never its content.

**author_client / author_kind** — provenance columns on each revision: which client wrote it, and
whether a human confirmed it.

**Background worker** — code that runs separately from request handling, doing deferred work.

**BACKEND_BUSY** — the failure code for a connection-pool timeout. Safe to retry.

**BACKEND_UNAVAILABLE** — the failure code for an unreachable database. Safe to retry.

**Benchmark** — a repeatable measurement.

**CAS** — see compare-and-set.

**Candidate** — a row that survived filtering and is eligible for ranking.

**Client** — the program making requests. Here: Claude Desktop, Cursor, or a test harness.

**client_request_id** — the caller-generated idempotency key, 8–128 characters.

**Commit** — make a transaction's changes permanent and visible.

**Compare-and-set (CAS)** — write only if the current value still equals what you read, checked and
written in one atomic operation.

**Composite foreign key** — a foreign key spanning more than one column. Here it makes cross-project
supersession unrepresentable.

**Concurrency** — more than one thing happening in overlapping time.

**Connection pool** — a set of reusable open database connections. Here 10 plus 5 overflow.

**Constraint** — a rule the database itself enforces.

**Content hash** — SHA-256 of normalised content, backing deduplication.

**Context window** — the total amount of text a model can hold at once.

**Corroboration** — two or more distinct clients independently asserting the same fact.

**Cosine distance** — `1 - cosine similarity`, from 0 to 2; smaller is more similar. The `<=>`
operator.

**Cosine similarity** — a measure of the angle between two vectors, from -1 to 1.

**DEADLINE_EXCEEDED** — the failure code for a statement cancelled by `statement_timeout`. Not
usefully retryable.

**Deadlock** — two transactions each holding what the other needs. Prevented here by a consistent
lock order.

**Deduplication** — recognising that two independently submitted pieces of knowledge are the same.

**DELETED** — the lifecycle state set by `memory_forget`. A tombstone; content is retained.

**Dimension** — how many numbers are in a vector. Here 384.

**Docker Compose** — the tool defining the PostgreSQL service used for development and tests.

**Dual write** — writing to two systems without a transaction spanning them; the bug the outbox
prevents.

**Durable** — written so it survives a crash.

**Embedding** — a list of numbers representing text's meaning.

**EvalPlanQual** — PostgreSQL's re-check of an `UPDATE`'s `WHERE` clause against the newest committed
row version after being unblocked, under READ COMMITTED. The mechanism that makes CAS correct.

**Expired** — a memory past its `expires_at`. Derived at read time; **not** a stored status.

**expected_revision** — the caller's claim about which revision it read.

**fastembed** — the ONNX-based embedding runtime used for the local model.

**FOR SHARE** — a shared row lock that blocks until the owning transaction resolves. Used to wait for
an in-flight idempotency claim.

**FOR UPDATE** — an exclusive row lock taken by a `SELECT`.

**Foreign key** — a constraint requiring a column's value to exist in another table.

**Full-text search** — matching on words, using `tsvector`, a GIN index, and `ts_rank_cd`.

**GIN index** — Generalised Inverted Index; maps each value to the rows containing it.

**Greedy selection** — repeatedly taking the currently best-looking option.

**HNSW** — Hierarchical Navigable Small World; the graph-based ANN index type used here.

**Idempotency** — the property that doing something twice has the same effect as doing it once, and
the mechanism providing it.

**Immutable** — never changed after being written.

**Invariant** — a rule that must always be true.

**Jaccard similarity** — the size of the intersection divided by the size of the union of two sets.
Used here over content words, for diversity.

**JSON-RPC** — the JSON message format MCP uses.

**Knapsack problem** — choosing items with weights and values to maximise value under a weight limit.

**Latency** — how long an operation takes.

**Lexeme** — a normalised, stemmed word unit in PostgreSQL full-text search.

**Lexical search** — see full-text search.

**Lost update** — two writers read the same value and the second silently erases the first's change.

**Memory** — one durable, self-contained statement about a project, with an identity.

**Migration** — a versioned script that changes the schema.

**MMR (Maximal Marginal Relevance)** — selection balancing relevance against difference from what is
already selected. Lambda 0.7 here.

**MRR** — Mean Reciprocal Rank; the average of `1 / position of the first relevant result`.

**MVCC** — Multi-Version Concurrency Control; PostgreSQL's approach of writing new row versions
rather than overwriting.

**MCP** — Model Context Protocol; the standard by which an AI application discovers and calls tools
in a separate program.

**Namespace** — a boundary keeping data separate. Here, a project.

**nDCG@k** — Normalised Discounted Cumulative Gain; a ranking quality measure using graded relevance,
discounted by position, normalised against the ideal ordering.

**Nearest neighbour** — the stored vector closest to a query vector.

**Normalisation** — reducing text to a comparison form. Here NFKC, casefold, collapse whitespace,
strip trailing punctuation.

**Optimistic concurrency** — assume conflicts are rare; detect them at write time rather than
locking in advance.

**Outbox** — a table where follow-up work is recorded in the same transaction as the change it
relates to.

**Partial index** — an index covering only rows matching a condition.

**Persistent** — surviving beyond the program that created it.

**Pessimistic locking** — taking a lock before reading, so others wait.

**pgvector** — the PostgreSQL extension adding vector types, distance operators, and vector indexes.

**Precision@k** — the fraction of the top k results that are relevant.

**Primary key** — the unique identifier for a row.

**Project** — a memory namespace, identified by a server-issued UUID.

**Project isolation** — the guarantee that one project's memories never appear in another.

**Protocol** — agreed rules for exchanging messages.

**Provenance** — where a piece of information came from.

**Pydantic** — the validation library used for settings and tool output schemas.

**Race condition** — a bug whose outcome depends on the timing of concurrent operations.

**READ COMMITTED** — PostgreSQL's default isolation level; each statement sees only data committed
before that statement started.

**Recall@k** — the fraction of all relevant memories that appear in the top k.

**Resource (MCP)** — an identity-addressed, read-only thing a client can fetch by URI.

**Retry** — sending the same request again after not receiving an answer.

**Revision** — one version of a memory's text; a row in `memory_revisions`.

**Rollback** — discard a transaction's changes.

**RRF (Reciprocal Rank Fusion)** — combining ranked lists by position: `sum of 1/(k + rank)`, `k=60`.

**Savepoint** — a marker inside a transaction you can roll back to without abandoning the whole
transaction.

**Schema** — the structure of the database.

**Semantic search** — matching on meaning rather than exact words.

**Sequential scan** — reading a table row by row.

**SERIALIZABLE** — the strictest isolation level; conflicts raise `40001` and must be retried.

**SKIP LOCKED** — skip rows another transaction has locked rather than waiting.

**Source of truth** — the one place that decides the answer. Here, PostgreSQL.

**SQLAlchemy** — the Python toolkit used for models and query building.

**Stale client** — a client acting on a value that has since changed.

**Stale memory** — knowledge that was true once and is not current now.

**stale_inclusion_rate** — the fraction of queries where a forbidden memory appeared in the top k.
Target exactly 0.

**Stateless** — holding no state between requests that matters.

**stdio** — standard input and output, used here as the MCP transport.

**Stemming** — reducing a word to its root form.

**Subprocess** — a process started by another process.

**SUPERSEDED** — the lifecycle state of a memory retired by a different memory.

**Supersession** — one memory retiring another because the knowledge was replaced.

**Tombstone** — a marker saying something is gone while the data is retained.

**Token (model)** — the unit a language model reads text in.

**Token (text search)** — one indexed word unit.

**Token budget** — a limit on how many tokens a response may cost.

**Tool (MCP)** — a named operation the server offers, callable by the model.

**Transaction** — a group of statements treated as one unit.

**Transactional outbox** — the pattern of writing the follow-up job row in the same transaction as
the change.

**Transport** — the channel protocol messages travel over.

**tsquery** — a parsed search expression in PostgreSQL full-text search.

**tsvector** — a PostgreSQL type holding a document's lexemes and their positions.

**ts_rank_cd** — cover-density ranking; rewards query terms appearing near each other.

**Unique constraint** — a rule that no two rows may share a value.

**UNKNOWN_OUTCOME** — the failure code for a connection lost mid-flight, where the write may or may
not have committed. Not safe to retry blindly.

**Vector** — a list of numbers representing meaning.

**websearch_to_tsquery** — a lenient query parser that accepts what a search box accepts and never
raises a syntax error.

---

# CHEAT SHEETS

Twenty-three one-page summaries. Read one before an interview; read them all the night before.

---

## CHEAT SHEET 1 — What problem does the MCP Shared Memory Server solve?

**The situation.** You settle a decision with one AI coding client. A different client did not
participate in that interaction and does not automatically know it. Six months later the decision is
reversed, and now two contradictory statements exist.

**The easy half.** Give the tools one shared place to write things down.

**The hard half, and the actual project:**

| Problem | Why it is hard |
|---|---|
| Two clients write the same fact at once | they are separate OS processes sharing nothing |
| A decision is reversed | the reversal has to take effect everywhere, immediately |
| The old decision is still needed | someone will ask why it changed |
| Both versions are retrievable | the retired phrasing usually matches a query **better** |
| Keyword and meaning search each miss things | neither alone is enough |
| Results must fit a token budget | relevance ordering is not enough |
| Every client must reach it the same way | not a bespoke integration each |

**The one-sentence framing.** *The hard problem is not recall. It is truth maintenance in a
multi-writer store whose read path has a hard budget constraint.*

---

## CHEAT SHEET 2 — Overall architecture

```
 Claude Desktop            Cursor
       |                      |
       | MCP / stdio / JSON-RPC
       v                      v
 memhub process 1      memhub process 2      <- two OS processes,
   (stateless)           (stateless)             NO shared memory
       |                      |
       +----------+-----------+
                  v
          +----------------+
          |   PostgreSQL   |   <- the ONLY shared state
          |   + pgvector   |
          +----------------+
```

**Layers:** MCP handlers → services → repositories → PostgreSQL.
**Beside them:** domain (pure), retrieval (read-only), context (selection), embeddings (port +
worker), cli (operator only), observability.

**The single most important sentence:** the two server processes share nothing but PostgreSQL,
which is why every invariant is enforced in the database.

---

## CHEAT SHEET 3 — MCP terminology

| Term | One line |
|---|---|
| Protocol | agreed rules for messages |
| MCP | the standard for an AI app to discover and call tools in another program |
| Client | Claude Desktop, Cursor — decides *when* to call |
| Server | this program — answers |
| Tool | a named operation with declared input/output schemas |
| Tool arguments | **the entirety of what the server sees** |
| Resource | identity-addressed, read-only, fetched by URI |
| Transport | how the bytes travel |
| stdio | stdin/stdout — one subprocess per client |
| JSON-RPC | the message format: method, params, id → result or error |
| Protocol error | the request was malformed or the server broke |
| Tool error | the request was fine, the domain refused it |
| Domain outcome | neither — a legitimate result carrying `outcome` |

**Say this:** MCP is the interface, not the memory engine.

---

## CHEAT SHEET 4 — MCP tools

| Tool | Purpose | Key argument | Key output |
|---|---|---|---|
| `project_use` | resolve or create a namespace | `slug`, `create` | `project_id`, `created` |
| `memory_remember` | record knowledge, optionally retiring old | `supersedes`, `client_request_id` | `outcome`: created / deduplicated / idempotent_replay |
| `memory_revise` | update a memory you have read | `expected_revision` | `outcome`: revised / conflict / idempotent_replay |
| `memory_forget` | tombstone | `memory_id` | forgotten / already_forgotten |
| `memory_search` | retrieve current memories | `query`, `limit` | results + `filtered_out` |
| `memory_history` | the full record, including retired | `memory_id` | revisions, lineage, attestations, audit |
| `memory_context` | best brief within a token budget | `token_budget` | brief + budget report |

Plus three resources: `memory://projects`, `memory://memories/{id}`, `.../history`.

**Deliberately absent:** `memory_supersede` (folded into remember), any purge (operator CLI only).

---

## CHEAT SHEET 5 — Memory lifecycle

```
   memory_remember
        |
        v
     ACTIVE  ------ memory_revise ------> ACTIVE at revision N+1
        |                                  (old revision kept, is_current = false)
        |
        +-- memory_remember(supersedes=[me]) --> SUPERSEDED  (superseded_by_id set)
        |
        +-- memory_forget --------------------> DELETED      (deleted_at set)
        |
        +-- expires_at passes ---------------->  expired     (DERIVED, not stored)

   ACTIVE      -> appears in search
   SUPERSEDED  -> never in search, always in memory_history
   DELETED     -> never in search, always in memory_history
   expired     -> never in search, always in memory_history
```

**Only `memhub-admin purge` destroys content.** It is not reachable over MCP.

---

## CHEAT SHEET 6 — Revision vs supersession

| | Revision | Supersession |
|---|---|---|
| Question | "how did *this fact* change?" | "which fact replaced *that* fact?" |
| Cardinality | 1 memory → N revisions | N old → 1 new |
| `memories` rows | one | two or more |
| Tool | `memory_revise` | `memory_remember(supersedes=[...])` |
| Guarded by | CAS on `current_revision_no` | CAS on `status = 'ACTIVE'` |
| SQL | `cas_revise.sql` | `cas_supersede.sql` |
| Status after | still ACTIVE | old becomes SUPERSEDED |
| In search after | yes, at the new revision | old one: never again |
| Example | "…on Redis" → "…on Redis, with dedicated workers" | "…on Redis" retired by "…on PostgreSQL" |

**Why both:** forcing the replacement to be "revision 2" would mean lying about who wrote it and
when, and could not express one memory retiring three.

---

## CHEAT SHEET 7 — Concurrency and CAS

**The race:**
```
A reads rev 4.  B reads rev 4.  A writes -> rev 5.  B still expects 4.
Unprotected: B overwrites A silently. That is a LOST UPDATE.
```

**The fix — one statement:**
```sql
UPDATE memories SET current_revision_no = current_revision_no + 1, updated_at = now()
 WHERE id = :id AND project_id = :pid
   AND current_revision_no = :expected AND status = 'ACTIVE'
RETURNING current_revision_no;
```
Zero rows = conflict.

**Why it is airtight:** the `UPDATE` takes a row lock; when the loser unblocks, PostgreSQL walks to
the newest committed row version and re-evaluates the `WHERE` against it (**EvalPlanQual**, a READ
COMMITTED behaviour). The read and the write are the same statement.

**Why READ COMMITTED not SERIALIZABLE:** SERIALIZABLE raises `40001` and forces retries — 49 retry
storms instead of 49 clean, informative conflicts.

**Proof:** 50 writers from a barrier → exactly 1 success, 49 conflicts. Plus a fixture asserting the
pool supplies 50 distinct backends.

**Backstop:** `PK (memory_id, revision_no)` and `UNIQUE (memory_id) WHERE is_current`.

---

## CHEAT SHEET 8 — Idempotency vs deduplication

| | Idempotency | Deduplication |
|---|---|---|
| Trigger | **one** client retries **the same request** | **two** clients assert **the same fact** |
| Key | `client_request_id` | SHA-256 of normalised content |
| Table | `idempotency_keys` | `memory_dedup_keys` |
| Answer | replay the stored response | return the existing memory + attestation |
| Outcome | `idempotent_replay` | `deduplicated` |
| Without it | a network retry duplicates a memory | the corpus fills with near-copies |

**Protocol:** `INSERT ... ON CONFLICT DO NOTHING` to claim → if zero rows, `SELECT ... FOR SHARE`
blocks until the owner resolves → row present means replay; row absent means they rolled back, so
loop (max 3).

**The sharp insight:** `revise` does not *need* a key for correctness — CAS already prevents the
duplicate write. The key exists so a retry of an already-applied revise replays instead of reporting
a conflict against yourself. That is why the claim happens **before** the CAS.

---

## CHEAT SHEET 9 — Transactions and constraints

**One tool call = one transaction** (`session_scope`). For a superseding write that means: the
idempotency claim, memory, revision, dedup key, supersession updates, dedup releases, outbox row,
attestation, audit rows, and idempotency completion — all visible at the same instant, or none.

**One savepoint**, around the speculative insert for deduplication.

**Nine of fourteen invariants are schema-level:**

| Invariant | Enforced by |
|---|---|
| one current revision per memory | `UNIQUE (memory_id) WHERE is_current` |
| unique revision numbers | `PK (memory_id, revision_no)` |
| supersession stays in-project | composite FK `(superseded_by_id, project_id)` |
| no self-supersession | `CHECK (superseded_by_id IS DISTINCT FROM id)` |
| status/timestamp agreement | paired `CHECK`s |
| one write per idempotency key | `PK (project_id, client_request_id)` |
| no duplicate active content | `memory_dedup_keys` PK |
| revision belongs to same project | composite FK |
| every TASK has an expiry | `CHECK (type <> 'TASK' OR expires_at IS NOT NULL)` |

**The pattern:** the constraint is the guarantee; the Python validator is the good error message.

---

## CHEAT SHEET 10 — FTS vs semantic search

| | Full text | Semantic |
|---|---|---|
| Matches | words | meaning |
| Storage | `content_tsv` generated column | `vector(384)` |
| Index | GIN, partial `WHERE is_current` | HNSW, `vector_cosine_ops` |
| Query | `websearch_to_tsquery('english', q)` | embed the query, then `<=>` |
| Score | `ts_rank_cd(..., 32)` — unbounded | cosine distance 0–2, smaller better |
| Wins on | identifiers, rare words, negation, knowing there is no answer | different wording for the same idea |
| Loses on | `JWTs` vs `jwt` (0.000 → 1.000 with semantic) | precision; returns k results whether or not anything is close |
| Guard | any-term fallback when strict matching finds nothing | `MAX_COSINE_DISTANCE = 0.35` |

---

## CHEAT SHEET 11 — Embeddings and pgvector

- Model: `BAAI/bge-small-en-v1.5` via **fastembed** (ONNX, CPU, ~50 MB). Optional extra; default
  adapter is `none`. Loaded lazily.
- 384 dimensions, unit-normalised, fixed by the column type.
- Own table `memory_embeddings`, keyed `(memory_id, revision_no, model)`.
- Model in the key so vectors from different models never get compared.
- HNSW index with `vector_cosine_ops`; `SET LOCAL hnsw.iterative_scan = relaxed_order` per query to
  fix filtered-ANN under-return.
- Distance threshold **0.35**, chosen by a committed sweep. Without it: nDCG 0.881 but precision
  0.113.
- A deterministic hash fake exists for CI. It carries **no semantic signal** and cannot measure
  quality — deliberately.

---

## CHEAT SHEET 12 — RRF

**Problem:** `ts_rank_cd` is unbounded; cosine distance is 0–2 and points the other way. Adding them
is meaningless. Per-query min-max normalisation is worse — it makes a mediocre best match look
identical to a perfect one.

**Solution:** discard the scores, use only positions.

```
score(d) = sum over retrievers of  1 / (k + rank(d)),   k = 60
```

**Worked:** lexical `[A, C, B]`, semantic `[B, A, D]` →
A = 1/61 + 1/62 = 0.0325; B = 1/63 + 1/61 = 0.0323; C = 1/62 = 0.0161; D = 1/63 = 0.0159.
Order: **A, B, C, D** — agreement between retrievers beats a single first place.

**Properties:** scale-free; no per-query normalisation; robust to one retriever returning nothing (a
missing document just contributes no term); rewards agreement.

**Why k=60:** without it, rank 1 is worth twice rank 2, overweighting one retriever's confident
mistake.

---

## CHEAT SHEET 13 — The stale-memory guarantee

**The trap.** Query "redis". The retired memory is short and about Redis. The current one is longer
and mentions Redis only to say it was removed. **The retired one wins on both lexical and semantic
relevance.**

**The distinction.** *Relevance is not the same thing as current truth.*

**The weak fix:** rank it lower. Leaks at a higher limit; the penalty must beat an unbounded score;
couples correctness to tuning; cannot be stated as a guarantee.

**This project's fix:** exclude it from the candidate set *before* ranking, in one function every
retrieval path composes on:

```sql
WHERE m.project_id = :pid
  AND m.status = 'ACTIVE'
  AND (m.expires_at IS NULL OR m.expires_at > now())
  AND r.is_current
```

**Proof:** absent at **every** limit from 1 to 100 and **every** query, and at every token budget.
Measured `stale_inclusion_rate = 0.000` across all three retrieval strategies. Mutation-tested:
removing the status condition fails five tests.

---

## CHEAT SHEET 14 — Background jobs

**Why not inline:** holds a row lock across slow, fallible inference; a model outage becomes a write
outage.
**Why not after the commit:** the dual-write bug — crash in between and the memory exists with no
vector and nothing knows.
**So:** transactional outbox. The `embedding_jobs` row is inserted **in the same transaction** as the
revision.

**The claim:**
```sql
SELECT ... FROM embedding_jobs j JOIN memory_revisions r ON ...
 WHERE j.state='PENDING' AND j.model=:model AND j.next_attempt_at <= now()
 ORDER BY j.next_attempt_at
   FOR UPDATE OF j SKIP LOCKED
 LIMIT :batch_size
```

- **SKIP LOCKED** — workers take disjoint batches instead of blocking on each other.
- **OF j** — lock the job only, so embedding does not contend with revision writes.
- Batch of 16, one transaction: claim → embed → store → mark DONE.
- Failure: exponential backoff from 2s, `DEAD` after 5 attempts, error stored.
- Crash mid-batch: the transaction aborts, locks release, jobs return to PENDING.
- Consistency: **eventual, with an explicit queryable pending state** — `semantic_coverage` reports
  it.

---

## CHEAT SHEET 15 — Token-budgeted context

**Pipeline:** over-fetch 60 candidates → hold back a 10% margin → drop anything too large alone →
per-type quotas (CONSTRAINT 25%, DECISION 40%, FACT 20%, TASK 15%) → greedy fill by score-per-token
with MMR novelty (λ=0.7) → reject anything ≥0.6 Jaccard-similar → redistribute the unspent → stable
order → render.

**The estimator.** `ceil(len(text)/3.2) + 12`, then a 10% margin.
**The contract.** *Never exceed the budget. May under-fill by roughly 10%.*
**The calibration story.** 3.6 under-estimated 2 of 33 real memories, because identifiers tokenise
densely — the worst sample was 3.25 chars/token. Corrected to 3.2: zero under-counts, worst case
4.8% over, at the cost of ~32% average over-estimation.

**The report.** `requested`, `estimated_used`, `utilisation`, `estimator`, `considered`, `selected`,
`dropped: {too_similar, no_budget_left, too_large_alone}`.

---

## CHEAT SHEET 16 — Failure handling

| Code | Retry? | Because |
|---|---|---|
| `BACKEND_UNAVAILABLE` | yes | the connection never opened |
| `BACKEND_BUSY` | yes | the pool timed out; nothing was sent |
| `UNKNOWN_OUTCOME` | **no** | the connection died mid-flight; it may have committed |
| `DEADLINE_EXCEEDED` | no | the query was too slow; the server is healthy |

**The branch order is load-bearing:** check `connection_invalidated` **before** the SQLSTATE. In a
real mid-flight drop both signals are present, and `08006` alone would say "safe to retry", which is
how duplicate writes happen. Mutation-tested: inverting it fails three tests.

**UNKNOWN_OUTCOME's answer:** replay the idempotency key, or re-read. That is the payoff of having
idempotency at all.

**Degradation:** an embedder failure or a cancelled semantic query degrades to lexical-only with
`degraded: "lexical_only: ..."`. Only `57014` degrades; other database errors re-raise.

**Timeouts:** `pool_timeout` 2 s (fail fast, never queue unboundedly); `statement_timeout` 5000 ms,
server-side.

**Schema drift:** refuse to start in **both** directions. The ahead direction is the dangerous one.

---

## CHEAT SHEET 17 — Testing

| Category | Count | Proves |
|---|---|---|
| `tests/unit/` | 133 | domain logic in isolation, no database |
| `tests/integration/` | 109 | real behaviour against real PostgreSQL |
| `tests/protocol/` | 36 | the MCP contract, plus a real subprocess and a golden manifest |
| `tests/failure/` | 19 | driver failures, partial writes, schema drift, purge |
| `tests/concurrency/` | 12 | 50 real writers, exactly 1 winner |
| `tests/eval/` | 7 | retrieval quality against a graded corpus, gated |
| `tests/perf/` | 3 | latency and cost-vs-corpus, deselected by default |

**319 test functions, 30 modules.** No mocked database anywhere.

**Harness:** migrations once into a template database; each module clones it with
`CREATE DATABASE ... TEMPLATE`; each test rolls back — except concurrency tests, which cannot.

**Three things worth naming:** the pool-size fixture that makes the 50-way test honest; the
mutation experiments; and the golden manifest, because tool descriptions are the prompt.

---

## CHEAT SHEET 18 — Retrieval metrics

| Metric | Meaning | Value here |
|---|---|---|
| Precision@10 | fraction of returned results that are relevant | 0.691 FTS, 0.671 hybrid |
| Recall@10 | fraction of relevant memories that appeared | 0.817 → 0.828 |
| nDCG@10 | graded, position-discounted, normalised ranking quality | 0.802 → **0.853** |
| MRR | 1 / position of the first relevant result | 0.823 → 0.914 |
| **stale_inclusion_rate** | fraction of queries returning a forbidden memory | **0.000** everywhere |
| empty_for_unanswerable | unanswerable queries correctly returning nothing | 0.667 → 0.333 |

**Dataset:** 200 memories, 34 queries (31 answerable), two projects, judgments written before any
strategy ran, gated against a committed baseline with 0.02 tolerance.

**What it cannot prove:** small corpus, synthetic, single-author judgments, 34 queries is a small
sample, hybrid measured locally once. Say this before you are asked.

---

## CHEAT SHEET 19 — Tech stack

| Technology | Why here |
|---|---|
| Python 3.12 | `StrEnum`, PEP 695 generics, `match` |
| MCP SDK v2 | tools, resources, stdio, schema from annotations |
| PostgreSQL 16 | every guarantee is a PostgreSQL feature |
| pgvector | vectors **inside** the same transaction as the filter |
| SQLAlchemy 2 async | typed models, composable queries, drift detection |
| asyncpg | async driver |
| Alembic | six versioned migrations + a drift test |
| fastembed + bge-small | local, private, ONNX, ~50 MB |
| Pydantic / pydantic-settings | validated settings, declared output schemas |
| Docker Compose | reproducible PostgreSQL + pgvector on port 5435 |
| pytest / pytest-asyncio | 319 tests, real database |
| ruff + mypy strict | including `T20`, which bans `print()` because stdout is the protocol |

**Three comparisons to have ready:** PostgreSQL vs SQLite (multi-process writers, and the features
being demonstrated); pgvector vs a vector database (transactions and truth maintenance);
PostgreSQL vs Redis for jobs (the outbox needs one transaction).

---

## CHEAT SHEET 20 — The four resume bullets

| Bullet | Verdict | The one thing to add when speaking |
|---|---|---|
| 1. MCP shared memory, isolation, provenance, immutable revisions, supersession | **accurate** | "Two processes sharing nothing but PostgreSQL — that is why the concurrency work is real" |
| 2. Safe concurrent updates, versioned memory, idempotent writes | **accurate** | name **both** duplicate mechanisms: idempotency for retries, content hash for two clients |
| 3. Hybrid FTS + pgvector, context filtering | **accurate**, one wording fix | say "a structural filter applied **before ranking**", not "context filtering" |
| 4. Background embeddings, retry/failure handling, token budget, migrations, tests | **accurate** | retry applies to **embedding jobs**; database failures are *classified*, not auto-retried |

**Numbers to attach:** 1 winner and 49 conflicts deterministically · nDCG 0.803 → 0.853 ·
stale-memory return rate exactly 0.000 · 100× corpus, 1.9× query time.

---

## CHEAT SHEET 21 — Strongest engineering decisions

1. **Structural exclusion over ranking-based suppression.** A retired memory leaves the candidate
   set before ranking, so no scoring function — today's or a future one — can resurrect it.
2. **Optimistic concurrency with a real correctness argument.** Not "I added a version column" but
   "I chose READ COMMITTED *because* EvalPlanQual re-checks the predicate against the updated row,
   which turns a lost-update race into a deterministic 0-row result".
3. **Invariants in the schema, not the application.** Nine of fourteen, including a composite foreign
   key that makes cross-project supersession unrepresentable.
4. **Revision and supersession as separate relations.** Different cardinalities, different
   authorship, different guards.
5. **Transactional outbox for embeddings.** With an explicit argument for why enrichment must not sit
   inside the write transaction.
6. **RRF over score blending.** Because the two scales are genuinely incomparable and per-query
   normalisation is worse than doing nothing.
7. **The evaluation harness before the vectors.** Build the ruler first — and it found a real defect
   on its first run.
8. **Measuring the threshold rather than guessing it**, and publishing the row that shows what it
   costs.
9. **Failure classification that distinguishes safe-to-retry from genuinely ambiguous.**
10. **Mutation testing**, which found a vacuous test that had been passing for the wrong reason.

---

## CHEAT SHEET 22 — Current limitations

| Gap | Why out of scope | When it matters |
|---|---|---|
| No authentication or authorization | single-user local V1 | the moment a server has untrusted callers |
| No metrics exporter | pull-scraping cannot find a short-lived subprocess | any deployment with an on-call rotation |
| No tracing | scoped, not built | debugging latency across layers |
| stdio only | it is what the clients use locally | many concurrent connections; hosted use |
| Retention is manual | `gc` exists; nothing schedules it | any long-running deployment |
| **No secret screening** | specified in the architecture, **not implemented** | immediately — the only control today is a tool description |
| Search does not report coverage or degradation over MCP | fields exist on the domain object, not the schema | when a caller needs to know the search was partial |
| Priors are not applied to fused results; no attestation prior; no `why` block | simplification | ranking explainability |
| Two tool descriptions are stale | drift | now — one tells the model supersession is unavailable |
| Ranking weights untuned | deliberately not fitted before measuring | any ranking work |
| No HA, backups, or replication | local developer tool | production |

---

## CHEAT SHEET 23 — Top interview questions

**Ten to rehearse out loud:**

1. What problem does this solve, in one sentence? → *Truth maintenance in a multi-writer store whose
   read path has a hard budget constraint.*
2. Does your server read conversations? → *No. It receives tool-call arguments and nothing else.*
3. Where does the shared state live, and why? → *PostgreSQL, because two separate OS processes share
   nothing else and it is the only component that can see both writers.*
4. What is a lost update and how do you prevent it? → *Compare-and-set in one statement; zero rows
   means you lost, and you get the winning revision back.*
5. Why READ COMMITTED? → *For its conflict semantics. SERIALIZABLE turns a clean refusal into a retry
   storm.*
6. Revision vs supersession? → *Same fact refined vs a different fact retiring it. Different
   cardinalities, different authorship.*
7. Why not rank stale memories lower? → *It leaks at a higher limit, the penalty must beat an
   unbounded score, and it couples correctness to tuning.*
8. Why hybrid, and why RRF? → *The two retrievers fail on different queries; their scores are
   incomparable, so fuse by position.*
9. What is UNKNOWN_OUTCOME? → *The connection died mid-flight, so the write may or may not have
   committed. Replay the idempotency key or re-read; do not retry blindly.*
10. What would you improve? → *Auth, a metrics exporter, scheduled retention, and the secret
    screening the architecture specifies but the code does not implement.*

**Three where being honest scores highest:**

- *"What does your 50-writer test not prove?"* — it is 50 concurrent callers, not 50 MCP clients; it
  is a correctness test, not a benchmark.
- *"How good is your evaluation, really?"* — 200 synthetic memories, single-author judgments, 34
  queries, hybrid measured locally once. It demonstrates a direction, not a statistically robust
  delta.
- *"What is the biggest gap between your design and your code?"* — secret screening, which is fully
  specified and entirely absent.

---

## Final verification note

Everything in this document was checked against the repository at the current commit. Three classes
of claim are flagged where they appear:

1. **Verified in code** — the default. File paths and function names are given.
2. **Documentation-only claims** — for example the mutation-test failure counts (5, 3), which are
   reported in `README.md` and `docs/failure-modes.md` as the results of manual experiments. The
   repository contains no automated mutation runner, so I could not independently reproduce those
   counts. The assertions those experiments protect are real and readable in the tests.
3. **Could not be verified from the repository** — the hybrid retrieval numbers were produced by a
   test marked `real_embeddings` that is deselected by default and requires a model download, so I
   could not re-run it; and I could not run `pytest` at all during this review, because it requires
   a live PostgreSQL instance. Test counts here are static counts of `def test_` definitions.
