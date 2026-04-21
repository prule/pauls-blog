---
layout:     post
title:      "Eliminate PR Noise with ktfmt"
description: "How to use ktfmt to format Kotlin code consistently and reduce friction in code reviews."
date:    2026-04-21
author: "Paul"
publishDate: 2026-04-21
tags:
  - Kotlin
  - Programming
  - ktfmt
  - Productivity
categories: [ code ]
---

Few things are as frustrating during a code review as seeing a 50-line diff, only to realize that 45 of those lines are just re-ordered imports or adjusted indentation. This is "PR noise".

In the Kotlin world, **ktfmt** is one of the best tools to solve this. Developed by Meta, it's a deterministic formatter that ensures every developer on your team produces the exact same output, regardless of their individual IDE settings.

<!--more-->

## Why ktfmt?

Unlike some other formatters that allow for a lot of configuration, `ktfmt` is intentionally opinionated. It is based on the Google Java Formatter, but adapted for Kotlin. 

The philosophy is simple: **One way to format code.** By removing the ability to argue over where a curly brace should go, you free up your brain (and your PR comments) for things that actually matter: logic, architecture, and bugs.

## 1. IntelliJ Integration: Format on Save

The best way to use `ktfmt` is to never think about it. By installing the **ktfmt IntelliJ plugin**, you can have your IDE automatically format your code every time you save.

> See https://plugins.jetbrains.com/plugin/14912-ktfmt and https://github.com/facebook/ktfmt#intellij-android-studio-and-other-jetbrains-ides

### Setup:
1.  Install the **ktfmt** plugin from the IntelliJ Marketplace.
2.  Go to `Settings > Other Settings > ktfmt Settings`.
3.  Tick **Enable ktfmt**.
4.  Select one of the styles.
5.  Ensure **Reformat on save** is enabled in your IDE's "Actions on Save" settings.

Now, you can type as messily as you want; hitting `Cmd + S` will snap everything into a clean, consistent structure.

To make intellij format as closely as possible, grab one of the editorconfig files from here https://github.com/facebook/ktfmt/tree/main/docs/editorconfig and make sure to have the editorconfig plugin enabled https://plugins.jetbrains.com/plugin/7294-editorconfig



{{< notice type="info" title="Java Projects?" >}}
If you are working on a Java project instead of Kotlin, you can get the exact same benefits by using the **google-java-format** plugin and its corresponding Gradle/Maven plugins.

Use the spotless gradle plugin
https://github.com/diffplug/spotless/tree/main/plugin-gradle#google-java-format

and the Intellij plugin
https://plugins.jetbrains.com/plugin/8527-google-java-format
{{< /notice >}}

> Note, IntelliJ configuration (files in the `.idea` folder) can be added to git so it's contained in the repository for everyone.

## 2. Gradle Integration: The Source of Truth

While IDE plugins are great for individuals, you need a way to enforce formatting across the whole project. This is where the **ktfmt-gradle** plugin comes in.

> See https://github.com/cortinico/ktfmt-gradle

Add the following to your `build.gradle.kts`:

```kotlin
plugins {
    id("com.ncorti.ktfmt.gradle") version "0.26.0"
}

ktfmt {
    googleStyle() // or kotlinLangStyle() / facebookStyle()
}
```

Now you have access to two critical tasks:
- `./gradlew ktfmtCheck`: Validates that all code is correctly formatted (perfect for CI/CD).
- `./gradlew ktfmtFormat`: Re-formats the entire project from the command line.

## 3. Git Hooks: No More "Oops" Commits

Even with a Gradle plugin, developers sometimes forget to run the check before pushing. You can prevent unformatted code from ever entering your repository by using a **Git Hook**.

I recommend using **Husky** or a simple shell script in your `.git/hooks/pre-commit` file:

```bash
#!/bin/sh
# Run ktfmt on changed files before committing
./gradlew ktfmtFormat
git add .
```

This ensures that every commit is already compliant with the team's style guide, making your PRs look pristine from day one.

## Conclusion

Consistency is a superpower in software development. By adopting `ktfmt`, you reduce PR noise, allowing your team to move faster and focus on what really matters: building great software.

{{< notice type="info" title="Try it out" >}}
If you're starting a new Kotlin project, add `ktfmt` in the first commit. It's much easier to maintain consistency from the start than to reformat a massive legacy codebase!
{{< /notice >}}

---

*What are you using to keep your Kotlin code clean? Let me know on [LinkedIn](https://www.linkedin.com/in/paulrule/)!*
