---
title: "Claude Codeのセッション残量、把握してますか？ statuslineに常時表示させたら快適だった"
emoji: "📊"
type: "tech"
topics: ["claudecode", "claude", "ccusage", "ai", "cli"]
published: true
---

## 始めに

Claude Codeのstatuslineはカスタマイズできます。[ccusage](https://github.com/syumai/ccusage)を始め、各ユーザーがオリジナルのstatuslineを公開しており、自分もccusageをメインに使っていました。

ccusageにはいろんな機能があり、最初はあれこれ表示しまくっていました。が、しばらく使ってみると、大半の情報はそこまで必要ないと気付きます。

一方で**一番知りたい「セッションの残量」が表示されない**。これが最大の不満でした。

Claude Code Teamで複数セッションを並列に走らせていると、5時間レート制限ウィンドウの残量把握が死活問題です。残り少ないまま重いタスクを投げると、レート制限で途中停止する。逆に残量が十分なら、もう1セッション立ち上げても大丈夫。この判断を常にやりたかったので、自作しました。

## 完成形

こうなります。

![statusline表示例](/images/022/statusline.png)

```text
📋 usage │ 🤖 Opus 4.6 │ 💵 $12.05 │ 🟠 残28.8%(2h 21m left) │ 🟢 CTX残64% │ 📂 ~/dotfiles/claude-code
```

| セグメント | データソース | 説明 |
| --- | --- | --- |
| `📋 usage` | stdin JSON `session_name` | `/rename` 済みの場合非表示。UUID時は表示 |
| `🤖 Opus 4.6` | stdin JSON `model.display_name` | 使用中のモデル名 |
| `💵 $12.05` | stdin JSON `cost.total_cost_usd` | セッション累計コスト（Bedrock時は非表示） |
| `🟠 残28.8%(2h 21m left)` | ccusageキャッシュ | 5時間レート制限ウィンドウの残量 |
| `🟢 CTX残64%` | stdin JSON `context_window.remaining_percentage` | コンテキストウィンドウ残量 |
| `📂 ~/dotfiles/claude-code` | stdin JSON `cwd` | 作業ディレクトリ |

残量に応じて絵文字の色が変わるので、数字を読まなくても状態が分かります。

| 残量 | 色分け |
| --- | --- |
| 61%〜 | 🟢 |
| 41〜60% | 🟡 |
| 21〜40% | 🟠 |
| 〜20% | 🔴 |

## セットアップ

2ステップで終わります。

### 1. スクリプトを配置

後述のコードを `~/.claude/statusline.sh` に保存します。パスは任意です。

### 2. settings.jsonに追加

`~/.claude/settings.json` に以下を追加します。

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline.sh"
  }
}
```

ccusageが未インストールの場合、セッション残のセグメントだけが省略され、他はそのまま動きます。

## コード全文

`~/.claude/statusline.sh`

```bash
#!/bin/bash

set -euo pipefail

command -v jq &>/dev/null || { echo "Missing: jq"; exit 1; }

# ─── Config ───

CCUSAGE_CACHE="/tmp/ccusage-cache.json"
CCUSAGE_TTL=300  # 5 minutes
TOKEN_LIMIT=43000000

# ─── Parse Claude input (single jq call) ───

