---
title: "AI in the SDLC Part 3 - Linking Design Debt to ArchUnit"
date: 2026-07-26
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

<p align="center">
  <img src="../../../images/Warrior.png" alt="Warrior" />
</p>

In the previous post, I described how Design Debt can be indicated to AI Coding Tools.

When I say AI Coding Tools, that could be Claude Code, Amazon Kiro, Copilot, Cursor, or whatever happens to be your preferred AI Coding Tool. The specific tool is not important.

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
2. What if the AI Coding Tool reads another example that contains the same anti-pattern but does not contain the annotation?
3. What if the AI Coding Tool ignores both examples and simply reproduces the same design mistake?

Realistically, nobody can review everything.

This is where ArchUnit comes in.

## Expressing Design Intent With ArchUnit

If something is being identified as Design Debt, it should be possible to express the intended architecture as an executable architecture rule.

By Design Debt, I mean a structural or architectural compromise that breaks an established design pattern. This is different from lower-level Technical Debt such as formatting, naming, or syntax conventions that can generally be enforced through linting.

Let's look at an example.

Suppose every API Resource owns all of its API-related code within its own package.

For example:

```text
/pets
```

might support:

- GET
- POST
- PATCH
- DELETE
- QUERY

Each resource package contains a single Query object responsible for capturing query parameters.

The intention is that there should only ever be one Query class per resource.

This prevents situations such as:

- Multiple Query objects inheriting from each other unnecessarily
- Several Query endpoints being created instead of extending the existing query capabilities
- Resource packaging conventions being broken

Imagine somebody accidentally introduces three Query classes.

Why might this happen?

1. They forgot the packaging strategy and squeezed several resources into one package.
2. They introduced multiple query endpoints instead of extending the existing query model.

Either way, we want ArchUnit to prevent this from spreading.

## Creating the ArchUnit Rule

The rule could be declared as:

```java
@ArchTest
public ArchRule RESOURCE_PACKAGE_CAN_ONLY_HAVE_ONE_QUERY_CLASS_RULE =
    ApiConstructsHelper.ENTRY_POINT_CLASSES
        .should(RESOURCE_PACKAGE_CAN_ONLY_HAVE_ONE_QUERY_CLASS_CONDITION);
```

The description of the rule:

```java
public static final String
    RESOURCE_PACKAGE_CAN_ONLY_HAVE_ONE_QUERY_CLASS_CONDITION_DESC =
        "have only one Query class in the resource package";
```

And the condition might look something like:

```java
public static final ArchCondition<JavaClass>
    RESOURCE_PACKAGE_CAN_ONLY_HAVE_ONE_QUERY_CLASS_CONDITION =
        new ArchCondition<>(
            RESOURCE_PACKAGE_CAN_ONLY_HAVE_ONE_QUERY_CLASS_CONDITION_DESC) {

    @Override
    public void check(
            final JavaClass javaClass,
            final ConditionEvents conditionEvents) {

        final JavaPackage javaPackage = javaClass.getPackage();
        final Set<JavaClass> classes = javaPackage.getClasses();

        final List<JavaClass> queryClasses =
            classes.stream()
                .filter(clazz ->
                    clazz.isAssignableTo(
                        com.mystartup.api.Query.class))
                .collect(Collectors.toList());

        if (queryClasses.size() > 1) {

            final String classNames =
                queryClasses.stream()
                    .map(JavaClass::getSimpleName)
                    .collect(Collectors.joining(", "));

            final String responseMessage =
                String.format(
                    "Package %s has more than one Query class. Query classes are currently: %s. You should only have one.",
                    javaPackage.getName(),
                    classNames);

            conditionEvents.add(
                SimpleConditionEvent.violated(
                    javaClass,
                    responseMessage));
        }
    }
};
```

Boom.

Now ArchUnit ensures there is only a single Query class per Resource package.

## The Problem

