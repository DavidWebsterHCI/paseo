# Fork notes

Personal fork of [getpaseo/paseo](https://github.com/getpaseo/paseo) (AGPL-3.0).

Goal: drive the three parallel `iamhuman` workspaces (`dev1`/`dev2`/`dev3`) from a
real UI instead of terminal tabs, without giving up the service isolation that
already works there.

## Branch model

| Branch | Contents |
|---|---|
| `main` | clean mirror of `upstream/main` — **never commit here** |
| `dw/customizations` | our patches, rebased onto `main` on every update |

Keeping `main` pristine is what makes updating cheap: each sync is a
fast-forward and our patches replay on top as a tidy stack. Committing to `main`
turns every future sync into conflict archaeology.

`upstream`'s push URL is set to `DISABLED` so a stray `git push upstream` can't fire.

## Updating from upstream

```bash
paseo-update            # fetch, fast-forward main, rebase our branch, push
paseo-update --no-push  # same, local only
```

The script lives at `~/.local/bin/paseo-update`, deliberately outside this
checkout — a script inside the repo would itself become a file upstream could
touch. It refuses to run with a dirty tree, and stops with instructions if the
rebase conflicts.

Upstream is very active (~4,700 commits, pushed daily), so expect to reinstall
after a sync — the script reminds you when upstream actually moved.

## Build

Requires Node **22.20.0** (`.tool-versions`); Node 24 works. npm workspaces, not pnpm.

```bash
npm ci                # NOT npm install — see below
npm run build
npm run dev:server    # daemon on 127.0.0.1:6768
npm run dev:app       # Expo client on :8081
npm run dev:desktop   # Electron, picks first free port 8082-8089
```

**Use `npm ci`, not `npm install`.** `install` re-resolves optional and peer
dependencies and rewrites `package-lock.json` even when nothing actually changed
— roughly 230 lines of churn on a fresh clone here, all optional/peer noise. That
dirty lockfile then blocks `paseo-update`, which refuses to rebase a dirty tree.
`npm ci` installs exactly what the lockfile specifies and never modifies it.

A fresh clone needs `npm run build` before the CLI works; `packages/cli/bin/paseo`
imports `dist/index.js`, which doesn't exist until then.

### Dependency audit

83 advisories at fork time, 7 critical — **all 7 are dev tooling** (vitest,
`@vitest/browser`, concurrently, shell-quote, tar). Production-only audit
(`npm audit --omit=dev`) shows no criticals; the notable ones there are a
picomatch ReDoS and sharp/libvips CVEs. Worth re-checking after each sync.

`PASEO_HOME` holds runtime state (agents, worktrees, sockets, daemon log);
defaults to `~/.paseo`.

## Where the relevant code lives

Mapped for the customizations we care about:

| Concern | File |
|---|---|
| Worktree creation / bootstrap | `packages/server/src/server/worktree-bootstrap.ts` |
| Worktree core logic | `packages/server/src/server/worktree-core.ts` |
| **Branch & directory naming** | `packages/server/src/server/worktree-branch-name-generator.ts` |
| Worktree service (daemon API) | `packages/server/src/server/paseo-worktree-service.ts` |
| **Port allocation** | `packages/server/src/server/workspace-service-port-allocator.ts` |
| Port registry | `packages/server/src/server/workspace-service-port-registry.ts` |
| **Config schema** (`paseo.json`) | `packages/protocol/src/paseo-config-schema.ts` |
| Low-level worktree utils | `packages/server/src/utils/worktree.ts` |

## Try configuration before patching

Much of what we want may not need a code change. Upstream already supports:

- `portScript` — an executable receiving service name, workspace ID, branch name
  and worktree path (also as `PASEO_SCRIPTNAME`, `PASEO_WORKSPACE_ID`,
  `PASEO_BRANCH_NAME`, `PASEO_WORKTREE_PATH`). Must be a real executable with a
  shebang; inline shell commands are rejected.
- `worktree.setup` / `worktree.teardown` — arbitrary shell, run with the worktree
  as cwd.
- `worktrees.root` — relocate the worktree base directory.

Since the iamhuman worktrees are on branches `wt/dev1|wt/dev2|wt/dev3`, the
**branch name is a stable index** — a `portScript` can derive N from it and return
the fixed offsets without touching Paseo's source at all.

Point the service command at `pnpm run dev:N` rather than bare `next dev`, so
`dev-env.sh`'s cross-environment guard stays in the path.

What upstream explicitly does *not* do, and would need real patches:

- No Docker Compose / database / infrastructure management (services only)
- No per-worktree `.env` templating — `setup` has to copy or generate it

Both are already handled on the iamhuman side by `scripts/worktree-env.mjs` and
`~/.local/bin/wt`, so the first attempt should wire those in via `setup`/`teardown`
rather than reimplementing them here.

## Known upstream issue to watch

[#783 — worktree metadata branch name goes stale after first-agent auto-rename](https://github.com/getpaseo/paseo/issues/783).
Any scheme deriving identity from the branch name (like the `portScript` above)
inherits this. Pin branch names and verify `PASEO_BRANCH_NAME` before trusting it.
