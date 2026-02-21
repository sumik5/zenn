---
title: "Claude Codeのhook通知、イベントごとに音を変えたら快適だった"
emoji: "🔔"
type: "tech"
topics: ["claudecode", "claude", "macos"]
published: true
---

## はじめに

Claude CodeのAgent Team APIでは、メインセッション・サブエージェント・チームメイトが同時に動きます。通知が鳴っても**どのエージェントの話なのか分からない**のが地味にストレスでした。tmuxで複数セッションを走らせているとなおさらです。

hookイベントごとに通知音とアイコンを変えたところ、画面を見なくても判別できるようになりました。その設定を紹介します。

## 通知の全体像

設定後はこうなります。

| hookイベント | スクリプト | 通知音 | アイコン | グループID |
| --- | --- | --- | --- | --- |
| Stop | notify-complete.sh | Glass | ToolbarInfo | claude-code-stop |
| SubagentStop | notify-subagent-stop.sh | Pop | ToolbarAdvanced | claude-code-subagent |
| TeammateIdle | notify-teammate-idle.sh | Tink | GroupIcon | claude-code-teammate |
| Notification | notify-waiting.sh | Glass | (なし) | (なし) |

Glass、Pop、Tinkと音色が違うので、画面を見なくてもどのイベントか分かります。

## 前提条件

`terminal-notifier` を入れると、グループIDによる通知のまとめ表示やアイコン指定が使えます。

```bash
brew install terminal-notifier
```

なくても `osascript` で通知自体は動きます。ただしグループIDとアイコンは効きません。

## hookイベントの種類

使うhookイベントは4つです。

| hookイベント | 発火タイミング |
| --- | --- |
| `Stop` | メインセッション完了時 |
| `SubagentStop` | Task toolで起動したサブエージェント終了時 |
| `TeammateIdle` | TeamCreateで起動したチームメイトがアイドル状態になった時 |
| `Notification` | Claude Codeが入力待ち状態になった時 |

## ファイル構成

```text
~/.claude/
├── settings.json          # hookの設定
└── hooks/
    ├── notify-complete.sh         # Stop hook
    ├── notify-subagent-stop.sh    # SubagentStop hook
    ├── notify-teammate-idle.sh    # TeammateIdle hook
    └── notify-waiting.sh          # Notification hook
```

## settings.jsonの設定

`~/.claude/settings.json` のhooksセクションに追加します。

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$HOME\"/.claude/hooks/notify-waiting.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$HOME\"/.claude/hooks/notify-complete.sh"
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$HOME\"/.claude/hooks/notify-subagent-stop.sh"
          }
        ]
      }
    ],
    "TeammateIdle": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$HOME\"/.claude/hooks/notify-teammate-idle.sh"
          }
        ]
      }
    ]
  }
}
```

## 各スクリプト

4つのスクリプトは共通の構造です。

1. stdinからhookのJSONデータを読む
2. jq（なければpython3）でパースする
3. 通知メッセージを組み立てる
4. terminal-notifier（なければosascript）で通知する

違いは「JSONから何を取り出すか」と「通知音・アイコンの設定」だけです。

### notify-complete.sh（Stop hook）

メインセッション完了時に発火します。プロジェクト名をサブタイトル、最後のアシスタントメッセージの先頭行を本文に表示します。

冒頭の `stop_hook_active` チェックが重要です。これを省くと、通知スクリプト自体がStopイベントを再発火させて無限ループに陥ります。

```bash
#!/bin/bash
set -euo pipefail

# stdinからStop hookデータを読み込む
HOOK_JSON=$(cat)

# JSONパース（jq優先、python3フォールバック）
if command -v jq &>/dev/null; then
    STOP_HOOK_ACTIVE=$(echo "$HOOK_JSON" | jq -r '.stop_hook_active // false')
    LAST_MESSAGE=$(echo "$HOOK_JSON" | jq -r '.last_assistant_message // ""')
else
    STOP_HOOK_ACTIVE=$(echo "$HOOK_JSON" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('stop_hook_active', False))" 2>/dev/null || echo "False")
    LAST_MESSAGE=$(echo "$HOOK_JSON" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('last_assistant_message', ''))" 2>/dev/null || echo "")
fi

# stop_hook_active が true の場合はスキップ（無限ループ防止）
if [ "$STOP_HOOK_ACTIVE" = "true" ] || [ "$STOP_HOOK_ACTIVE" = "True" ]; then
    exit 0
fi

# プロジェクト名
PROJECT_NAME=$(basename "$PWD")

# last_assistant_message の先頭の非空行を取得し、100文字に切り詰め
if [ -n "$LAST_MESSAGE" ]; then
    FIRST_LINE=$(echo "$LAST_MESSAGE" | grep -m1 -v '^[[:space:]]*$' || true)
    if [ -n "$FIRST_LINE" ]; then
        BODY="${FIRST_LINE:0:100}"
    else
        BODY="作業完了"
    fi
