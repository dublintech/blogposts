# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

A collection of Markdown blog posts by Alex Staveley, published at `https://dublintech.github.io/blogposts/`. The site is built with Jekyll and the Minimal Mistakes theme, deployed via GitHub Actions.

## Site Structure

- `index.md` — Landing page (DublinTech). Uses `layout: splash`. Lists all series with one-line summaries.
- `posts/<series-slug>/` — Blog post series. Current series: `posts/aisdlc/` (AI in the SDLC).
- `images/` — Shared images referenced by posts.
- `_data/navigation.yml` — Site navigation (overrides Minimal Mistakes default).
- `_config.yml` — Jekyll config: theme, author, Giscus comments, permalink, defaults.
- `Gemfile` — Ruby gems for local/CI build.
- `.github/workflows/jekyll.yml` — GitHub Actions deploy workflow.

## Jekyll & Theme

- **Theme**: `minimal-mistakes-jekyll` (gem, not remote_theme — required for Jekyll 4 compatibility).
- **Permalink**: `pretty` — pages generate as `path/to/page/index.html`, not `path/to/page.html`. Links in Markdown must use trailing slashes, not `.md` or `.html` extensions.
- **Wide layout**: All posts use `classes: wide` in frontmatter for a better desktop reading experience. New posts should include this.
- **Deployment**: GitHub Actions (`.github/workflows/jekyll.yml`). The GitHub Pages source must be set to "GitHub Actions" in repo settings — the native GitHub Pages builder is too old (Jekyll 3) for this theme.

## Post Frontmatter

All posts must include:

```yaml
---
title: "Post Title"
date: YYYY-MM-DD
classes: wide
author: "Alex Staveley"
tags:
  - tag1
  - tag2
---
```

## Image Paths

Images are in the root `images/` directory. Posts are served 3 URL levels deep (`/posts/<series>/<post>/`), so image references must use `../../../images/filename.jpg` (three levels up). Using `../../images/` will break.

## Navigation Links

In `_data/navigation.yml`, URLs must NOT include the baseurl (`/blogposts`). Minimal Mistakes applies `relative_url` automatically, so use `/` not `/blogposts/` — the latter results in a double-baseurl bug.

## Content Conventions

- All posts use **AI Coding Tool** (not "AI LLM", "Gen AI", or "AI tooling") when referring to tools like Claude Code, Amazon Kiro, Copilot, or Cursor.
- Java is used for all code examples; readers using other languages are directed to apply the underlying concepts with equivalent tooling.
- The series maintains a deliberate distinction between **design debt** (structural/architectural compromise) and **technical debt** (lower-level issues like formatting or naming).
- New blog series go under `posts/<series-slug>/` and should be listed on `index.md`.
- Cross-article links use relative paths with trailing slashes (e.g. `../ai-sdlc-part-2-archunit-design-debt/`).

## Comments & Reactions

Giscus is wired up for likes and comments on all posts, backed by GitHub Discussions (Announcements category, repo `dublintech/blogposts`). Enabled globally via `_config.yml` defaults.

## Understanding `@KnownApiDesignDebt`

When reading source files referenced in the articles, classes or methods annotated with `@KnownApiDesignDebt` signal **known, accepted structural debt** in a specific aspect of the class — not that the entire class, package, or solution is wrong. Read the `comment` field and the referenced JIRA ticket for context on the specific debt.
