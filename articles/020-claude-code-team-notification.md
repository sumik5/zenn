---
title: "Claude Code Teamの終了通知をイベントごとに切り分ける"
emoji: "🔔"
type: "tech"
topics: ["claudecode", "claude", "macos"]
published: true
---

## はじめに

Claude Code TeamのAgent Team APIを使うと、メインセッション、Task toolで起動したサブエージェント、TeamCreateで起動したチームメイトが同時に動きます。便利なんですが、通知が鳴っても**どのエージェントの話か分からない**のが地味にストレスでした。tmuxで複数セッションを走らせていればなおさらです。

hookイベントごとに通知音とアイコンを変えたところ、音だけでどれが終わったか分かるようになったので、その設定方法を共有します。

## hookイベントの種類

使うhookイベントは4つです。

| hookイベント | 発火タイミング |
|---|---|
| `Stop` | メインセッション完了時 |
| `SubagentStop` | Task toolで起動したサブエージェント終了時 |
| `TeammateIdle` | TeamCreateで起動したチームメイトがアイドル状態になった時 |
| `Notification` | Claude Codeが入力待ち状態になった時 |

それぞれ別のスクリプトを割り当てれば、通知音だけでどのイベントか判別できます。

## 全体構成

```
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

## 各スクリプトの解説

### notify-complete.sh（Stop hook）

メインセッション完了時に発火します。プロジェクト名をサブタイトル、最後のアシスタントメッセージの先頭行を本文に表示します。

ポイントは `stop_hook_active` フラグのチェックです。これを省くと通知処理自体がStopイベントをトリガーして無限ループになります。

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

# サニタイズ関数（ダブルクォート・バックスラッシュ・改行を除去）
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

Task toolで起動したサブエージェントの終了時に発火します。サブタイトルに `agent_type` を表示し、通知音を「Pop」にしてStopの「Glass」と区別しています。

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

# サニタイズ関数（ダブルクォート・バックスラッシュ・改行を除去）
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

# サニタイズ関数（ダブルクォート・バックスラッシュ・改行を除去）
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

入力待ち状態で発火します。グループIDは設定せず、シンプルに通知だけ飛ばしています。コメントに通知音の候補を列挙してあるので、好みで変えてみてください。

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

## 通知の使い分けまとめ

| hookイベント | スクリプト | 通知音 | アイコン | グループID | 用途 |
|---|---|---|---|---|---|
| Stop | notify-complete.sh | Glass | ToolbarInfo | claude-code-stop | メインセッション完了 |
| SubagentStop | notify-subagent-stop.sh | Pop | ToolbarAdvanced | claude-code-subagent | サブエージェント終了 |
| TeammateIdle | notify-teammate-idle.sh | Tink | GroupIcon | claude-code-teammate | チームメイト待機 |
| Notification | notify-waiting.sh | Glass | (なし) | (なし) | 入力待ち |

## 前提条件

`terminal-notifier` を入れるとグループIDによる通知のグルーピングが使えます。

```bash
brew install terminal-notifier
```

なくても `osascript` で通知自体は動きます。ただしグループIDとアイコン指定は効きません。

## 設計のポイント

**通知音の使い分け**

Glass、Pop、Tinkと音を変えておくと、画面を見なくてもどのイベントか分かります。実際に使ってみると、耳だけで判別できるのが想像以上に楽でした。

**グループIDでまとめる**

terminal-notifierの `-group` オプションで、通知センター上をイベント種別ごとにグループ化できます。同じグループIDの新しい通知が来ると前のを上書きするので、通知センターがあふれません。

**`stop_hook_active` で無限ループを防ぐ**

Stopフックのスクリプトがそれ自体Stopイベントをトリガーすると無限ループになります。Claude Codeは `stop_hook_active: true` を送ってくるので、必ずチェックしてください。

**jqがなくてもpython3で動く**

jqが入っていない環境向けにpython3をフォールバックにしています。macOSにはpython3が標準で入っているので、大抵の環境はこれでカバーできます。

**サニタイズ処理**

通知メッセージにダブルクォートやバックスラッシュが混じると `osascript` や `terminal-notifier` の引数解析が壊れます。`tr` で除去して表示崩れを防いでいます。

## おわりに

hookを種類ごとに切り分けるだけで、並列開発中の通知ストレスがほぼなくなりました。

hookイベントは今回の4種類以外にも `SessionStart` や `PostToolUse` があります。作業開始時に音を鳴らしたり、特定ツールの使用後に処理を挟んだりもできるので、好みに合わせて試してみてください。
