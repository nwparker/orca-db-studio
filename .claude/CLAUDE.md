# DB Studio for Orca — Agent Instructions

## Role & Responsibilities

Your role is to analyze user requirements, delegate tasks to appropriate sub-agents, and ensure cohesive delivery of features that meet the PRD and the system architecture.

## Project Overview

DB Studio is an Orca plugin for working with PostgreSQL, MySQL/MariaDB and SQLite from an editor tab, with safe, logged database access for Orca's agents over MCP.

**Every database call goes through the worker's policy layer** — the UI, the gateway and the MCP server never touch a driver directly.

## Conventions worth knowing

- `src/drivers/` is the only place that knows which engine is behind a connection. Code outside it must not branch on engine type — add a method to the driver interface instead.
- `mcp/` is a plain Node stdio server with zero dependencies. Keep it that way.
- UI code lives in `src/editor/` (grid, SQL editor, schema page) and `src/panel/` (connections, approvals, activity). Promote to a shared folder only when both actually need it.

## Global conventions

- Prefer named function declarations over arrow function assignments
- Prefer `interface` for object shapes and component props
- Use `type` for unions, intersections, and utility types
- Never use `any`; use `unknown` when the type cannot be determined
- **Biome** is the single tool for formatting and linting — do not use ESLint or Prettier
- Follow project docs in `docs/` and existing local patterns.
- Prefer YAGNI, KISS, and DRY in that order.
- Implement real behavior. Do not add fake data, mocks, or temporary shortcuts just to satisfy a check.
- Keep changes scoped to the request and the affected contracts.
- Use descriptive kebab-case file names for new files.
- Split code only when it reduces real complexity or matches existing module boundaries.

## Global Rules

- After completing any task, verify with `npx tsc --noEmit`, `orca plugin validate .`, then lint and format (`npm run lint`, `npm run format`).
- No native modules or binaries — the Orca directory rejects them. Drivers must be pure JS, bundled into `vendor/`; SQLite uses `node:sqlite`.
- An installed plugin cannot carry `node_modules`. Anything the worker needs at runtime must be bundled.
- Credentials live only in `secrets:own`. Never log them, never write them to `storage:own`, never pass them to agents through env vars or prompts.
- Read-only must be enforced by the database session (`SET TRANSACTION READ ONLY`, `START TRANSACTION READ ONLY`, read-only SQLite connection). Parsing SQL is a UX hint, never the security boundary.
- Every query, from the UI or an agent, is written to the activity log.
- Never load a full result set into memory — use cursors / streaming and page through the driver interface.
- Reference requirement IDs from `docs/PRD.md` (`C1`, `G1`, `A2`, ...) in PR descriptions when a change implements or changes one.
- Keep commits focused and use conventional commit format without AI references.
- Never commit secrets, dotenv files, tokens, private keys, database credentials, or personal data.

## Quality Gates

- Run the narrowest useful test first, then broaden when shared behavior or public contracts changed.
- Driver changes run the integration tests against all affected engines (Testcontainers), not just one.
- Policy changes must keep the read-only bypass tests passing (writes hidden in CTEs, functions, multi-statement batches).
- UI changes run `orca plugin test .` and `orca plugin check .`; use `orca plugin shot <key>` to look at the result.
- Do not hide failing tests, lint, type, build, validation, or syntax errors.
- Preserve public contracts (`protocol.ts`, the driver interface, MCP tool schemas, manifest settings) unless the change intentionally updates them and the user accepted that scope.

## Dev loop

```
orca plugin dev watch .     # load as dev plugin, reload on save
orca plugin types .         # regenerate .orca/types
orca plugin test .          # *.test.ts(x) with orca/testing
orca plugin check .         # render check scenarios
orca plugin validate .      # manifest and entries, same check as the directory
```

## Documentation

The `docs/` folder contains the product scope, the architecture, and decisions that explain the **why** behind non-obvious choices. Read relevant docs before working on a related area, and add an ADR when you make a decision that would otherwise surprise a future reader.

| File                          | Topic                                                                                                                                                              |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `docs/PRD.md`                 | What DB Studio is, who it is for, use cases, functional requirements (`C-`, `E-`, `Q-`, `G-`, `D-`, `S-`, `A-`, `X-`), non-functional requirements, open questions |
| `docs/system-architecture.md` | Components, key decisions, driver interface, security model, repo layout, testing                                                                                  |
| `docs/adr/`                   | One file per decision (`NNNN-kebab-case-title.md`)                                                                                                                 |
