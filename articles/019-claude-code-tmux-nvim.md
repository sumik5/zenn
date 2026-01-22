---
title: "Claude CodeのプロンプトをNeovimで使う（tmux版)"
emoji: "🌐"
type: "tech"
topics: ["neovim", "fzf", "tmux","claude","claudecode"]
published: true
---

## はじめに

[Claude CodeのプロンプトをNeovimで快適に書く](https://zenn.dev/shisashi/articles/0ba22e272d6f2f)

ではzellijを使った方法が紹介されていましたが、私はtmux派なのでtmuxで同様のことができるように設定してみました。最近neovimにお引っ越ししたばかりなので、こんな感じででるのかってすごく参考になりました🙏

今までClaude Code上のプロンプトで直接文字を書くのが辛く、間違って送信してしまうことも多かったので、一旦メモ帳に書いてからというのをしていました。
しかしこれにより、Claude Codeから離れることなく、ポップアップで表示されるNeovim上でプロンプトを書いて送信できるようになったのでかなり快適になりました。

## 動作イメージ

![動作イメージ画像](/images/019/screenshot.gif)

## 前提条件

* tmux
* neovim
* fzf

## 設定

### neovim

lazy.nvimを使っている場合、以下のように設定します。
`fzf`自体は`mise`などでインストールし、そのPATHを指定する必要があるので、該当する部分は修正する必要があります。

`~/.config/nvim/lua/plugins/fzf.lua`

```lua
return {
  "ibhagwan/fzf-lua",
  dependencies = { "nvim-tree/nvim-web-devicons" },

  config = function()
    local fzf = require("fzf-lua")

    -- ★ PATH 問題対策：fzf_bin を固定する
    fzf.setup({
      fzf_bin = os.getenv("HOME") .. "/.local/share/mise/shims/fzf",
    })

    -- ★ markdown で @ → ファイル挿入
    vim.api.nvim_create_autocmd("FileType", {
      pattern = "markdown",
      callback = function(ev)
        vim.keymap.set("i", "@", function()
          local selected_path = nil
          fzf.files({
            file_icons = false,
            git_icons = false,
            actions = {
              ["default"] = function(selected, opts)
                if selected and selected[1] then
                  local file = require("fzf-lua.path").entry_to_file(selected[1], opts)
                  selected_path = file and file.path or nil
                end
              end
            },
            winopts = {
              on_close = function()
                vim.schedule(function()
                  if selected_path then
                    vim.api.nvim_put({ "@" .. selected_path }, "", false, true)
                  else
                    vim.api.nvim_put({ "@" }, "", false, true)
                  end
                  vim.defer_fn(function()
                    vim.cmd("startinsert!")
                  end, 10)
                end)
              end
            }
          })
        end, { buffer = ev.buf, noremap = true })
      end,
    })
  end
}
```

### tmux

`~/.tmux.conf` に以下を追加します。私は`bind e`に設定していますが、ここはお好みで変更してください。

```tmux
# Alt-e で popup エディタ起動（呼び出し元 pane の ID を渡す）
bind e run-shell "bash ~/.config/tmux/claude-prompt-edit.sh '#{pane_id}'"
```

`~/.config/tmux/claude-prompt-edit.sh` というスクリプトを作成し、以下の内容を記述します。

```bash
#!/usr/bin/env bash

# 呼び出し元 pane の ID（tmux.conf から渡す）
TARGET_PANE="$1"

# 一時ファイルを作成
TMPFILE="$(mktemp /tmp/claude-prompt-XXXXXX.md)"

# tmux セッションの中で動いているか確認（保険）
if [ -z "${TMUX:-}" ]; then
  echo "Error: This script must be run inside a tmux session." >&2
  exit 1
fi

# popup の中で「いつものシェル環境 + nvim」を起動
# -E: コマンド(nvim)が終わったら popup を閉じる
tmux display-popup -E -w 80% -h 70% -T "Claude Prompt" \
  "$SHELL -i -c 'nvim \"$TMPFILE\"'"

# ==== ここから先は nvim を閉じたあとに実行される ====

# デバッグ：TMPFILE のパスとサイズを tmux ステータスに表示
if [ -f "$TMPFILE" ]; then
  SIZE="$(wc -c < "$TMPFILE" 2>/dev/null || echo 0)"
  tmux display-message "claude-popup: file=$TMPFILE size=${SIZE}B"
else
  tmux display-message "claude-popup: TMPFILE not found: $TMPFILE"
fi

# 一時ファイルに中身があれば貼り付け
if [ -s "$TMPFILE" ]; then
  CONTENT="$(cat "$TMPFILE")"

  # バッファに入れる
  tmux set-buffer -b claude_prompt -- "$CONTENT"

  # ★ 呼び出し元 pane に明示的に貼り付ける ★
  tmux paste-buffer -b claude_prompt -t "$TARGET_PANE"
else
  tmux display-message "claude-popup: TMPFILE is empty, skip paste"
fi

# 後片付け
rm -f "$TMPFILE"

exit 0
```

## おわりに

neovimのpopupから戻った時に、Claude Code上に自動的にプロンプトが貼り付けられる挙動ではないので、少し気持ち悪いです。そのままEnter押すとちゃんと送信されるので、一応動作上は問題ないのですが、解決方法が分かる方がいらっしゃいましたら是非教えてください🙇
