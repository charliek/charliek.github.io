# charliek.github.io

Project index page for Charlie Knudsen's GitHub projects.

## Development

Install dependencies:

```bash
uv sync --locked --group docs
```

Serve locally:

```bash
uv run --locked zensical serve
```

Build static site:

```bash
uv run --locked zensical build --strict
```

## Deployment

The site automatically deploys to GitHub Pages when changes are pushed to the `main` branch.
