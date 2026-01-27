---
title: Setting Up a Production-Grade Python Project (From Scratch)
date: 2026-01-27
categories:
  - Python
  - uv
  - ci/cd
authors:
  - david
tags:
  - pybites
  - pdm
description: >
  Notes on building a production-grade Python project from scratch, with an emphasis on tooling, quality gates, and repeatable workflows.
---

# Setting Up a Production-Grade Python Project (From Scratch)

Recently, I deliberately slowed down and spent time setting up a **production-grade Python project from first principles**, rather than jumping straight into feature development. The goal wasn’t to “get something working”, but to **understand how the tooling fits together** and why these practices matter in real teams.

This post captures what I focused on and what I learned.

<!-- more -->

---

## Why start from scratch?

It’s very easy to inherit a project where:
- CI already exists
- pre-commit hooks are already installed
- `pyproject.toml` is already configured

The problem is that you can *use* these things without really understanding them.

I wanted to:
- understand **when and why checks run**
- know **what lives locally vs in CI**
- be confident adjusting or introducing standards in the future
- remove “cargo-cult” tooling from my workflow

---

## A single source of truth: `pyproject.toml`

I started by treating `pyproject.toml` as the **central contract** for the project.

Key things I paid attention to:
- Separating **runtime dependencies** from **development dependencies**
- Explicitly configuring tooling rather than relying on defaults
- Making sure linting, formatting, typing, and testing all had a clear home

This helped reinforce that:
- tooling configuration belongs *with the code*
- new contributors should get the same behaviour by default
- CI should simply *reuse* what’s already defined locally

---

## Ruff, pytest, and mypy: clear responsibilities

I made sure I was clear on what each tool is responsible for:

- **Ruff**  
  Linting *and* formatting — fast feedback, consistent style, fewer bikeshedding discussions.

- **pytest**  
  Executable specifications. Tests describe what the code should do, not how it does it.

- **mypy**  
  Static type checking as a *design aid*, not a runtime requirement. Python still runs dynamically, but mypy gives early signals when assumptions don’t line up.

Understanding these roles made the setup feel intentional rather than “tool soup”.

---

## Pre-commit hooks: shifting quality left

One of the most useful things I revisited was **pre-commit**.

What clicked for me this time:
- Pre-commit is a **gate**, not a suggestion
- Hooks failing means the commit does *not* happen
- Auto-fixing hooks deliberately fail so you re-stage changes consciously

This creates a workflow where:
- mistakes are caught before they enter history
- CI becomes less noisy
- “fix lint” commits disappear

Local correctness first, shared correctness later.

---

## GitHub Actions: CI as an extension of local checks

Rather than inventing new rules in CI, I mirrored the local checks:

- lint
- format check
- type check
- tests + coverage

The important realisation here was:
> CI should *verify* standards, not *define* them.

If something fails in CI but wasn’t enforced locally, that’s a design smell.

---

## Why this mattered before writing more code

At first glance, this kind of setup can feel like “not making progress”.

In reality:
- I now have a **repeatable baseline** for future projects
- I’m confident explaining and justifying these choices to others
- adding new code feels safer and more deliberate
- I’m better equipped to improve or introduce standards in existing codebases

Most importantly, the tooling now *supports* learning rather than getting in the way of it.

---

## Final reflection

This was a reminder that good engineering isn’t just about writing code — it’s about **creating an environment where good code is the default outcome**.

Taking the time to understand the foundations has already paid off, and it’s a pattern I plan to repeat whenever I start something new.