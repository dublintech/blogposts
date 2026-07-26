# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

A collection of Markdown blog posts by Alex Staveley, published at `https://dublintech.github.io/blogposts/`. The site is built with Jekyll and the Minimal Mistakes theme, deployed via GitHub Actions.

## Site Structure

- `index.md` — Landing page (DublinTech). Uses `layout: splash`. Contains author photo + bio as inline HTML, then article list.
- `posts/<series-slug>/` — Blog post series. Current series: `posts/aisdlc/` (AI in the SDLC).
- `images/` — Shared images. `me.jpg` is the author photo used on the landing page.
- `_data/navigation.yml` — Site navigation (overrides Minimal Mistakes default nav).
- `_config.yml` — Jekyll config: theme, author, Giscus comments, permalink, defaults.
- `Gemfile` — Ruby gems for local/CI build.
- `.github/workflows/jekyll.yml` — GitHub Actions deploy workflow.
- `assets/css/main.scss` — Custom CSS: Lora serif font, line height, paragraph spacing, muted red links.
- `_includes/head/custom.html` — Loads Lora from Google Fonts.

## Adding a New Post — Checklist

Every new post needs all of the following or things will break silently:

1. **File location**: `posts/<series-slug>/your-post-slug.md`
2. **Frontmatter** — use the template below. Missing `date` means no publish date shows.
3. **No `# H1` heading in the markdown body** — the theme renders the title from frontmatter automatically. Adding a `# Heading` creates a duplicate title on the page.
4. **Image paths**: use `../../../images/filename.jpg` (3 levels up — see Image Paths section).
5. **Links to other posts**: use trailing slash, no `.md` or `.html` — e.g. `../other-post/` not `../other-post.md`.
6. **Add to `index.md`**: one-line summary + link with trailing slash.

### Post frontmatter template

```yaml
---
title: "Post Title"
date: YYYY-MM-DD
author: "Alex Staveley"
tags:
  - tag1
  - tag2
---
```

## Layout & Theme Decisions

- **Theme**: `minimal-mistakes-jekyll` gem. Do NOT switch to `remote_theme: mmistakes/minimal-mistakes` — the latest Minimal Mistakes uses the `include_cached` Liquid tag which is not on GitHub Pages' native gem whitelist, causing a build failure.
- **Permalink**: `pretty` — pages generate as `path/to/page/index.html`. Links in Markdown must use trailing slashes only — not `.md` or `.html` extensions. Using `.md` links will result in 404s.
- **Deployment**: GitHub Actions (`.github/workflows/jekyll.yml`). The GitHub Pages source in repo Settings must be set to "GitHub Actions" — the native GitHub Pages builder uses Jekyll 3 which cannot build this theme.
- **Author profile**: `author_profile: false` globally. The author photo and bio live on `index.md` only, as inline HTML using `{{ "/images/me.jpg" | relative_url }}`. Do not add `author_profile: true` to post frontmatter.
- **TOC**: `toc: true` and `toc_sticky: true` enabled globally — all posts get a sticky table of contents in the right sidebar automatically.
- **No `classes: wide`**: The layout uses a centred content column with TOC in the right sidebar. Do not add `classes: wide` to posts.

## Typography

Custom typography is in `assets/css/main.scss`:
- **Body font**: Lora serif (loaded from Google Fonts via `_includes/head/custom.html`)
- **Line height**: 1.8
- **Paragraph spacing**: 1.4em bottom margin
- **Link colour**: muted dark red (`#8B2222`), similar to martinfowler.com

## Image Paths

Images live in the root `images/` directory. Posts are served 3 URL levels deep (`/blogposts/posts/<series>/<post>/`), so image `src` paths must go up 3 levels: `../../../images/filename.jpg`.

**Gotcha**: `../../images/` (2 levels) looks correct but is wrong — it resolves to `/blogposts/posts/images/` not `/blogposts/images/`. Using it will silently serve a broken image with no build error.

## Navigation Links

In `_data/navigation.yml`, URLs must NOT include the baseurl (`/blogposts`). Minimal Mistakes applies the `relative_url` filter automatically, so:

- Use `url: /` for the home link ✓
- Do NOT use `url: /blogposts/` — this results in `/blogposts/blogposts/` (double-baseurl bug) ✗

## Content Conventions

- All posts use **AI Coding Tool** (not "AI LLM", "Gen AI", or "AI tooling") when referring to tools like Claude Code, Amazon Kiro, Copilot, or Cursor.
- Java is used for all code examples; readers using other languages are directed to apply the underlying concepts with equivalent tooling.
- The series maintains a deliberate distinction between **design debt** (structural/architectural compromise) and **technical debt** (lower-level issues like formatting or naming).
- New blog series go under `posts/<series-slug>/` and should be listed on `index.md`.

## Comments & Reactions

Giscus is wired up for likes and comments on all posts, backed by GitHub Discussions (Announcements category, repo `dublintech/blogposts`). Enabled globally via `_config.yml` defaults — new posts get comments automatically without any extra config.

If Giscus ever needs to be set up again: (1) enable Discussions in repo Settings, (2) install the Giscus GitHub App at `https://github.com/apps/giscus`, (3) get the repo ID and category ID via the GitHub GraphQL API, then add to `_config.yml`.

## Understanding `@KnownApiDesignDebt`

When reading source files referenced in the articles, classes or methods annotated with `@KnownApiDesignDebt` signal **known, accepted structural debt** in a specific aspect of the class — not that the entire class, package, or solution is wrong. Read the `comment` field and the referenced JIRA ticket for context on the specific debt.
