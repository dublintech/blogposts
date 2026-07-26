# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

A collection of Markdown blog posts in the "AI in the SDLC" series by Alex Staveley. No build system, test suite, or package manager — this is a pure content repository.

## Article Series Structure

Articles follow the naming convention `ai-sdlc-part-N-<slug>.md`. The series explores integrating AI tooling into software delivery while maintaining architectural integrity.

- `index.md` — Series overview and article index
- `posts/aisdlc/ai-sdlc-part-2-archunit-design-debt.md` — Introducing `@KnownApiDesignDebt` annotation to signal design debt to AI tools
- `posts/aisdlc/ai-sdlc-part-3-archunit-design-debt.md` — Linking `@KnownApiDesignDebt` to executable ArchUnit rules

Images are stored in `images/`. All blog post series live under `posts/<series-slug>/`.

## Content Conventions

- Java is used for all code examples; readers using other languages are directed to apply the underlying concepts with equivalent tooling.
- Part 3 onward uses YAML frontmatter (`title`, `author`, `tags`). New articles should follow this pattern.
- The series maintains a deliberate distinction between **design debt** (structural/architectural compromise) and **technical debt** (lower-level issues like formatting or naming).
- Cross-article references use relative Markdown links to other `.md` files.

## Understanding `@KnownApiDesignDebt`

When reading source files referenced in the articles, classes or methods annotated with `@KnownApiDesignDebt` signal **known, accepted structural debt** in a specific aspect of the class — not that the entire class, package, or solution is wrong. Read the `comment` field and the referenced JIRA ticket for context on the specific debt.
