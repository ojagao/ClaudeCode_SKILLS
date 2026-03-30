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

### 2. 履歴の取得

`~/.claude/history.jsonl` から直近のユニークなセッション一覧を取得する。

```bash
python3 -c "
import json, sys
from datetime import datetime

with open('$HOME/.claude/history.jsonl') as f:
    lines = f.readlines()

# 末尾（最新）から走査し、セッションごとに最新のエントリを収集
# 同一セッション内の全displayを結合して、セッションの全体像を把握できるようにする
sessions = {}  # sessionId -> { first_display, all_displays, project, timestamp }
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
    if sid not in sessions:
        sessions[sid] = {
            'sessionId': sid,
            'displays': [display],
            'project': entry.get('project', ''),
            'timestamp': entry.get('timestamp', 0)
        }
    else:
        sessions[sid]['displays'].append(display)
    # 最大100セッション分を収集したら打ち切り
    if len(sessions) >= 100:
        break

# 新しい順にソートして出力
sorted_sessions = sorted(sessions.values(), key=lambda x: x['timestamp'], reverse=True)
for s in sorted_sessions:
    # displaysを逆順にして時系列順にし、結合（各display先頭150文字まで）
    displays_chrono = list(reversed(s['displays']))
    summary = ' | '.join(d[:150] for d in displays_chrono[:5])
    ts = datetime.fromtimestamp(s['timestamp'] / 1000).strftime('%Y-%m-%d %H:%M') if s['timestamp'] else 'unknown'
    print(json.dumps({
        'sessionId': s['sessionId'],
        'summary': summary,
        'project': s['project'],
        'time': ts
    }, ensure_ascii=False))
"
```

### 3. AIによるセッション選定

上記で取得したセッション一覧を読み、**ユーザーの検索意図（`$ARGUMENTS`）に最も合致するセッションをAI（あなた自身）が判断して選定する。**

判断基準:
- **意味的な関連性を重視する**: 完全一致ではなく、ユーザーが探しているであろうセッションを文脈から推測する
- 例: 「セキュリティ対応」→ SecurityGroup、WAF、IAM、脆弱性修正なども関連とみなす
- 例: 「PR作ったやつ」→ PR作成、レビュー、マージ関連のセッションを含む
- 例: 「先週の作業」→ タイムスタンプも考慮して判断する
- **プロジェクトパスも判断材料にする**: ユーザーが特定のプロジェクトに言及している場合、projectフィールドも参考にする
- **確信度が高い順に最大5件**選定する

### 4. 結果の表示

選定したセッションについて、以下の情報を表示する:

1. **セッション概要**: displayの内容をもとに、何をしていたセッションかを自分の言葉で簡潔に説明する
2. **プロジェクト**: projectフィールド
3. **日時**: タイムスタンプ
4. **resumeコマンド**: コピー可能な形式で提示

**表示フォーマット:**

```
### セッション 1
- **概要**: <AIが生成したセッションの説明>
- **プロジェクト**: <project>
- **日時**: <YYYY-MM-DD HH:MM>

`claude --resume <sessionId>`
```

複数件ある場合は関連度が高い順に番号付きで全て表示する（最大5件）。

### 5. 該当なしの場合

意味的に関連するセッションが見つからない場合は以下を表示する:

```
該当するセッションが見つかりませんでした。

別の表現で試してみてください:
- より具体的に: 「SSTのセキュリティグループ修正」
- より広く: 「AWS関連の作業」
- 時期で: 「先週やった作業」
```

## 引数（$ARGUMENTS）の解釈

引数はユーザーの自然言語による検索意図として解釈する:

- `"AWSのセキュリティ対応したやつ"` → AWS、セキュリティ、SecurityGroup等に関連するセッション
- `"PRレビューしてたやつ"` → PR作成・レビュー関連のセッション
- `"認証周り"` → 認証、auth、ログイン関連のセッション
- `"instant-winの作業"` → プロジェクトパスにinstant-winを含むセッション
- `"先週やったやつ"` → タイムスタンプが先週の範囲にあるセッション

## 注意事項

- `~/.claude/history.jsonl` が存在しない場合はエラーメッセージを表示して終了する
- 最新のセッションが先に表示される（降順）
- 最大100セッション分の履歴を読み込んで判断する
- 各セッションのdisplayを最大5エントリ結合し、セッション全体の内容を把握する
- 判断に迷う場合は候補を多めに提示し、ユーザーに選んでもらう
