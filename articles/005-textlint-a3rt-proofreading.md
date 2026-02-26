---
title: "textlint a3rt-proofreading-v2をリリースしました"
emoji: "✍️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "textlint", "a3rt" ]
published: true
---

このzenn執筆環境のtextlintをいろいろ導入している中で、リクルートが運営している文章校閲API `Proofreading API` の存在を知りました。

https://a3rt.recruit.co.jp/product/proofreadingAPI/

textlintには、すでにtextlint-rule-a3rt-proofreadingというプラグインがあるのですが、どうもAPIが変わってURLも変わり、今は動作しませんでした。

textlint pluginを作ってみたかった＋動かすためのAPIキーをtextlintrcに入れなければならなかったこともあり（さすがにAPIキーをGitに入れたくない）、新規にv2としてプラグインを作ったので手順などを書いておきます。

本プラグインのGitHubページはこちら。

https://github.com/shivase/textlint-rule-a3rt-proofreading-v2

## 先に残念なお話

実際にプラグインを作って文章にかませてみたのですが、このAPIはあまり賢くなくてよいサジェスションをしてくれません😭。

しかし外部APIを呼ぶプラグインでも、そんなに遅延を感じたりしないのでアリだなとは思いました。何か他におもしろいAPIないかなぁ。あったら教えてください！

本当はgingerを使った英語の添削textlintを作りたかったんですが、既存のプラグインは動かない＋そもそもGingerAPIがなくなっている？な感じでしょんぼりです。

## インストール方法

### APIキーの取得

まず実行するためには、Proofreading APIのAPIキーが必要ですので、次のページで発行してください。フォーム入力後、すぐにメールにてAPIキーが送られてきます。

https://a3rt.recruit.co.jp/product/proofreadingAPI/registered/

### プラグインのインストール

次のコマンドでプラグインを入れてください。

```bash
npm install textlint-rule-a3rt-proofreading-v2
```

### 設定ファイルの修正

`.textlintrc.json` に次のような感じで設定を追加してください。

```json
{
  "rules": {
    "a3rt-proofreading-v2": {
      "apiKey": "./key.yaml"
    }
  }
}
```

上記設定の場合、プロジェクトの直下に `key.yaml` をおき、次のフォーマットで取得したAPIキーを記入してください。

```yaml
version: 1
rules:
  apiKey: 'APIKEY'
```

直接実行するとこんな感じです。

```bash
yarn textlint --rule a3rt-proofreading-v2 hoge.txt
yarn run v1.22.19

./hoge.txt
  1:1  error  Suggest: 'ホ' => '「|、|の' (score:0.78)  a3rt-proofreading-v2
  1:2  error  Suggest: 'ゲ' => 'マ|ン|ル' (score:0.76)  a3rt-proofreading-v2
  1:3  error  Suggest: 'ホ' => 'リ|ル|・' (score:0.77)  a3rt-proofreading-v2
  1:4  error  Suggest: 'ゲ' => 'は|を|が' (score:0.78)  a3rt-proofreading-v2
  1:5  error  Suggest: 'ほ' => 'イ|ー|リ' (score:0.99)  a3rt-proofreading-v2
  1:6  error  文末が"。"で終わっていません。            ja-technical-writing/ja-no-mixed-period

✖ 6 problems (6 errors, 0 warnings)

error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

結果には `score` が一緒に表示されますが、これはProofreading APIが返す、どの程度推奨しているかを0〜1で表したものになります（1が推奨）。

### 感想

textlintのプラグインは思ったよりシンプルに作れるのでよい感じです。
ハマりどこが何個かありましたが、textlintのプラグイン開発HowToも別途記事にしていこうかなと考えています。

せっかくなので興味がある人はインストールし試してくれるとうれしいです！
