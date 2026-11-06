# PRD: DB Studio for Orca (PostgreSQL · MySQL · SQLite)

## Summary

DB Studio is an Orca plugin for working with PostgreSQL, MySQL/MariaDB and SQLite from an editor tab: browse the schema, write SQL with autocomplete, scroll through millions of rows without lag, and give Orca's agents safe, visible access to the database.

## Problem

Today there is no good way to work with a server database inside Orca.

- **The official plugin only covers SQLite.** The official SQLite Browser opens `.db`/`.sqlite` files in an editor tab and is read-only. It cannot connect to a database server.
- **Other plugins are early.** Community attempts exist, but they live in a small side panel, cap results at a few hundred rows, and lack basics like autocomplete or cell editing.
- **The good experience lives on other platforms.** Mature database tooling exists in other editors and standalone clients, but it is usually tied to a single engine or a single cloud, and it knows nothing about Orca's agents or worktrees.

The result: Orca users keep TablePlus, DBeaver or `psql` open on the side, and agents work against the database with no visibility or control for the person.

## Goals

1. **Replace the external client** for 80% of daily work: connect, browse, query, edit individual rows.
2. **A grid with no practical limit:** scroll and filter over millions of rows on all three engines.
3. **Safe, visible agent access:** Orca's agents query the database over MCP, read-only by default, and every query is logged.
4. **One model for three engines:** same UI and features everywhere; engine differences stay inside the driver.
5. **Publishable in the Orca plugin directory.**

## Non-goals (for now)

- Server administration: creating instances, users, backups, replicas.
- Cloud console integrations (Azure, AWS RDS, etc.) beyond reading credentials.
- Schema migrations or diffs between databases.
- Non-relational engines (MongoDB, Redis) and other SQL engines (SQL Server, Oracle).
- A built-in "chat with your database": Orca's agents cover that through MCP.

## Users and use cases

The primary user is a developer who runs several agents in parallel in Orca, each in its own worktree, and needs to see what is happening in the database while they work.

| Use case                                                                | What they need                                                        |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Check what an agent left in the database after a migration or seed      | Open the table instantly, filter it, see the updated schema           |
| Debug an issue with real data in dev or staging                         | Fast queries with autocomplete, edit a single row, copy results       |
| Let Claude Code query the database to write a query or an endpoint      | Read-only MCP access, no credentials pasted into prompts              |
| Let an agent fix data in dev                                            | Writes with explicit approval and a log of what ran                   |
| Each worktree points at its own database (Docker, Neon/Supabase branch) | Connection read from the worktree's `.env`, not one global connection |
| Investigate a slow query                                                | Readable `EXPLAIN ANALYZE` without leaving Orca                       |

Per-worktree connections are the key differentiator: no existing tool understands that each Orca workspace can have its own database.

## Functional requirements

P0 is the MVP, P1 comes next, P2 later. Every requirement applies to all three engines unless stated otherwise.

| ID  | Area         | Requirement                                                                                                            | Priority |
| --- | ------------ | ---------------------------------------------------------------------------------------------------------------------- | -------- |
| C1  | Connections  | Create, edit, test and delete PostgreSQL, MySQL/MariaDB and SQLite connections; passwords only in Orca's secrets store | P0       |
| C2  | Connections  | Paste a connection string (`postgres://`, `mysql://`) and prefill the form                                             | P0       |
| C3  | Connections  | Open a `.db`/`.sqlite` file as a file editor                                                                           | P0       |
| C4  | Connections  | Per-worktree connection: detect `DATABASE_URL` (or a configurable variable) in the worktree's `.env` and offer it      | P0       |
| C5  | Connections  | Environment label (`local`, `dev`, `staging`, `prod`) shown as a colored tab; `prod` is read-only by default           | P0       |
| C6  | Connections  | SSL/TLS modes and certificates; SSH tunnel                                                                             | P1       |
| C7  | Connections  | Credentials from AWS Secrets Manager / RDS IAM auth                                                                    | P2       |
| E1  | Explorer     | Tree: connection → database → schema → tables, views, functions, indexes, triggers, sequences                          | P0       |
| E2  | Explorer     | Search objects by name and type without expanding the tree                                                             | P1       |
| E3  | Explorer     | Refresh schema; notify when an agent changes the schema (DDL in the activity log)                                      | P1       |
| Q1  | SQL editor   | Query tab with dialect-aware highlighting; run the statement under the cursor or the selection                         | P0       |
| Q2  | SQL editor   | Autocomplete for tables, columns and keywords from the live schema                                                     | P1       |
| Q3  | SQL editor   | Cancel a running query; configurable timeout                                                                           | P0       |
| Q4  | SQL editor   | Per-connection query history; saved queries as `.sql` files in the repo                                                | P1       |
| Q5  | SQL editor   | Multiple result sets when running several statements                                                                   | P1       |
| G1  | Grid         | Virtualized, paged against the server (keyset or offset); no fixed row cap                                             | P0       |
| G2  | Grid         | Sort and filter by column, applied on the server (`ORDER BY` / `WHERE`)                                                | P0       |
| G3  | Grid         | Panel with the full cell value; formatted JSON/JSONB                                                                   | P0       |
| G4  | Grid         | Copy cells, rows or results as CSV, JSON, Markdown or `INSERT`                                                         | P0       |
| G5  | Grid         | Export the full result to CSV/JSON via streaming                                                                       | P1       |
| D1  | Data editing | Edit cells, add and delete rows on tables with a primary key; changes stay as a draft until "Apply"                    | P1       |
| D2  | Data editing | Preview the SQL to be executed; run it inside a transaction                                                            | P1       |
| S1  | Schema       | Schema page with the DDL of every object                                                                               | P0       |
| S2  | Schema       | ER diagram with foreign keys                                                                                           | P2       |
| S3  | Schema       | `EXPLAIN` / `EXPLAIN ANALYZE` visualizer (tree with costs and timings)                                                 | P2       |
| A1  | Agents       | MCP server with `list_connections`, `describe_schema`, `run_query`, injected when Orca launches agents                 | P0       |
| A2  | Agents       | Agent access per connection: `off`, `read-only` (default) or `read-write`                                              | P0       |
| A3  | Agents       | Agent writes wait for approval in the UI, with a notification                                                          | P1       |
| A4  | Agents       | Activity log: SQL, origin (person or agent, and worktree), duration, rows, outcome                                     | P0       |
| A5  | Agents       | Open the result of an agent's query in the grid                                                                        | P1       |
| X1  | General      | Commands in Orca's palette: new query, open table, switch connection                                                   | P0       |
| X2  | General      | Open `psql` / `mysql` in an Orca terminal with the active connection                                                   | P2       |

## Non-functional requirements

**Performance**

- First page of a table in under 500 ms against a local database; smooth scrolling with 1M rows.
- Never load a full result into memory: page with server-side cursors / streaming.
- The UI never blocks during a query.

**Security**

- Passwords and tokens only in Orca's secrets store; never in logs, plain storage or agent environment variables.
- Agent read-only mode is enforced by the database session, not only by parsing SQL.
- Human writes on `prod` connections require a second confirmation.

**Compatibility**

- PostgreSQL 13+, MySQL 8+, MariaDB 10.6+, SQLite 3.40+.
- macOS, Windows and Linux, same as Orca.

**Quality**

- Driver tests against real databases in CI.
- MIT license, with third-party notices for bundled drivers.

## Open questions

- Can one plugin declare both a file editor (`.db`) and a non-file editor (a table or a query)?
- Does the `agent.launch.before` hook know the agent's worktree, so we can inject that worktree's connection?
- Final name and publisher: personal repo or an organization?
