---
layout:     post
title:      "Authoring markdown posts in Hugo"
description: "Some simple examples of using markdown with Hugo"
excerpt: "The following demonstrates how to create Markdown posts with Hugo."
date:    2025-02-02
author: "Paul"
publishDate: 2025-02-02
tags:
    - Hugo
    - Markdown
categories: [ tips ]
---


## Writing Articles using Markdown

> The following demonstrates how to create Markdown posts with Hugo.

<!--more-->

### Images

````text
![Porsche 911 RSR 2017](/images/porsche.webp)
````

renders as

![Porsche 911 RSR 2017](/images/porsche.webp)

### YouTube

```text
{{</* youtube lfGjtivHb-o */>}}
```

renders as

{{< youtube lfGjtivHb-o >}}

### Code Blocks

Use triple backticks to create code blocks, and specify the language for syntax highlighting.

````
```typescript
console.log('Hello, World!');
console.log('Hello, World!');
console.log('Hello, World!');
```
````

renders as

```typescript
console.log('Hello, World!');
console.log('Hello, World!');
console.log('Hello, World!');
```

## Mermaid diagrams

````text
```mermaid
graph TD
    A --> B
    A --> C
    B --> D
    C --> D
```
````

renders as

```mermaid
graph TD
    A --> B
    A --> C
    B --> D
    C --> D
```
