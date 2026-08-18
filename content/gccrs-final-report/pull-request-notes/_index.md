---
title: "Pull Request Notes"
description: "An overview of the gccrs Drop patches from my GSoC project."
weight: 30
layout: "pr-notes"
groups:
  - id: "manual-drop"
    short_title: "Manual Drop"
    title: "1. Initial manual Drop emission"
    description: "Add the first Drop calls for supported locals and parameters, including Drop order, nested scopes, and normal function exits."
  - id: "structured-cleanup"
    short_title: "Structured cleanup"
    title: "2. Structured cleanup with TRY_FINALLY_EXPR"
    description: "Use one cleanup structure for blocks, functions, explicit returns, and supported unlabeled break and continue paths."
  - id: "move-analysis"
    short_title: "Move-aware cleanup"
    title: "3. Move-aware cleanup with BIR and CFG"
    description: "Track straight-line and conditional whole-local moves, then connect the analysis to backend cleanup with Drop flags."
---

> **Note.** All development during this project was submitted directly to the upstream [Rust-GCC/gccrs repository](https://github.com/Rust-GCC/gccrs).

All of the Drop-related contributions below were merged upstream. The detailed notes are based mainly on my pull request descriptions and commit messages. Each page also links to the related test cases.

[Read the Design Note](/gccrs-final-report/design-explanation/) for the full design story, or [view all of my gccrs pull requests on GitHub](https://github.com/Rust-GCC/gccrs/pulls?q=is%3Apr+author%3ALishin1215).
