# Agent Notes

## Repository nature

This repo is a **personal fork** of upstream tmux, recompiled and packaged by the owner.

- **`github` remote** → upstream: `https://github.com/tmux/tmux.git`
- **`origin` remote** → owner's Forgejo: `https://forgejo.xynogen.xyz/xynogen/tmux.git`

The owner regularly pulls from `github` (upstream) and merges into `origin` (their fork). The fork carries local patches — primarily CI/CD configuration for self-hosted Forgejo Actions producing `.deb` and `.rpm` artifacts attached to Forgejo Releases.

## Merge instability

Pulling and merging upstream tmux into the fork is **mostly stable but watch these points**:

- Upstream is a C codebase with autotools; churn is concentrated in `*.c`, `tmux.h`, `cmd-*.c`, and `configure.ac`
- The owner's patches live almost entirely under `.forgejo/` — collisions with upstream are rare but possible if upstream ever adds Forgejo/CI files
- `configure.ac` `AC_INIT([tmux], next-X.Y)` version string is parsed by the workflow → upstream version bumps directly affect derived package versions when not tag-driven
- `Makefile.am` / generated `Makefile` / `configure` script changes can break the `autogen.sh && ./configure && make` chain in CI; regenerate locally if you suspect autotools drift

### When helping with a post-merge issue

1. Always check `git log --oneline -20` first to see whether a merge commit is the source of the breakage.
2. Treat anything under `.forgejo/` as **owner's local patches** — preserve them unless explicitly told otherwise. Upstream rarely touches this path.
3. If the build fails in CI but works locally, suspect autotools cache or stale `configure` — run `sh autogen.sh` cleanly in the container.
4. For packaging dep mismatches (`libevent`, `libtinfo`, `libutempter`), distro version bumps may rename runtime libs — adjust `--depends` in `.forgejo/workflows/packages.yml`.

## Build environment summary

- Build targets (matrix): `debian:12`, `ubuntu:24.04`, `fedora:40` containers
- Build chain: `autogen.sh` → `./configure --prefix=/usr --sysconfdir=/etc --mandir=/usr/share/man` → `make` → `make install DESTDIR=stage`
- Packaging tool: **fpm** (single tool, both `.deb` and `.rpm` outputs)
- Trigger: git tag push (`on: push: tags: ["*"]`) + `workflow_dispatch` with optional version override
- Release publishing: `actions/forgejo-release@v2` attaches artifacts to the tag's release
- Runner: self-hosted Forgejo Actions runner (`runs-on: docker`)

### Version resolution priority (CI)

1. `workflow_dispatch.inputs.version` (manual override)
2. Git tag name with leading `v` stripped (e.g. `v3.7` → `3.7`)
3. Fallback: `configure.ac` `AC_INIT` version with `next-` prefix stripped + `+git<shortsha>`

Deb-side normalization: `-` → `~` (fpm rejects `-` in deb versions).

### Triggering CI

```sh
git tag -a v3.7 -m "3.7"
git push origin v3.7
```

Or manually via Forgejo UI → Actions → packages → Run workflow.

### Common gotchas

- `actions/forgejo-release@v2` requires Forgejo ≥ 1.21
- Runner labels: workflow uses `runs-on: docker`; adjust if runner registers different labels
- Deb runtime deps pinned (`libevent-2.1-7`, `libtinfo6`) — verify after Debian/Ubuntu version bumps
- `--architecture native` lets fpm pick `amd64`/`x86_64` from the runner; cross-arch builds require explicit overrides
