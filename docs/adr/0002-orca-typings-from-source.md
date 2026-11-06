# 0002. Orca typings and validator from the pinned Orca source

Status: accepted (temporary)

## Context

Plugin code imports `orca/worker` and `orca/testing`. Their typings are not on npm; `orca plugin types .` writes them to `.orca/types`.

The packaged Orca app (checked on 1.4.248 and 1.4.251) does not ship those `.d.ts` files, so `orca plugin types .` fails with `ENOENT ... orca.d.ts` and `tsc` cannot resolve `orca/*`. CI runners have no Orca app either, so `orca plugin validate .` is not available there.

The `stablyai/orca` repository has the typings (`resources/plugins/declarative/`, `resources/plugins/types/`) and the validator (`pnpm validate:plugin`).

## Decision

- `npm run types` (`build/orca-types.mjs`) downloads the typings from a pinned `stablyai/orca` commit and writes `.orca/types` the same way `orca plugin types` does. `tsconfig.json` extends the result.
- CI runs `npm run types` before every task, and has an `orca-validate` job that checks out `stablyai/orca` at the same commit and runs its validator on the plugin.
- The commit is pinned in two places, `ORCA_REF` in `build/orca-types.mjs` and in `.github/workflows/ci.yml`. Bump both together, to a commit matching the Orca version we target.

## Consequences

- `tsc` and the manifest check work locally and in CI without an Orca app.
- Typings can drift from the installed Orca until the pin is bumped.
- `npm run types` needs network access.

## Revisit

When a packaged Orca ships the typings, drop `build/orca-types.mjs` and use `orca plugin types .`. The validator job can stay or be replaced by the installed CLI.
