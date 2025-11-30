# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a MkDocs-based static site serving as a project index for charliek.github.io. It uses the Material theme with indigo colors and light/dark mode support.

## Commands

```bash
# Install dependencies
uv sync --group docs

# Serve locally (http://127.0.0.1:8000)
uv run mkdocs serve

# Build static site (output to site-build/)
uv run mkdocs build
```

## Deployment

The site automatically deploys to GitHub Pages via GitHub Actions when changes are pushed to the `main` branch. The workflow is defined in `.github/workflows/docs.yml`.
