---
layout: post
title: "Formatting Showcase"
date: 2026-02-03 14:30:00 +0000
categories: reference
tags: [markdown, code]
---

A dummy post that stresses the theme's typography.

### Lists

1. First item
2. Second item
   - Nested bullet
   - Another one
3. Third item

### Code

Inline `code` looks like this. A fenced block:

```al
codeunit 50100 "TES Sample"
{
    procedure TESGreet(Name: Text): Text
    begin
        exit(StrSubstNo('Hello, %1', Name));
    end;
}
```

### Table

| Object   | ID    | Purpose        |
|----------|-------|----------------|
| Codeunit | 50100 | Sample logic   |
| Page     | 50101 | Sample UI      |

That's the tour.
