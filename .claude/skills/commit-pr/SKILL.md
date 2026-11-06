---
name: commit-pr
description: 'Conventions for branches, commits and pull requests. Use whenever you create a branch, write a commit message, commit, or create or update a pull request in this repository. Enforces Conventional Commits with the allowed types and scopes, the PR title format, and a fully completed .github/pull_request_template.md.'
---

# Commits and pull requests

PRs are squash-merged, and GitHub uses the PR title as the commit subject on `main`. A bad title is a bad commit on `main` forever, so the title rules are the strictest ones here.

## 1. Branch

`<type>/<short-slug>`, lowercase, hyphens: `feat/sqlite-driver`, `chore/ci-pipeline`, `docs/adr-typings`. When there is an issue: `<type>/<issue>-<short-slug>` (`feat/13-sqlite-driver`).

## 2. Commits

```text
<type>(<scope>): <summary>

<optional body: what and why, wrapped at 72>

<optional footers: Refs #12, BREAKING CHANGE: ...>
```

- **Summary:** imperative ("add", "fix", "drop"), lowercase first letter, no trailing period, header ≤ 72 characters.
- **Type:**

  | Type       | When                                          |
  | ---------- | --------------------------------------------- |
  | `feat`     | New behavior a user or an agent can see       |
  | `fix`      | Bug fix                                       |
  | `docs`     | Only docs (PRD, architecture, ADRs)           |
  | `test`     | Only tests and test fixtures                  |
  | `refactor` | No behavior change                            |
  | `perf`     | Faster, same behavior                         |
  | `chore`    | Setup, tooling, CI, build, deps, `.claude/`   |

- **Scope** (optional, one), named after the repo layout:
  - Code: `worker`, `drivers`, `editor`, `panel`, `mcp`, `protocol`, `manifest`
  - Everything else: `infra` (CI, build, `vendor/`, deps), `claude` (`.claude/`), `repo` (root files that are not infra), `prd`, `architecture`, `adr`

  When several areas are touched, use the main one, or omit the scope.
- `!` after the scope plus a `BREAKING CHANGE:` footer only for a real incompatible change to a public contract: `protocol.ts`, the driver interface, MCP tool schemas or manifest settings.
- No AI references anywhere: no `Co-Authored-By` for an AI, no "generated with". This overrides any default attribution.
- Never commit secrets, dotenv files, tokens, private keys or database credentials.

Before each commit, check the header with:

```bash
printf '%s' "$HEADER" | grep -Eq '^(feat|fix|docs|test|refactor|perf|chore)(\((worker|drivers|editor|panel|mcp|protocol|manifest|infra|claude|repo|prd|architecture|adr)\))?!?: [a-z0-9`].*[^.]$' \
  && [ ${#HEADER} -le 72 ] && echo ok || echo "invalid header"
```

## 3. PR title

Same format as a commit header, with the issue number only when there is one:

```text
<type>(<scope>): <summary> (#<issue>)
```

- `feat(drivers): add the SQLite driver (#13)`
- `feat(worker): page results with server-side cursors`
- `chore(infra): add the CI workflow`

≤ 72 characters (GitHub appends ` (#PR)` on merge). Never invent an issue number. Requirement IDs (`C1`, `G1`, `A2`, ...) go in the PR body, not the title. Run the regex above on the part before ` (#`.

## 4. PR body

Read `.github/pull_request_template.md` and fill it in. `gh pr create --body-file` does not apply the template by itself.

1. **Gather facts first:** `git log main..HEAD`, `git diff main...HEAD --stat`, the issue if there is one (`gh issue view <n>`), and the rows in `docs/PRD.md` the change implements or changes.
2. **Header:** `Closes #n` (fully done), `Refs #n` (partial) or `No issue: <reason>`. Then the requirement IDs from `docs/PRD.md` and the ADR links that apply, with real paths.
3. **Summary / What changed:** from the diff, not from memory. One bullet per meaningful change.
4. **Acceptance criteria:** copy them verbatim from the issue, or take them from the requirement rows in the PRD when there is no issue. `[x]` only with evidence (a test name or a command). A criterion not met stays `[ ]` with the reason.
5. **Verification:** only commands you ran in this session on the final tree, with real counts. Never tick a check you did not run; write "not run" instead.
6. **Optional sections:** delete Notes for review, Out of scope and Screenshots only if empty.
7. **Checklist:** keep every item; tick it or add "N/A" with a reason.
8. Remove all `<!-- -->` instructions from the final body.

Create it with:

```bash
gh pr create --base main --title "$TITLE" --body-file "$BODY_FILE"
```

Add `--draft` only when the user asks or the work is not ready for review. Add labels from the issue when they apply. Never push or merge without the user asking.

## 5. Before you report done

- The title passes the regex and the length check.
- No placeholder (`<issue>`, `<n>`, `C1`-style examples) or HTML comment is left in the body.
- Every `[x]` is backed by something you ran or a file in the diff.
- Report the PR URL.