else
    BODY="作業完了"
fi

# サニタイズ（ダブルクォート・バックスラッシュ・改行を除去）
sanitize() {
    echo "$1" | tr -d '"\\' | tr '\n\r' ' '
}

# 通知設定
ICON_BASE="/System/Library/CoreServices/CoreTypes.bundle/Contents/Resources"
TITLE_SAFE=$(sanitize "Claude Code")
SUBTITLE_SAFE=$(sanitize "$PROJECT_NAME")
MESSAGE_SAFE=$(sanitize "$BODY")
SOUND="Glass"
ICON="${ICON_BASE}/ToolbarInfo.icns"
GROUP="claude-code-stop"

# terminal-notifier（優先）
if command -v terminal-notifier &>/dev/null; then
    terminal-notifier \
        -title "$TITLE_SAFE" \
        -subtitle "$SUBTITLE_SAFE" \
        -message "$MESSAGE_SAFE" \
        -sound "$SOUND" \
        -contentImage "$ICON" \
        -group "$GROUP"
else
    # osascriptフォールバック
    osascript <<EOF 2>/dev/null || true
display notification "${MESSAGE_SAFE}" \
    with title "${TITLE_SAFE}" \
    subtitle "${SUBTITLE_SAFE}" \
    sound name "${SOUND}"
EOF
fi

echo "通知完了: ${PROJECT_NAME}"
```

### notify-subagent-stop.sh（SubagentStop hook）

サブエージェントの終了時に発火します。サブタイトルに `agent_type` を表示し、通知音を「Pop」にしてStopの「Glass」と区別します。

```bash
#!/bin/bash
set -euo pipefail

# stdinからSubagentStop hookデータを読み込む
HOOK_JSON=$(cat)

# JSONパース（jq優先、python3フォールバック）
if command -v jq &>/dev/null; then
    AGENT_TYPE=$(echo "$HOOK_JSON" | jq -r '.agent_type // ""')
    LAST_MESSAGE=$(echo "$HOOK_JSON" | jq -r '.last_assistant_message // ""')
else
    AGENT_TYPE=$(echo "$HOOK_JSON" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('agent_type', ''))" 2>/dev/null || echo "")
    LAST_MESSAGE=$(echo "$HOOK_JSON" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('last_assistant_message', ''))" 2>/dev/null || echo "")
fi

# last_assistant_message の先頭の非空行を取得し、100文字に切り詰め
if [ -n "$LAST_MESSAGE" ]; then
    FIRST_LINE=$(echo "$LAST_MESSAGE" | grep -m1 -v '^[[:space:]]*$' || true)
    if [ -n "$FIRST_LINE" ]; then
        BODY="${FIRST_LINE:0:100}"
    else
        BODY="完了"
    fi
else
    BODY="完了"
fi

# サニタイズ（ダブルクォート・バックスラッシュ・改行を除去）
sanitize() {
    echo "$1" | tr -d '"\\' | tr '\n\r' ' '
}

# 通知設定
ICON_BASE="/System/Library/CoreServices/CoreTypes.bundle/Contents/Resources"
TITLE_SAFE=$(sanitize "Claude Code")
SUBTITLE_SAFE=$(sanitize "$AGENT_TYPE")
MESSAGE_SAFE=$(sanitize "$BODY")
SOUND="Pop"
ICON="${ICON_BASE}/ToolbarAdvanced.icns"
GROUP="claude-code-subagent"

# terminal-notifier（優先）
if command -v terminal-notifier &>/dev/null; then
    terminal-notifier \
        -title "$TITLE_SAFE" \
        -subtitle "$SUBTITLE_SAFE" \
        -message "$MESSAGE_SAFE" \
        -sound "$SOUND" \
        -contentImage "$ICON" \
        -group "$GROUP"
else
    # osascriptフォールバック
    osascript <<EOF 2>/dev/null || true
display notification "${MESSAGE_SAFE}" \
    with title "${TITLE_SAFE}" \
    subtitle "${SUBTITLE_SAFE}" \
    sound name "${SOUND}"
EOF
fi

echo "通知完了: ${AGENT_TYPE}"
```

### notify-teammate-idle.sh（TeammateIdle hook）

チームメイトがアイドル状態になったときに発火します。`teammate_name` と `team_name` を表示するので、どのチームの誰が待機中か一目で分かります。

```bash
#!/bin/bash
set -euo pipefail

# stdinからTeammateIdle hookデータを読み込む
HOOK_JSON=$(cat)

# JSONパース（jq優先、python3フォールバック）
if command -v jq &>/dev/null; then
    TEAMMATE_NAME=$(echo "$HOOK_JSON" | jq -r '.teammate_name // ""')
    TEAM_NAME=$(echo "$HOOK_JSON" | jq -r '.team_name // ""')
