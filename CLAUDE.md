# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a [Zensical](https://zensical.org)-based static site serving as a project index for charliek.github.io. Its look comes from the shared [stridelabs-docs-theme](https://github.com/charliek/stridelabs-docs-theme) package, pinned by tag in `pyproject.toml` — palette, fonts and feature toggles live there, not here, so restyling every StrideLabs docs site is a version bump rather than an edit in each repo.

## Commands

```bash
# Install dependencies
uv sync --locked --group docs

# Serve locally (http://127.0.0.1:8000)
uv run --locked zensical serve

# Build static site (output to site-build/) -- same as CI
uv run --locked zensical build --strict
```

`--strict` fails the build on broken links and heading anchors, and is what
both CI workflows run. Note `zensical serve --strict` is unsupported; verify
via `build`.

Two gotchas worth knowing: Zensical **silently ignores unknown config keys**
even under `--strict`, so a green build does not prove a config edit did what
you meant; and the `pymdownx.emoji` callables live in the
`zensical.extensions.emoji` namespace — the Material for MkDocs
`material.extensions.emoji` namespace aborts the build.

## Deployment

The site automatically deploys to GitHub Pages via GitHub Actions when changes are pushed to the `main` branch. The workflow is defined in `.github/workflows/docs.yml`; `.github/workflows/docs-pr.yml` runs the same strict build on pull requests.
