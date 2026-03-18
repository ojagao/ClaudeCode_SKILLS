---
name: Sample Skill
description: これはサンプルSKILLです。Claude Codeでの基本的なSKILL構造を示します。
author: ojagao
version: 1.0.0
---

# Sample Skill

このSKILLはClaude Codeのサンプルです。

## 使い方

`/sample` コマンドで実行できます。

## 実装

```javascript
export default {
  name: "sample-skill",
  version: "1.0.0",
  description: "A sample skill for Claude Code",

  async execute(context) {
    const { input } = context

    return `Hello from Sample Skill! You said: ${input}`
  },
}
```
