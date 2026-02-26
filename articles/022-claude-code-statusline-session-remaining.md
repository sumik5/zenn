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

:::message
**2026-02-26追記**: セッション残量の取得元をccusageから **Anthropic OAuth Usage API** に切り替えました。ccusageのトークン集計ではAnthropicの実際のレート制限計算と乖離が生じ、残量がマイナスになったりリセット時刻がずれたりする問題がありました。OAuth Usage APIはAnthropicが提供する公式のエンドポイントで、正確な利用率（`utilization`）とリセット時刻（`resets_at`）を返します。ccusageはOAuth APIが利用できない環境向けのフォールバックとして残しています。
:::

## 完成形

こうなります。

![statusline表示例](/images/022/statusline.png)

```text
📋 usage │ 🤖 Opus 4.6 │ 💵 $12.05 │ 🟠 残28.8%(2h 21m left) │ 7d:残80.0% │ 🟢 CTX残64% │ 📂 ~/dotfiles/claude-code
```

| セグメント | データソース | 説明 |
| --- | --- | --- |
| `📋 usage` | stdin JSON `session_name` | `/rename` 済みの場合非表示。UUID時は表示 |
| `🤖 Opus 4.6` | stdin JSON `model.display_name` | 使用中のモデル名 |
| `💵 $12.05` | stdin JSON `cost.total_cost_usd` | セッション累計コスト（Bedrock時は非表示） |
| `🟠 残28.8%(2h 21m left)` | OAuth Usage API（ccusageフォールバック） | 5時間レート制限ウィンドウの残量 |
| `7d:残80.0%` | OAuth Usage API | 7日間レート制限の残量（OAuth使用時のみ） |
| `🟢 CTX残64%` | stdin JSON `context_window.remaining_percentage` | コンテキストウィンドウ残量 |
| `📂 ~/dotfiles/claude-code` | stdin JSON `cwd` | 作業ディレクトリ |

残量に応じて絵文字の色が変わるので、数字を読まなくても状態が分かります。

| 残量 | 色分け |
| --- | --- |
| 80%〜 | 🟢 |
| 60〜79% | 🟡 |
| 40〜59% | 🟠 |
| 〜39% | 🔴 |

## セットアップ

### 前提

