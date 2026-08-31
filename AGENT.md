# Agent Notes

## Repository nature

This repository is a **public personal fork** of upstream tmux.

- **`upstream` remote** → `https://github.com/tmux/tmux.git`
- **`origin` remote** → `https://github.com/xynogen/tmux.git`

The fork regularly merges `upstream/master` into `master`. Fork-only changes are packaging automation under `.github/workflows/packages.yml` and this file; prefer upstream for tmux source changes unless explicitly maintaining a local patch.

## Merge guidance

1. Check `git log --oneline -20` before resolving post-merge problems.
2. Preserve `.github/workflows/packages.yml` when merging upstream.
3. Upstream churn is concentrated in `*.c`, `tmux.h`, `cmd-*.c`, `configure.ac`, and autotools files.
4. If CI fails but a prior build worked, run `sh autogen.sh` from a clean tree before changing build logic.
5. Recheck package dependencies when Ubuntu changes runtime library names.

## Build and packaging

Workflow: `.github/workflows/packages.yml`

- Runner: GitHub-hosted `ubuntu-latest`
- Build container: `ubuntu:24.04`
- Build chain: `autogen.sh` → `./configure --prefix=/usr --sysconfdir=/etc --mandir=/usr/share/man` → `make` → staged install
- Packaging: `fpm` creates `.deb` and `.rpm` artifacts
- Trigger: any tag push or manual `workflow_dispatch`
- Release: tag runs create/update a GitHub Release and upload package artifacts

Version resolution priority:

1. `workflow_dispatch.inputs.version`
2. Tag name with leading `v` removed
3. `configure.ac` version with `next-` removed, plus `+git<shortsha>`

Debian versions replace `-` with `~` because `fpm` rejects `-` in Debian package versions.

## Release safety

Pushing a tag triggers packaging and release publication. Never tag or push a tag without explicit confirmation. Before release, verify the tag does not already exist locally or on `origin`.

```sh
git tag -l "<version>"
git ls-remote --tags origin "<version>"
```

Common issue: Ubuntu 24.04 uses `libevent-2.1-7t64`; recheck pinned runtime dependencies after base-image upgrades.
