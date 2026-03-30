---
name: resume-session
description: 過去のClaudeCodeセッションをAIで意味検索し、resume用コマンドを生成する
argument-hint: <探したいセッションの説明> 例: "AWSのセキュリティ対応したやつ" "認証周りの実装" "先週のPRレビュー"
allowed-tools: Bash, Read
---

# Resume Session スキル

過去のClaudeCodeセッション履歴をAIが意味的に判断して検索し、該当セッションを再開するための `claude --resume <sessionId>` コマンドを提示する。

## 手順

### 1. 引数の確認

`$ARGUMENTS` からユーザーの検索意図を取得する。

- 引数がない場合（`$ARGUMENTS` が空文字列または未設定の場合）は、以下のメッセージを表示してユーザーに入力を求める。**検索は実行せず、ここで処理を終了する。**

```
探したいセッションの内容を教えてください。

使い方:
  /resume-session <探したいセッションの説明>

例:
  /resume-session AWSのセキュリティ対応したやつ
  /resume-session 認証周りの実装
  /resume-session 先週のPRレビュー
  /resume-session SSTの設定変えたセッション

自然な日本語で説明するだけでOKです。キーワード一致ではなくAIが内容を判断します。
```

### 2. 履歴の取得（ページネーション方式）

`~/.claude/history.jsonl` からやり取り（個別エントリ）を **100件ずつ** 取得する。
**セッション単位で集約せず、各エントリを独立したやり取りとして扱う。**

初回は `OFFSET=0` で実行する。

```bash
OFFSET=0
python3 -c "
import json, sys
from datetime import datetime

offset = int(sys.argv[1])
page_size = 100

with open('$HOME/.claude/history.jsonl') as f:
    lines = f.readlines()

# 末尾（最新）から走査し、各エントリを独立したやり取りとして出力
count = 0
skipped = 0
for line in reversed(lines):
    try:
        entry = json.loads(line)
    except:
        continue
    sid = entry.get('sessionId')
    if not sid:
        continue
    display = entry.get('display', '').strip()
    if not display:
        continue
    # offsetまでスキップ
    if skipped < offset:
        skipped += 1
        continue
    ts = entry.get('timestamp', 0)
    time_str = datetime.fromtimestamp(ts / 1000).strftime('%Y-%m-%d %H:%M') if ts else 'unknown'
    print(json.dumps({
        'sessionId': sid,
        'display': display[:300],
        'project': entry.get('project', ''),
        'time': time_str
    }, ensure_ascii=False))
    count += 1
    if count >= page_size:
        break

# 残りがあるかどうかを末尾に出力
remaining = sum(1 for l in lines if l.strip()) - offset - count
print('---HAS_MORE---' if remaining > 0 else '---END---')
" "$OFFSET"
```

### 3. AIによるやり取りの選定（ページネーション）

取得した100件のやり取りを読み、**ユーザーの検索意図（`$ARGUMENTS`）に最も合致するやり取りをAI（あなた自身）が判断して選定する。**

重要なポイント:
- **各エントリは独立したやり取りとして評価する**: セッション単位ではなく、個々の発言内容が検索意図に合致するかを判断する
- これにより、セッションの途中で話題が変わったケースでも正しくヒットする

判断基準:
- **意味的な関連性を重視する**: 完全一致ではなく、ユーザーが探しているであろうやり取りを文脈から推測する
  - 例: 「セキュリティ対応」→ SecurityGroup、WAF、IAM、脆弱性修正なども関連とみなす
  - 例: 「PR作ったやつ」→ PR作成、レビュー、マージ関連のやり取りを含む
  - 例: 「先週の作業」→ タイムスタンプも考慮して判断する
- **プロジェクトパスも判断材料にする**: ユーザーが特定のプロジェクトに言及している場合、projectフィールドも参考にする
- **同一sessionIdのやり取りが複数ヒットした場合は1つにまとめる**: 最も関連度の高いやり取りの内容を概要として使う
- **確信度が高い順に最大5件**選定する

**ページネーション:**
- 現在の100件に該当するやり取りが見つからず、出力末尾が `---HAS_MORE---` の場合:
  - `OFFSET` を +100 して手順2のスクリプトを再実行し、次の100件を取得する
  - 再度手順3で選定を行う
  - これを該当が見つかるか `---END---` に到達するまで繰り返す
- 該当が見つかった時点でページネーションを停止し、手順4へ進む
- **最大5ページ（500件）まで**遡る。それでも見つからない場合は手順5へ

### 4. 結果の表示

選定したやり取りについて、以下の情報を表示する:

1. **該当するやり取り**: ヒットしたdisplayの内容（ユーザーが何を言ったか）
2. **セッション概要**: そのやり取りの文脈をもとに、AIが何をしていたセッションかを簡潔に説明する
3. **プロジェクト**: projectフィールド
4. **日時**: タイムスタンプ
5. **resumeコマンド**: コピー可能な形式で提示

**表示フォーマット:**

```
### 1.
- **該当やり取り**: <ヒットしたdisplayの内容（長い場合は先頭150文字程度）>
- **概要**: <AIが推測するセッションの説明>
- **プロジェクト**: <project>
- **日時**: <YYYY-MM-DD HH:MM>

`claude --resume <sessionId>`
```

複数件ある場合は関連度が高い順に番号付きで全て表示する（最大5件）。

### 5. 該当なしの場合

意味的に関連するやり取りが見つからない場合は以下を表示する:

```
該当するセッションが見つかりませんでした。

別の表現で試してみてください:
- より具体的に: 「SSTのセキュリティグループ修正」
- より広く: 「AWS関連の作業」
- 時期で: 「先週やった作業」
```

## 引数（$ARGUMENTS）の解釈

引数はユーザーの自然言語による検索意図として解釈する:

- `"AWSのセキュリティ対応したやつ"` → AWS、セキュリティ、SecurityGroup等に関連するやり取り
- `"PRレビューしてたやつ"` → PR作成・レビュー関連のやり取り
- `"認証周り"` → 認証、auth、ログイン関連のやり取り
- `"instant-winの作業"` → プロジェクトパスにinstant-winを含むやり取り
- `"先週やったやつ"` → タイムスタンプが先週の範囲にあるやり取り
- `"パスワードUIの話"` → セッション途中のパスワードUI関連の発言にもヒット

## 注意事項

- `~/.claude/history.jsonl` が存在しない場合はエラーメッセージを表示して終了する
- 最新のやり取りが先に表示される（降順）
- 100件ずつページネーションで取得し、最大500件（5ページ）まで遡る
- 同一sessionIdが複数ヒットした場合は1エントリにまとめてresume コマンドは1つだけ提示する
- 判断に迷う場合は候補を多めに提示し、ユーザーに選んでもらう