- **jq**: JSONパースに使用
- **curl**: OAuth Usage APIの呼び出しに使用（macOSは標準搭載）
- **[ccusage](https://github.com/ryoppippi/ccusage)**（任意）: OAuth APIが使えない場合のフォールバック

```bash
# macOS
brew install jq

# ccusageも使う場合（任意）
brew install ccusage
```

:::message alert
OAuth Usage APIを使うには、Claude Codeのログインセッションに `user:profile` スコープが必要です。以前にログインしたまま使っている場合、このスコープが含まれていない可能性があります。

残量セグメントが表示されない場合は、以下の手順でログインし直してください。

```bash
claude auth logout
claude auth login
```

再ログインによる料金は発生しません。OAuth Usage APIは読み取り専用のアカウント情報エンドポイントです。
:::

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

## コード全文

`~/.claude/statusline.sh`

```bash
#!/bin/bash

set -euo pipefail

command -v jq &>/dev/null || { echo "Missing: jq"; exit 1; }

# ─── Config ───

OAUTH_CACHE="/tmp/oauth-usage-cache.json"
OAUTH_TTL=60  # 60 seconds
CCUSAGE_CACHE="/tmp/ccusage-cache.json"
CCUSAGE_TTL=300  # 5 minutes (fallback)
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

# ─── OAuth Usage API ───

# Get access token from macOS Keychain (handles both plain JSON and hex-encoded formats)
_get_access_token() {
  local raw
  raw=$(security find-generic-password -s "Claude Code-credentials" -w 2>/dev/null) || return 1
  [ -z "$raw" ] && return 1

  local json
  if [[ "$raw" == "{"* ]]; then
    # Plain JSON format (post re-login)
    json="$raw"
  else
    # Hex-encoded format (legacy)
    json=$(echo "$raw" | xxd -r -p 2>/dev/null) || return 1
  fi

  echo "$json" \
    | grep -o '"accessToken":"[^"]*"' \
    | head -1 \
    | sed 's/"accessToken":"//;s/"$//'
}

# Fetch OAuth usage and write to cache file
_fetch_oauth_usage() {
  local access_token
  access_token=$(_get_access_token) || return 1
  [ -z "$access_token" ] && return 1

  curl --silent --max-time 5 \
    --header "Authorization: Bearer ${access_token}" \
    --header "anthropic-beta: oauth-2025-04-20" \
    "https://api.anthropic.com/api/oauth/usage" 2>/dev/null
}

# Check if cache is fresh
_oauth_cache_fresh() {
  [ -f "$OAUTH_CACHE" ] || return 1
  local now mtime
  now=$(date +%s)
  mtime=$(stat -f "%m" "$OAUTH_CACHE" 2>/dev/null) \
    || mtime=$(stat -c "%Y" "$OAUTH_CACHE" 2>/dev/null) \
    || return 1
  [ $((now - mtime)) -lt "$OAUTH_TTL" ]
}

# Parse ISO8601 timestamp to epoch seconds (macOS + Linux compatible)
_iso8601_to_epoch() {
  local ts="$1"
  # Remove fractional seconds and Z, then convert
  local normalized
  normalized=$(echo "$ts" | sed 's/\.[0-9]*Z$/Z/; s/Z$/+00:00/')
  date -j -f "%Y-%m-%dT%H:%M:%S%z" "${normalized/+00:00/+0000}" "+%s" 2>/dev/null \
    || date -d "$ts" "+%s" 2>/dev/null \
    || echo ""
}

# Build rate indicator emoji based on remaining percentage
_rate_indicator() {
  local pct_int="$1"
  if [ "$pct_int" -ge 80 ]; then echo "🟢"
  elif [ "$pct_int" -ge 60 ]; then echo "🟡"
  elif [ "$pct_int" -ge 40 ]; then echo "🟠"
  else echo "🔴"
  fi
}

# Build rate limit string from OAuth cache
_build_oauth_rate_str() {
  [ -f "$OAUTH_CACHE" ] || return 1

  local five_util resets_at
  IFS='|' read -r five_util resets_at <<< \
    "$(jq -r '[
      (.five_hour.utilization // ""),
      (.five_hour.resets_at // "")
    ] | join("|")' "$OAUTH_CACHE" 2>/dev/null || echo "|")"

  [ -z "$five_util" ] && return 1

  # Remaining percentage = 100 - utilization
  local remaining_pct
  remaining_pct=$(awk "BEGIN { printf \"%.1f\", 100 - ${five_util} }")
  local remaining_int=${remaining_pct%%.*}
  local rate_ind
  rate_ind=$(_rate_indicator "$remaining_int")

  # Time remaining from resets_at
  local rate_time=""
  if [ -n "$resets_at" ]; then
    local reset_epoch now_epoch diff_sec
    reset_epoch=$(_iso8601_to_epoch "$resets_at")
    now_epoch=$(date +%s)
    if [ -n "$reset_epoch" ] && [ "$reset_epoch" -gt "$now_epoch" ]; then
      diff_sec=$((reset_epoch - now_epoch))
      local h=$((diff_sec / 3600))
      local m=$(((diff_sec % 3600) / 60))
      rate_time=$(printf "(%dh %dm left)" "$h" "$m")
    fi
  fi

  # 7-day segment
  local seven_util seven_str=""
  seven_util=$(jq -r '.seven_day.utilization // ""' "$OAUTH_CACHE" 2>/dev/null || echo "")
  if [ -n "$seven_util" ]; then
    local seven_remaining
    seven_remaining=$(awk "BEGIN { printf \"%.1f\", 100 - ${seven_util} }")
    seven_str=" │ 7d:残${seven_remaining}%"
  fi

  echo "${rate_ind} 残${remaining_pct}%${rate_time}${seven_str}"
}

# ─── Rate limit remaining (OAuth API with ccusage fallback) ───

session_str_rate=""

if [ "${CLAUDE_CODE_USE_BEDROCK:-0}" = "0" ]; then
  # Try OAuth API path
  if _oauth_cache_fresh; then
    # Cache is fresh: use it
    session_str_rate=$(_build_oauth_rate_str 2>/dev/null || echo "")
  else
    # Cache is stale or missing: use existing cache for display, refresh in background
    if [ -f "$OAUTH_CACHE" ]; then
      session_str_rate=$(_build_oauth_rate_str 2>/dev/null || echo "")
    fi
    # Background refresh (non-blocking)
    (
      result=$(_fetch_oauth_usage 2>/dev/null) || exit 0
      # Validate it looks like a usage response
      echo "$result" | jq -e '.five_hour' &>/dev/null || exit 0
      echo "$result" > "${OAUTH_CACHE}.tmp" \
        && mv "${OAUTH_CACHE}.tmp" "$OAUTH_CACHE"
    ) & disown 2>/dev/null
  fi

  # Fallback to ccusage if OAuth produced no output
  if [ -z "$session_str_rate" ] && command -v ccusage &>/dev/null; then
    cache_age_ok=false
    if [ -f "$CCUSAGE_CACHE" ]; then
      now=$(date +%s)
      mtime=$(stat -f "%m" "$CCUSAGE_CACHE" 2>/dev/null) \
        || mtime=$(stat -c "%Y" "$CCUSAGE_CACHE" 2>/dev/null) \
        || mtime=0
      has_data=$(jq -r '.blocks | length > 0' "$CCUSAGE_CACHE" 2>/dev/null || echo "false")
      local_ttl=$CCUSAGE_TTL
      [ "$has_data" = "true" ] || local_ttl=30
      [ $((now - mtime)) -lt "$local_ttl" ] && cache_age_ok=true
    fi

    if [ -f "$CCUSAGE_CACHE" ]; then
      IFS='|' read -r total_tokens remaining_minutes <<< \
        "$(jq -r '([.blocks[] | select(.isActive == true)][0] // null) | if . == null then "|" else [(.totalTokens // 0 | tostring), (.projection.remainingMinutes // "" | tostring)] | join("|") end' "$CCUSAGE_CACHE" 2>/dev/null || echo "|")"

      if [ -n "$total_tokens" ] && [ "$total_tokens" != "0" ] && [ "$total_tokens" != "null" ]; then
        remaining_pct=$(awk "BEGIN { printf \"%.1f\", (1 - $total_tokens / $TOKEN_LIMIT) * 100 }")
        rate_int=${remaining_pct%%.*}
        rate_ind=$(_rate_indicator "$rate_int")

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

    if ! $cache_age_ok; then
      (ccusage blocks --active --offline --json > "${CCUSAGE_CACHE}.tmp" 2>/dev/null \
        && mv "${CCUSAGE_CACHE}.tmp" "$CCUSAGE_CACHE") & disown 2>/dev/null
    fi
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
                                ├─ OAuth API cache: /tmp/oauth-usage-cache.json（優先）
                                ├─ ccusage cache: /tmp/ccusage-cache.json（フォールバック）
                                └─ awk: 浮動小数点計算（残量%）
```

Claude Codeはstatuslineコマンドを呼ぶたびに、stdinにセッション情報のJSONを流します。モデル名、コスト、コンテキストウィンドウ残量はこのJSONから取れます。

セッション残量だけはClaude Codeが提供していないため、外部から取得する必要があります。現在はAnthropicのOAuth Usage APIを優先し、利用できない場合はccusageのキャッシュにフォールバックしています。

## 設計判断

### ccusageからOAuth Usage APIへの移行

ccusageベースで運用していたところ、Webダッシュボードの表示と大きく乖離する場面が出てきました。statuslineでは `残-2.0%` と表示されるのに、Webでは9%使用済みと表示される。原因を調べたところ、2つの問題がありました。

#### 1. トークン集計の食い違い

ccusageの `totalTokens` は `inputTokens + outputTokens + cacheCreationInputTokens + cacheReadInputTokens` をそのまま合計しています。一方、Anthropicのレート制限計算では `cacheReadInputTokens` が大幅に割り引きされています（公式ドキュメントの料金表でも、キャッシュ読み取りは入力トークンの1/10の価格です）。

ヘビーに使うと `cacheReadInputTokens` が総トークンの大半を占めます。ccusageはこれを100%でカウントするため、実際の利用率よりも大幅に高い値になります。

#### 2. ブロック境界の食い違い

ccusageは `floorToHour(最初のエントリ時刻) + 5時間` でブロック境界を計算します。Anthropicの実際のレートウィンドウ開始時刻とは微妙にずれるため、リセット時刻が正確ではありません。

#### 解決: OAuth Usage API

OAuth Usage APIエンドポイント（`/api/oauth/usage`）なら、Anthropic自身が算出した正確な値を直接取得できます。

```json
{
  "five_hour": {
    "utilization": 29.0,
    "resets_at": "2026-02-26T10:00:00.717869+00:00"
  },
  "seven_day": {
    "utilization": 20.0,
    "resets_at": "2026-03-04T00:00:00.717889+00:00"
  }
}
```

`utilization` はパーセント値ですので、`100 - utilization` で残量が出ます。`resets_at` はISO 8601タイムスタンプですので、現在時刻との差分でリセットまでの残り時間が計算できます。トークン上限の推定もブロック境界の推定も不要になりました。

### キーチェインからのトークン抽出

OAuth Usage APIの呼び出しにはアクセストークンが必要です。Claude Codeはログイン時にトークンをmacOSキーチェインに保存しています。`security find-generic-password` コマンドで取得できます。

ただし、キーチェインに保存される形式がClaude Codeのバージョンやログインタイミングによって異なることが分かりました。

```bash
# パターン1: 生のJSON文字列（再ログイン後）
{"claudeAiOauth":{"accessToken":"sk-ant-oat01-...","refreshToken":"..."}}

# パターン2: 16進数エンコード（レガシー）
7b22636c617564654169...
```

そのため、取得した文字列が `{` で始まるかどうかで形式を判定し、16進数の場合は `xxd -r -p` でデコードしてからトークンを抽出しています。

### Stale-While-Revalidateキャッシュ

OAuth Usage APIの呼び出しは数秒かかる場合があります。statuslineは頻繁に呼ばれるので、毎回API呼び出しを待つわけにはいきません。

Webのキャッシュ戦略と同じ考え方で解決しました。

1. **キャッシュが有効（TTL内）**: そのまま使う
2. **キャッシュが期限切れ**: 古いデータを即座に返しつつ、バックグラウンドで更新
3. **キャッシュが存在しない**: セッション残セグメントを省略し、バックグラウンドで取得開始

OAuth APIキャッシュのTTLは60秒にしています。APIの値自体がリアルタイムに変動するため、ccusageの5分より短めです。

```bash
# バックグラウンド更新（ブロックしない）
(
  result=$(_fetch_oauth_usage 2>/dev/null) || exit 0
  echo "$result" | jq -e '.five_hour' &>/dev/null || exit 0
  echo "$result" > "${OAUTH_CACHE}.tmp" && mv "${OAUTH_CACHE}.tmp" "$OAUTH_CACHE"
) & disown
```

レスポンスの検証（`jq -e '.five_hour'`）を挟んでいるのは、認証エラーやネットワーク障害時にキャッシュを壊さないためです。書き込みは旧バージョンと同様にtmpファイル経由のatomic mvです。

### jqの1回呼び出しでまとめて抽出

stdinのJSONパースでjqを5回呼ぶと、プロセス起動のオーバーヘッドが積み重なります。`join("|")` でパイプ区切りの1行にまとめ、`IFS='|' read` で展開することで、jqの起動を1回に抑えています。

### セッション残量の計算

OAuth Usage APIを使う場合、計算は単純です。

```text
セッション残% = 100 - utilization
リセットまで = resets_at - now
```

APIが `utilization: 29.0` を返せば、残量は71.0%です。`resets_at` のISO 8601タイムスタンプから現在時刻を引けば、リセットまでの残り時間が出ます。

ccusageフォールバックの場合は、従来どおりトークン使用量と上限の比率で計算します。

```text
ccusageフォールバック: セッション残% = (1 - totalTokens / TOKEN_LIMIT) × 100
```

:::message
ccusageフォールバック時の `TOKEN_LIMIT=43000000` は実際の使用量から推定した値です。プランごとに上限は異なりますし、Anthropic側でレートが変更される可能性もあります。OAuth Usage APIが利用できる環境では、この推定値は使われません。
:::

### セッション残 vs CTX残

この2つは別物です。

| | セッション残 | CTX残 |
| --- | --- | --- |
| 意味 | 5時間レート制限ウィンドウのトークン残量 | 現在の会話のコンテキストウィンドウ残量 |
| ソース | OAuth Usage API（ccusageフォールバック） | Claude Code stdin JSON |
| リセット | 5時間ウィンドウ終了時 | 新しい会話開始時 |
| 不在時 | セグメント自体を省略 | 常に表示 |

セッション残が減ると「今日のAPI利用枠がなくなりそう」、CTX残が減ると「この会話が長くなりすぎている」。対処法も違います。前者はタスクの優先度を見直す、後者は新しいセッションを立ち上げるか `/compact` する。両方見えるのが大事です。

## セッション名は非表示

`/rename` していないセッションのsession_nameはUUIDです。`/rename` でセッション名を変えた場合は右上に表示されるので、その場合はstatuslineから省いたほうがすっきりします。
そこで、session_nameがUUIDのままの場合のみ、先頭8文字を表示するようにしています。

### Bedrock対応

`CLAUDE_CODE_USE_BEDROCK` 環境変数が設定されている場合、コストとセッション残のセグメントを非表示にしています。Bedrockは課金体系が異なるためです。

## 終わりに

statuslineに必要なのは「今どれくらい使えるか」だけでした。OAuth Usage APIを使えば、Anthropicが計算した正確な利用率とリセット時刻がそのまま取れます。ccusageのトークン推定で生じていた乖離も解消しました。

OAuth APIが利用できない環境ではccusageにフォールバックするので、既存ユーザーの環境も壊れません。

## 宣伝

Claude CodeのCHANGELOGを日本語で随時ポストしているXアカウントを運用しています。アップデート情報をキャッチアップしたければぜひフォローしてください。

👉 [@CCChangelogJA](https://x.com/CCChangelogJA)
