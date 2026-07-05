# Portfolio site

This is a Hugo site for a simple portfolio plus writing.

## Preview locally

```bash
hugo server
```

Then open the local URL Hugo prints in the terminal.

## Edit project cards

Edit `data/projects.toml`.

Each project card looks like:

```toml
[[projects]]
label = "Project label"
title = "Project title"
meta = "Optional metadata"
bullets = [
  "First bullet.",
  "Second bullet.",
]
```

Reorder cards by moving the whole `[[projects]]` block.

## Build

```bash
hugo
```

The generated site goes into `public/`. Do not commit `public/`; GitHub Actions builds it for GitHub Pages.