claude_input=$(cat)
IFS='|' read -r session_name model_name cwd cost ctx_remaining <<< \
  "$(echo "$claude_input" | jq -r '[
    (.session_name // .session_id // ""),
    (.model.display_name // ""),
    (.cwd // ""),
    (.cost.total_cost_usd // 0 | tostring),
    (.context_window.remaining_percentage // 0 | tostring)
  ] | join("|")')"

cwd_str="${cwd/#$HOME/~}"

# ─── Rate limit remaining (non-blocking ccusage) ───

session_str_rate=""
if command -v ccusage &>/dev/null; then
  cache_age_ok=false
  if [ -f "$CCUSAGE_CACHE" ]; then
    now=$(date +%s)
    mtime=$(stat -c "%Y" "$CCUSAGE_CACHE" 2>/dev/null) && [ -n "$mtime" ] \
      || mtime=$(stat -f "%m" "$CCUSAGE_CACHE" 2>/dev/null) || mtime=0
    [ $((now - mtime)) -lt "$CCUSAGE_TTL" ] && cache_age_ok=true
  fi

  if [ -f "$CCUSAGE_CACHE" ]; then
    IFS='|' read -r total_tokens remaining_minutes <<< \
      "$(jq -r '([.blocks[] | select(.isActive == true)][0] // null) | if . == null then "|" else [(.totalTokens // 0 | tostring), (.projection.remainingMinutes // "" | tostring)] | join("|") end' "$CCUSAGE_CACHE" 2>/dev/null || echo "|")"

    if [ -n "$total_tokens" ] && [ "$total_tokens" != "0" ] && [ "$total_tokens" != "null" ]; then
      # Calculate remaining percentage: (1 - totalTokens/TOKEN_LIMIT) * 100
      remaining_pct=$(awk "BEGIN { printf \"%.1f\", (1 - $total_tokens / $TOKEN_LIMIT) * 100 }")
      rate_int=${remaining_pct%%.*}

      # Color indicator based on rate limit remaining
      if [ "$rate_int" -le 20 ]; then rate_ind="🔴"
      elif [ "$rate_int" -le 40 ]; then rate_ind="🟠"
      elif [ "$rate_int" -le 60 ]; then rate_ind="🟡"
      else rate_ind="🟢"
      fi

      rate_time=""
      if [ -n "$remaining_minutes" ] && [ "$remaining_minutes" != "null" ]; then
        int_min=${remaining_minutes%%.*}
        h=$((int_min / 60))
        m=$((int_min % 60))
        rate_time=$(printf "(%dh %dm left)" "$h" "$m")
      fi

      session_str_rate="${rate_ind} 残${remaining_pct}%${rate_time}"
    fi
  fi

  # Background refresh if cache missing or expired (never blocks)
  if ! $cache_age_ok; then
    (ccusage blocks --active --offline --json > "${CCUSAGE_CACHE}.tmp" 2>/dev/null \
      && mv "${CCUSAGE_CACHE}.tmp" "$CCUSAGE_CACHE") & disown 2>/dev/null
  fi
fi

# ─── Context window indicator ───

ctx_int=${ctx_remaining%%.*}
if [ "$ctx_int" -le 20 ]; then ctx_ind="🔴"
elif [ "$ctx_int" -le 40 ]; then ctx_ind="🟠"
elif [ "$ctx_int" -le 60 ]; then ctx_ind="🟡"
else ctx_ind="🟢"
fi
ctx_str="${ctx_ind} CTX残${ctx_remaining}%"

# ─── Output ───

# Session name: only show if still a UUID (reminder to rename)
session_str=""
if [[ "$session_name" =~ ^[0-9a-f]{8}-[0-9a-f]{4}- ]]; then
  session_str="📋 ${session_name:0:8}… │ "
fi

if [ "${CLAUDE_CODE_USE_BEDROCK:-0}" != "0" ]; then
  printf "%s🤖 %s │ %s │ 📂 %s" \
    "$session_str" "$model_name" "$ctx_str" "$cwd_str"
else
  # Build rate limit segment (only for non-Bedrock)
  rate_segment=""
  [ -n "$session_str_rate" ] && rate_segment=" │ $session_str_rate"

  printf "%s🤖 %s │ 💵 \$%.2f%s │ %s │ 📂 %s" \
    "$session_str" "$model_name" "$cost" "$rate_segment" "$ctx_str" "$cwd_str"
fi
```

## 処理の流れ

```text
Claude Code ──stdin JSON──→ statusline.sh ──→ tmux statusline
                                │
                                ├─ jq: JSON parse（1回の呼び出しで全フィールド抽出）
                                ├─ ccusage cache: /tmp/ccusage-cache.json（読み取りのみ）
                                └─ awk: 浮動小数点計算（残量%）
```

Claude Codeはstatuslineコマンドを呼ぶたびに、stdinにセッション情報のJSONを流します。モデル名、コスト、コンテキストウィンドウ残量はこのJSONから取れます。

セッション残量だけはClaude Codeが提供していないため、ccusageのキャッシュから取得しています。

## 設計判断

### Stale-While-Revalidateキャッシュ

ccusageの直接実行は約17秒かかります。statuslineは頻繁に呼ばれるので、毎回17秒待つわけにはいきません。

Webのキャッシュ戦略と同じ考え方で解決しました。

1. **キャッシュが有効（TTL 5分以内）**: そのまま使う
2. **キャッシュが期限切れ**: 古いデータを即座に返しつつ、バックグラウンドで更新
3. **キャッシュが存在しない**: セッション残セグメントを省略し、バックグラウンドで取得開始

```bash
# バックグラウンド更新（ブロックしない）
(ccusage blocks --active --offline --json > cache.tmp && mv cache.tmp cache) & disown
```

書き込み中にstatuslineが読み取ると壊れたJSONをつかむ可能性があります。そこでtmpファイルに書いてからmvする方式にしました。mvはatomicですので、中途半端な状態は発生しません。

### jqの1回呼び出しでまとめて抽出

stdinのJSONパースでjqを5回呼ぶと、プロセス起動のオーバーヘッドが積み重なります。`join("|")` でパイプ区切りの1行にまとめ、`IFS='|' read` で展開することで、jqの起動を1回に抑えています。

### セッション残量の計算

ccusageのキャッシュから `totalTokens`（現在のウィンドウで使用済みのトークン数）を読み、上限43,000,000との比率で残量%を算出しています。

```text
セッション残% = (1 - totalTokens / 43,000,000) × 100
```

:::message
5時間ウィンドウで使えるトークン量は公式に公開されていません。`TOKEN_LIMIT=43000000` は実際の使用量から推定した値です。プランごとに上限は異なりますし、Anthropic側でいつレートが変更されるかも分かりません。ご自身の環境に合った値へ変更してください。
:::

### セッション残 vs CTX残

この2つは別物です。

| | セッション残 | CTX残 |
| --- | --- | --- |
| 意味 | 5時間レート制限ウィンドウのトークン残量 | 現在の会話のコンテキストウィンドウ残量 |
| ソース | ccusageキャッシュ（外部ツール） | Claude Code stdin JSON |
| リセット | 5時間ウィンドウ終了時 | 新しい会話開始時 |
| 不在時 | セグメント自体を省略 | 常に表示 |

セッション残が減ると「今日のAPI利用枠がなくなりそう」、CTX残が減ると「この会話が長くなりすぎている」。対処法も違います。前者はタスクの優先度を見直す、後者は新しいセッションを立ち上げるか `/compact` する。両方見えるのが大事です。

## セッション名は非表示

`/rename` していないセッションのsession_nameはUUIDです。`/rename` でセッション名を変えた場合は右上に表示されるので、その場合はstatuslineから省いたほうがすっきりします。
そこで、session_nameがUUIDのままの場合のみ、先頭8文字を表示するようにしています。

### Bedrock対応

`CLAUDE_CODE_USE_BEDROCK` 環境変数が設定されている場合、コストとセッション残のセグメントを非表示にしています。Bedrockは課金体系が異なるためです。

## 終わりに

statuslineに必要なのは「今どれくらい使えるか」だけでした。ccusageのキャッシュとClaude Codeが渡すJSONを組み合わせれば、ブロッキングなしで残量が見えます。

`TOKEN_LIMIT` やカラー閾値は好みで調整してください。

## 宣伝

Claude CodeのCHANGELOGを日本語で随時ポストしているXアカウントを運用しています。アップデート情報をキャッチアップしたければぜひフォローしてください。

👉 [@CCChangelogJA](https://x.com/CCChangelogJA)