else
    TEAMMATE_NAME=$(echo "$HOOK_JSON" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('teammate_name', ''))" 2>/dev/null || echo "")
    TEAM_NAME=$(echo "$HOOK_JSON" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('team_name', ''))" 2>/dev/null || echo "")
fi

# サニタイズ（ダブルクォート・バックスラッシュ・改行を除去）
sanitize() {
    echo "$1" | tr -d '"\\' | tr '\n\r' ' '
}

# 通知設定
ICON_BASE="/System/Library/CoreServices/CoreTypes.bundle/Contents/Resources"
TITLE_SAFE=$(sanitize "Claude Code")
SUBTITLE_SAFE=$(sanitize "$TEAM_NAME")
MESSAGE_SAFE=$(sanitize "${TEAMMATE_NAME} が待機中")
SOUND="Tink"
ICON="${ICON_BASE}/GroupIcon.icns"
GROUP="claude-code-teammate"

# terminal-notifier（優先）
if command -v terminal-notifier &>/dev/null; then
    terminal-notifier \
        -title "$TITLE_SAFE" \
        -subtitle "$SUBTITLE_SAFE" \
        -message "$MESSAGE_SAFE" \
        -sound "$SOUND" \
        -contentImage "$ICON" \
        -group "$GROUP"
else
    # osascriptフォールバック
    osascript <<EOF 2>/dev/null || true
display notification "${MESSAGE_SAFE}" \
    with title "${TITLE_SAFE}" \
    subtitle "${SUBTITLE_SAFE}" \
    sound name "${SOUND}"
EOF
fi

echo "通知完了: ${TEAMMATE_NAME} (${TEAM_NAME})"
```

### notify-waiting.sh（Notification hook）

入力待ち状態で発火します。他の3つと違い、terminal-notifierではなくosascriptのみで実装しています。通知音の候補をコメントに列挙してあるので、好みで変えてください。

```bash
#!/bin/bash
set -euo pipefail

# プロジェクト名を取得
PROJECT_NAME=$(basename "$PWD")
MESSAGE="${PROJECT_NAME}、入力待ち"

# 設定
NOTIFICATION_SOUND="Glass"  # 通知音を変更する場合はここを編集
DISPLAY_DURATION=5          # 通知の表示時間（秒）

# === 利用可能な通知音 ===
# Basso      - 低音
# Blow       - 風の音
# Bottle     - ボトル音
# Frog       - カエルの鳴き声
# Funk       - ファンキーな音
# Glass      - ガラス音（デフォルト）
# Hero       - ヒーロー音
# Morse      - モールス信号
# Ping       - シンプルなピン音
# Pop        - ポップ音
# Purr       - 猫の喉鳴らし
# Sosumi     - クラシックなMac音
# Submarine  - 潜水艦音
# Tink       - 金属音
# ※ 通知音を無効にする場合は NOTIFICATION_SOUND="" に設定

# 通知センターに表示（表示時間付き）
if ! osascript <<EOF 2>/dev/null
display notification "${MESSAGE}" \
    with title "Claude Code" \
    subtitle "プロジェクト: ${PROJECT_NAME}" \
    sound name "${NOTIFICATION_SOUND}"

-- 指定秒数待機（通知を表示し続ける）
delay ${DISPLAY_DURATION}
EOF
then
    # osascriptが失敗した場合は音声のみ
    echo "通知の表示に失敗しましたが、音声通知は実行されました" >&2
fi

# 音声通知の完了を待つ
wait

echo "通知完了: ${PROJECT_NAME}"
```

## 設計のポイント

### グループIDで通知をまとめる

terminal-notifierの `-group` オプションで、通知センター上をイベント種別ごとにグルーピングできます。同じグループIDの通知は新しいもので上書きされるため、通知センターがあふれません。

### `stop_hook_active` による無限ループ防止

Stop hookのスクリプトが完了すると、それ自体がStopイベントを再トリガーします。Claude Codeはこの再帰呼び出し時に `stop_hook_active: true` を送るので、先頭で必ずチェックしてください。

### jqなしでも動くフォールバック

jqが入っていない環境向けにpython3をフォールバックにしています。macOSにはpython3が標準搭載されているので、追加インストールなしで動きます。

### サニタイズの必要性

通知メッセージにダブルクォートやバックスラッシュが混じると、`osascript` や `terminal-notifier` の引数解析が壊れます。`tr` で事前に除去して表示崩れを防いでいます。

## おわりに

hookを種類ごとに切り分けるだけで、並列開発中の「今の通知、どれ？」がなくなりました。

hookイベントは今回の4種類以外にも `SessionStart` や `PostToolUse` があります。作業開始時の通知や特定ツール使用後の後処理にも使えるので、試してみてください。
