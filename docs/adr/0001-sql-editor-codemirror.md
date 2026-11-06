# 0001. SQL editor: CodeMirror 6, not Monaco

Status: accepted

## Context

The SQL editor needs a code editor component inside the plugin frame.

Orca 1.4.248 does not expose it. Monaco appears only in Orca's own renderer (`renderer/assets`) and in its file-language detection. No plugin API, manifest field or frame provides it. A custom editor runs in an isolated guest frame (`frame: "isolated"`) with `default-src 'self'` and only the plugin's own files, so it cannot reuse Orca's copy.

The only Monaco option would be to bundle our own: several MB plus separate worker files, against a 1 MiB limit per source module and a 32 MiB limit per frame asset, and it would make `vendor/` heavy for the directory checks.

## Decision

Use CodeMirror 6 with `@codemirror/lang-sql` dialects (PostgreSQL, MySQL, MariaDB, SQLite). It is modular and small, needs no workers, and ships as plain ESM that fits the frame CSP.

The editor is bundled into `vendor/` with esbuild, like the drivers, because an installed plugin cannot carry `node_modules`.

## Consequences

- Autocomplete (Q2) is built on `@codemirror/autocomplete` fed by the live schema from the worker.
- Reopen this decision only if Orca starts exposing its Monaco instance to plugins.