Unfortunately, the rule also fails because the Design Debt already exists.

So how do we stop new violations without immediately failing because of historic problems that are already known?

This is where we connect Design Debt and ArchUnit.

## Linking Design Debt to Architecture Rules

Extend the annotation so that it can reference the ArchUnit rule it is violating.

For example:

```java
package com.mystartup.api.endpoints.pet;

@KnownApiDesignDebt(
    jira = "BLIP-70",
    comment = "Single Query Class broken",
    archUnitDescription =
        "have only one Query class in the resource package"
)
public class PetQuery {
    // ...
}
```

And:

```java
package com.mystartup.api.endpoints.pet;

@KnownApiDesignDebt(
    jira = "BLIP-70",
    comment = "Single Query Class broken",
    archUnitDescription =
        "have only one Query class in the resource package"
)
public class SomeOtherPetQuery {
    // ...
}
```

The ArchUnit condition can then be enhanced:

```java
if (queryClasses.size() > 1) {

    if (KnownApiDesignDebtHelper.hasKnownApiDesignDebt(
            queryClasses.get(0),
            RESOURCE_PACKAGE_CAN_ONLY_HAVE_ONE_QUERY_CLASS_CONDITION_DESC)) {

        // Known and acknowledged Design Debt.
        // Do not fail the build.

        return;
    }

    final String classNames =
        queryClasses.stream()
            .map(JavaClass::getSimpleName)
            .collect(Collectors.joining(", "));

    final String responseMessage =
        String.format(
            "Package %s has more than one Query class. Query classes are currently: %s.",
            javaPackage.getName(),
            classNames);

    conditionEvents.add(
        SimpleConditionEvent.violated(
            javaClass,
            responseMessage));
}
```

## Utility Method

The helper method is straightforward:

```java
public static boolean hasKnownApiDesignDebt(
        final JavaClass javaClass,
        final String conditionName) {

    try {

        final Class<?> clazz = javaClass.reflect();

        final KnownApiDesignDebt[] debts =
            clazz.getAnnotationsByType(
                KnownApiDesignDebt.class);

        return Arrays.stream(debts)
            .anyMatch(kd ->
                conditionName.equals(
                    kd.archUnitDescription()));

    } catch (final Exception e) {

        return false;
    }
}
```

## What Does This Achieve?

Now ArchUnit understands the difference between:

- New violations
- Existing acknowledged Design Debt

As a result:

- Existing debt can remain visible and documented
- New occurrences are prevented
- Architectural intent becomes machine-readable
- AI Coding Tools receive stronger signals about what is and is not a valid pattern

Importantly, if the ArchUnit rule is green, you know one of two things is true:

1. The violation does not exist.
2. The violation exists but has been explicitly acknowledged as Design Debt.

This significantly reduces the risk of AI Coding Tools learning the wrong lessons from legacy code.

## The Interesting Part for AI Coding Tools

This becomes even more powerful when combined with AI agents.

Imagine an agent that:

1. Finds `@KnownApiDesignDebt`.
2. Identifies the associated ArchUnit rule.
3. Uses the architecture rule to understand the intended design.
4. Refactors the code to comply.
5. Removes the `@KnownApiDesignDebt` annotation.
6. Runs the ArchUnit suite.
7. Confirms the rule now passes.

The architecture rule becomes the specification.

The Design Debt annotation becomes the exception.

The AI Coding Tool becomes the remediation engine.

## Conclusion

By linking Design Debt annotations to executable ArchUnit rules, architectural intent becomes explicit for both humans and AI Coding Tools.

This provides several benefits:

- Known Design Debt can be acknowledged without blocking delivery.
- New violations are prevented automatically.
- AI Coding Tools are less likely to learn from architectural anti-patterns.
- Architecture rules become a source of truth for future automated remediation.

The result is a codebase that is easier to govern, easier to modernise, and easier for both developers and AI Coding Tools to understand.
