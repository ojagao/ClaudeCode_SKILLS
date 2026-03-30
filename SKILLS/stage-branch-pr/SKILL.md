---
name: stage-branch-pr
description: addされた変更を確認し、内容に基づいたブランチを作成、commit、push、PR作成までを一括で行う
argument-hint: [追加指示(省略可) 例: "PRのベースブランチをdevelopにして" "commitメッセージにissue番号を入れて"]
allowed-tools: Bash, Read, Grep, Glob, Agent
---

# Stage → Branch → Commit → Push → PR スキル

addされた変更内容を分析し、適切なブランチ名でブランチを切り、commit、push、PR作成まで一気通貫で実行する。

## 手順

### 1. ステージング状態の確認

```bash
git status
git diff --cached --stat
git diff --cached
```

- ステージされたファイルがない場合は「git add でファイルをステージしてから再実行してください」と案内して終了する
- ステージされていない変更がある場合は、その一覧を表示して「これらはコミットに含まれません」と案内する

### 2. 変更内容の分析

ステージされた差分(`git diff --cached`)を読み、以下を判定する:

- **変更の種類**: feat(新機能), fix(修正), refactor(リファクタ), docs(ドキュメント), test(テスト), chore(雑務), perf(パフォーマンス), ci(CI/CD)
- **変更の対象**: 何が変わったのか（機能名、コンポーネント名、モジュール名など）
- **変更の概要**: 1行で説明できる要約

### 3. ブランチ作成

変更内容に基づいてブランチ名を生成し、作成する。

**ブランチ命名規則:**
```
<type>/<短い説明（kebab-case、英語）>
```

例:
- `feat/add-reservation-reminder`
- `fix/calendar-slot-count`
- `refactor/extract-auth-middleware`
- `chore/update-seed-data`

```bash
# 現在のブランチを確認
git branch --show-current

# 新しいブランチを作成して切り替え
git checkout -b <branch-name>
```

**注意:**
- 既に適切な名前のブランチにいる場合は新しいブランチを作成しない（ユーザーに確認する）
- mainやdevelopブランチ上の場合は必ず新しいブランチを作成する

### 4. コミット

変更内容に基づいてコミットメッセージを生成し、コミットする。

**コミットメッセージ形式:**
```
<type>: <日本語での変更概要>

<変更の詳細（必要に応じて箇条書き）>
```

```bash
git commit -m "$(cat <<'EOF'
<type>: <概要>

<詳細>
EOF
)"
```

### 5. プッシュ

```bash
git push -u origin <branch-name>
```

### 6. PR作成

変更内容を分析してPRのタイトルと本文を生成し、`gh` コマンドでPRを作成する。

**ベースブランチの決定:**
- デフォルトは `main`
- ユーザーが `$ARGUMENTS` で指定した場合はそれに従う

```bash
gh pr create --title "<PRタイトル>" --body "$(cat <<'EOF'
## Summary
- <変更内容の箇条書き>

## Test plan
- [ ] <テスト項目>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### 7. 完了報告

以下を表示する:
- 作成したブランチ名
- コミットメッセージ
- PR URL
- PRのベースブランチ

## 引数（$ARGUMENTS）の解釈

引数は自由文で追加指示を受け付ける。以下は代表的な指示例:

- `"PRのベースブランチをdevelopにして"` → `--base develop` を指定
- `"commitメッセージにissue番号を入れて #123"` → コミットメッセージにissue番号を含める
- `"ブランチ名は fix/xxx にして"` → 指定されたブランチ名を使用
- `"draftで作って"` → `--draft` フラグを追加
- 引数なし → デフォルト設定（ベース: main、通常PR）で実行

## 注意事項

- ステージされた変更のみを対象とする（未ステージの変更は無視）
- ブランチ名は英語kebab-case、コミットメッセージの概要は日本語
- pre-commitフックが失敗した場合は、問題を修正して新しいコミットを作成する（amendしない）
- `.env`やクレデンシャルファイルがステージされている場合は警告して中断する
- 強制プッシュは行わない
