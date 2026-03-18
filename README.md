# ClaudeCode SKILLs

Claude Code の拡張機能（SKILL）を管理するリポジトリです。

## ディレクトリ構成

```
SKILLS/
└── <skill-name>/
    └── SKILL.md
```

各 SKILL は `SKILLS/<skill-name>/SKILL.md` に配置します。

## SKILL.md のフォーマット

```markdown
---
name: Skill Name
description: SKILLの説明
author: 作成者
version: 1.0.0
---

# Skill Name

SKILLの詳細な説明やコード
```

frontmatter の `name`, `description`, `author`, `version` が WeSKILL カタログに反映されます。
