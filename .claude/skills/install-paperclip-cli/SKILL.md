---
name: install-paperclip-cli
description: >
  Install or update a global `paperclipai` command backed by this local
  checkout's source, rather than a published npm version. Use when asked to
  "install the paperclip CLI locally", "set up paperclipai on my machine",
  or "update my local paperclipai install" from ./paperclip.
---

# Installing the local paperclipai CLI

This gives you a `paperclipai` command on `PATH` that always runs whatever is
currently checked out in this repo — useful for operating a deployment (e.g.
`paperclipai auth bootstrap-ceo`, `paperclipai configure`) or testing CLI
changes without publishing.

## Why not `npm link` / `pnpm build`

The obvious approach — `pnpm build` (esbuild bundles `cli/src/index.ts` to
`cli/dist/index.js`) then `npm link` — **does not work standalone**.
`cli/esbuild.config.mjs` deliberately leaves some npm dependencies external
(e.g. `zod`) rather than bundling them, on the assumption that `cli/package.json`
will be regenerated before publish by `scripts/generate-npm-package-json.mjs`
(which aggregates every workspace package's real dependencies — including
transitive-only ones like `zod`, pulled in via `packages/shared` — into a
publish-ready `cli/package.json`). Without running that regeneration, the
linked binary fails immediately:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'zod' imported from
.../cli/dist/index.js
```

Don't "fix" this by running `generate-npm-package-json.mjs` locally — it
overwrites `cli/package.json` in place with the publish-time dependency set,
which diverges the checkout from normal dev state (workspace:* references
replaced with pinned versions) and leaves the repo dirty.

## What actually works: a shim around the dev script

The root `package.json` already defines the correct invocation for running
the CLI straight from TypeScript source via `tsx`:

```
"paperclipai": "node cli/node_modules/tsx/dist/cli.mjs cli/src/index.ts"
```

`tsx`-based execution resolves each module from its own file's location, so
it correctly walks the pnpm workspace graph (e.g. resolves `zod` via
`packages/shared`'s own `node_modules/zod` symlink) — no bundling, no
external-deps gap. This is also exactly what runs inside the production
Docker image (`docker-entrypoint.sh` invokes `paperclipai` the same way).

### Install

```bash
cat > ~/.local/bin/paperclipai <<'EOF'
#!/bin/sh
# Runs the local ./paperclip checkout's CLI from source (tsx), matching the
# monorepo's own `pnpm paperclipai` dev script. Not the published npm
# package — always reflects whatever is currently checked out.
exec node <REPO_PATH>/cli/node_modules/tsx/dist/cli.mjs \
  <REPO_PATH>/cli/src/index.ts "$@"
EOF
chmod +x ~/.local/bin/paperclipai
```

Replace `<REPO_PATH>` with the absolute path to this checkout. `~/.local/bin`
must already be on `PATH` (check with `echo "$PATH" | tr ':' '\n' | grep .local/bin`;
if it isn't, use whichever directory on `PATH` you already use for personal
scripts).

### Verify

```bash
paperclipai --version
paperclipai --help
```

Both should work from any directory, not just inside the repo.

### Update

Since the shim always re-reads the live source, there is no separate
"update" step for CLI code changes — `git pull` (or switch branches) and the
next invocation picks it up immediately. Only re-run `pnpm install` (from the
repo root) if `pnpm-lock.yaml` or a `package.json` changed underneath you —
otherwise a dependency `tsx` needs to resolve may be missing or stale.

### Uninstall

```bash
rm ~/.local/bin/paperclipai
```

## Alternative: the CLI's own managed installer

For a version-pinned install (not tied to a live local checkout — e.g. to
test against a specific release or a colleague's branch without them handing
you a path), use the CLI's built-in installer instead:

```bash
paperclipai install                    # latest published npm version
paperclipai install --version 2026.916.0
paperclipai install --ref my-branch    # GitHub branch/tag/commit SHA
```

This installs into a managed per-user store (supports atomic updates and
rollback via `paperclipai update` / `paperclipai uninstall`) and is
independent of any local checkout. Prefer this over the shim above when you
don't need to reflect uncommitted local changes.
