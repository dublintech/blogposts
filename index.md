# AI in the SDLC

A series exploring how AI tooling can be integrated into software delivery without losing architectural integrity and engineering discipline.

Most AI discussions focus on prompts, agents, and code generation. This series focuses on something different:

- Architectural governance
- Machine-readable design intent
- Design debt management
- AI-assisted modernization
- Executable architecture
- Architecture fitness functions

---

## Articles

### Part 1

Coming Soon

---

### Part 2 - Indicating Your Design Debt to AI Tooling

This article introduces the concept of explicitly marking known design debt within a codebase so that AI tooling can distinguish intentional architectural exceptions from desired design patterns.

Key concepts:

- Design Debt vs Technical Debt
- Machine-readable architecture knowledge
- Annotating architectural exceptions
- Reducing the risk of AI reproducing existing anti-patterns

---

### Part 3 - Indicating Your Design Debt to AI Tooling (ArchUnit Integration)

This article extends the Design Debt concept by connecting known design debt to executable architecture rules using ArchUnit.

Key concepts:

- Architecture fitness functions
- ArchUnit rules
- Known architectural exceptions
- Architectural traceability
- AI-readable design intent
- Preventing the spread of architectural anti-patterns

Read the article:

./ai-sdlc-part-3-archunit-design-debt.md

---

## Core Idea

Traditional architecture governance relies heavily on tribal knowledge, documentation, design reviews, and code reviews.

AI-assisted development changes the equation.

If an AI tool cannot distinguish between:

- an approved design pattern
- a historical mistake
- a temporary workaround
- known design debt

then it may unintentionally reproduce undesirable patterns throughout a codebase.

The objective of this series is to make architectural intent:

- Explicit
- Discoverable
- Executable
- Verifiable
- Understandable by both humans and AI tooling

---

## Example Workflow

```text
Architecture Rule
        │
        ▼
ArchUnit Test
        │
        ▼
Known Design Debt Annotation
        │
        ▼
AI Tooling Understands Exception
        │
        ▼
No New Architectural Debt Introduced
        │
        ▼
Future AI Agent Removes Debt
        │
        ▼
Architecture Rule Passes Cleanly
