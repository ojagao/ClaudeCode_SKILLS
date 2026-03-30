---
name: resume-session
description: 過去のClaudeCodeセッションを検索し、resume用コマンドを生成する
argument-hint: <検索キーワード> 例: "AWS セキュリティ" "認証機能の実装" "PRレビュー"
allowed-tools: Bash, Read
---

# Resume Session スキル

過去のClaudeCodeセッション履歴を検索し、該当セッションを再開するための `claude --resume <sessionId>` コマンドを提示する。

## 手順

### 1. 引数の確認

`$ARGUMENTS` から検索キーワードを取得する。

- 引数がない場合（`$ARGUMENTS` が空文字列または未設定の場合）は、以下のメッセージを表示してユーザーに入力を求める。**検索は実行せず、ここで処理を終了する。**

```
検索キーワードを指定してください。

使い方:
  /resume-session <検索キーワード>

例:
  /resume-session AWS セキュリティ
  /resume-session 認証機能の実装
  /resume-session PRレビュー

複数キーワードを指定するとAND検索になります。
```

### 2. 履歴ファイルの検索

`~/.claude/history.jsonl` を末尾（最新）から検索する。

```bash
# ファイルの末尾から読み込み、検索キーワードに一致するエントリを探す
python3 -c "
import json, sys

keyword = sys.argv[1]
keywords = keyword.lower().split()

with open('$HOME/.claude/history.jsonl') as f:
    lines = f.readlines()

# 末尾（最新）から検索
matches = []
seen_sessions = set()
for line in reversed(lines):
    try:
        entry = json.loads(line)
    except:
        continue
    sid = entry.get('sessionId')
    if not sid or sid in seen_sessions:
        continue
    display = entry.get('display', '').lower()
    if all(k in display for k in keywords):
        seen_sessions.add(sid)
        matches.append(entry)
    if len(matches) >= 5:
        break

for m in matches:
    print(json.dumps(m, ensure_ascii=False))
" "$ARGUMENTS"
```

### 3. 検索結果の処理

- **一致なしの場合**: 「該当するセッションが見つかりませんでした。別のキーワードで試してください。」と案内して終了する
- **一致ありの場合**: 以下のフォーマットで各セッションの情報を表示する

### 4. 結果の表示

見つかったセッションについて、以下の情報を表示する:

1. **セッション概要**: `display` フィールドの内容を要約（長い場合は先頭200文字程度）
2. **プロジェクト**: `project` フィールド
3. **日時**: `timestamp` をローカル時刻に変換して表示
4. **resumeコマンド**: コピー可能な形式で提示

**表示フォーマット:**

```
### セッション 1
- **概要**: <displayの要約>
- **プロジェクト**: <project>
- **日時**: <YYYY-MM-DD HH:MM>

\`\`\`
claude --resume <sessionId>
\`\`\`
```

複数件ある場合は番号付きで全て表示する（最大5件）。

### 5. 補足

- 検索は部分一致（キーワードが複数の場合はAND検索）
- 同一sessionIdの重複は排除し、最新のエントリのみ表示する
- 最大5件まで表示する
- sessionIdを持たないエントリはスキップする

## 引数（$ARGUMENTS）の解釈

引数は検索キーワードとして扱う:

- `"AWS セキュリティ"` → displayに「AWS」と「セキュリティ」の両方を含むセッションを検索
- `"PR作成"` → displayに「PR作成」を含むセッションを検索
- `"認証"` → displayに「認証」を含むセッションを検索

## 注意事項

- `~/.claude/history.jsonl` が存在しない場合はエラーメッセージを表示して終了する
- 検索は大文字小文字を区別しない
- 最新のセッションが先に表示される（降順）
