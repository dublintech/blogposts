---
title: "AI in the SDLC Part 3 - Linking Design Debt to ArchUnit"
author: "Alex Staveley"
tags:
  - ai
  - sdlc
  - architecture
  - archunit
  - java
  - design-debt
---

# AI in the SDLC Part 3 - Linking Design Debt to ArchUnit

In the previous post, I described how Design Debt can be indicated to AI tooling.

When I say AI tooling, that could be Claude Code, Amazon Kiro, Copilot, Cursor, or whatever happens to be your preferred development assistant. The specific tool is not important.

This week we take the idea a step further by connecting those Design Debt indicators to ArchUnit.

> Throughout this series, Java is used for examples. Where Java-specific technologies such as ArchUnit are mentioned, readers using other languages should focus on the underlying concepts and substitute equivalent tooling.

## Recap

In the previous article, the idea was to indicate that a piece of code represents Design Debt or an anti-pattern that should not be repeated.

For example:

```java
@KnownApiDesignDebt(
    jira = "BLIP-7",
    comment = "Semantic version pattern broken. Should only be using SemanticService, not separate services."
)
public class PaymentController {
    // ...
}
```

This immediately raises some important questions:

1. How do you know the same Design Debt does not exist elsewhere?
2. What if the AI
