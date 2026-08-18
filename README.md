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

## Edit the gccrs final report

Edit `content/gccrs-final-report/project-overview.md` for the project summary,
`content/gccrs-final-report/design-explanation.md` for the technical design,
and `content/gccrs-final-report/weekly-updates.md` for the weekly progress log.
Each Pull Request Note is a Markdown file in
`content/gccrs-final-report/pull-request-notes/`. Edit the prose directly in the
matching `pr-XXXX.md` file. Its front matter stores the PR number, status,
GitHub URL, overview group, and display order. Edit `_index.md` in the same
folder to change the overview introduction or group descriptions. PR
descriptions and commit messages are the main source for this text.
Standard Markdown headings, lists, links, and code blocks are rendered
automatically. All four pages are linked from `/gccrs-final-report/`.

## Build

```bash
hugo
```

The generated site goes into `public/`. Do not commit `public/`; GitHub Actions builds it for GitHub Pages.
