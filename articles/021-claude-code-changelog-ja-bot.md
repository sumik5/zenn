---
title: "Claude Codeの更新、追えてますか？ 日本語CHANGELOG自動投稿Botを作った"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "claude", "githubactions"]
published: true
---

## 始めに

Claude Codeは週に何回もバージョンが上がる。リリースノートは英語のみ。気付いたら知らない機能が増えていた、CLAUDE.mdの設定が古くなっていた。そんな経験はないでしょうか。

自分がまさにそうだったので、CHANGELOGを日本語化してXに自動投稿するBotを作りました。

**[@CCChangelogJA](https://x.com/CCChangelogJA)**

![投稿例](/images/021/post.png)

ソースコードは公開しています。

https://github.com/sumik5/claude-code-changelog-bot

英語のCHANGELOG Bot [@ClaudeCodeLog](https://x.com/ClaudeCodeLog)（[GitHub](https://github.com/marckrenn/claude-code-changelog)）を参考にさせていただきました。最初は自分用のWebページで済ませるつもりでしたが、せっかくなのでXへの投稿にしました。

このBotの特徴は**2つの情報源を組み合わせている**点です。公式リリースノートに加えて、システムプロンプトの内部変更まで追えます。「Task toolに `isolation: "worktree"` パラメーターが追加された」のような、リリースノートに載らない変更も拾えるのが売りです。

## 全体フロー

GitHub Actionsのcronで30分ごとに動きます。

```text
GitHub Actions (30分ごと or 手動トリガー)
  │
  ├─ Step 1: state.json 読み込み（前回投稿済みバージョン）
  ├─ Step 2: GitHub Releases API で最新バージョン取得
  ├─ Step 3: 新バージョン？ → No → 終了
  ├─ Step 4: cchistory でシステムプロンプト抽出 + 前バージョンとの diff 生成
  ├─ Step 5: Claude API で日本語ツイートスレッド生成
  ├─ Step 6: X API でスレッド投稿
  └─ Step 7: state.json 更新 + prompts/ キャッシュ保存 → git commit & push
```

依存パッケージは3つだけ。 `@anthropic-ai/sdk` で翻訳、 `twitter-api-v2` でX投稿、 `@mariozechner/cchistory` でシステムプロンプト抽出。

Step 4のcchistoryは実行のたびにnpmパッケージをダウンロードするため、1回数十秒かかります。抽出済みのプロンプトは `prompts/{version}.md` にキャッシュし、Step 7で `state.json` と一緒にGit commit & pushします。

## データソース：2段構成

### 公式リリースノート

`anthropics/claude-code` のGitHub Releases APIから取得する "What's changed" リスト。PRリンク集で、ユーザーから見える変更点の一覧です。

### システムプロンプト diff

[@mariozechner/cchistory](https://www.npmjs.com/package/@mariozechner/cchistory) というCLIツールで、Claude Codeのnpmパッケージからシステムプロンプトとツール定義を直接抽出します。2バージョン間の全文をClaude APIに渡し、差分を解析させるしくみです。

リリースノートには載らない内部変更を拾えるのが、この2段構成の強みです。

## ツイート生成のプロンプト設計

`translator.ts` の `buildPrompt()` で、Claudeにスレッド構成を細かく指示しています。

| ツイート | 内容 | 備考 |
| --- | --- | --- |
| 1（概要） | バージョン番号 + 主な変更点3〜5項目 + リリースURL | URLカード表示のためURLはツイート末尾 |
| 2（新機能） | 全新機能をまとめて1ツイート | サブカテゴリで整理可 |
| 3（修正・改善） | バグ修正 + パフォーマンス改善 | — |
| 4（プロンプト変更） | システムプロンプトdiffの解説 | diffがある場合のみ |

X Premiumでは1ツイート最大10,000文字のため、カテゴリごとの分割は不要です。

## 作ってみて

### diffをライブラリでなくClaudeに丸投げした

最初は `diff` パッケージのような行単位diffライブラリでプロンプト差分を出そうとしました。でもすぐにやめました。行単位diffは「何が変わったか」は教えてくれますが、「それが何を意味するか」は教えてくれません。

結局、両バージョンの全文をそのままClaude APIへ渡す設計にしています。

システムプロンプトは数万トークン規模ですが、Claudeの入力コンテキストには十分収まる量。依存パッケージも増えず、コードもシンプルになりました。「diffの計算」と「diffの意味解釈」を分けず、LLMへ一括で任せる、という割り切りです。

### `execSync` ではなく `execFileSync` を使う

cchistoryを子プロセスで呼ぶ際、地味に大事な判断でした。

```typescript
execFileSync("npx", ["--yes", "@mariozechner/cchistory", npmVersion], {
  cwd,
  stdio: "pipe",
  timeout: 300_000,
});
```

`execSync` はシェル経由で実行するため、バージョン文字列に `; rm -rf /` が紛れ込むとそのまま実行されます。 `execFileSync` なら引数を配列で渡すので、シェル展開が起きません。GitHub Releases APIの戻り値をそのまま渡している以上、念のための防御です。GitHub Actions上でsecretsを扱うBotだからこそ、この一手間は省けませんでした。

### Claudeが7〜8件のツイートを量産してきた

一番手を焼いたのがこれです。初期実装でClaudeにツイート生成を頼んだら、バグ修正だけで「（1/2）」「（2/2）」と分割して、計7〜8件のスレッドが出力されました。X Premiumなら1ツイート10,000文字使えるので分割の必要はないのですが、Claudeはそんな事情を知りません。

プロンプトに「絶対に同カテゴリを分割しない」「ツイート数の目標は2〜4件」と制約を追加し、 `MAX_TWEETS` のデフォルトを4に下げて対応しました。

## 終わりに

公式が明示しない内部変更まで拾えるので、Claude Codeを日常的に使っている人には役立つはずです。

ぜひフォローをお願いします。

**[@CCChangelogJA](https://x.com/CCChangelogJA)**
