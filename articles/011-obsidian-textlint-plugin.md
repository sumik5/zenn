---
title: "Obisidian Textlint pluginを作りました"
emoji: "📔"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [ 'obsidian', 'textlint' ]
published: true
---

みなさんObsidianは使っていますか？　obisidianはMarkdownなメモアプリケーションですが、textlintが動けばなぁと思っていた人に朗報？です。

textlintでの文章校正が当たり前になってしまうと、自動校正してくれないソフトに乗り換えるのはどうしても考えてしまうのですが、今後obsidianを使ってみたかったのでプラグインを作成しました。

[shivase/obsidian-textlint](https://github.com/shivase/obsidian-textlint)

![動作イメージ](/images/011/sample-image.png)

仕事ではNotionを使っているので、そちらのchrome拡張とどちらを作ろうか悩んで、Obsdianはpluginで気軽に作れそうだったのでtexlint開発の練習で作った次第です。

まだ知ったばかりで全然初心者の域を出ていないため、機能として不十分なところが多々あると思いますがご了承ください。

使ってみて、こういう機能があるとよいなだったり、こうしてほしいなどありましたら、[issue](https://github.com/shivase/obsidian-textlint/issues)や[Discussion](https://github.com/shivase/obsidian-textlint/discussions)に気軽にコメントをください。

## 機能の紹介

機能としてはtextlintが吐き出すエラーを表示するだけになっています。推奨値への変更やエラーを無視するなどの対応はまだできていません。

### インストールしているtextlint plugin達

GitHubのREADMEにも書いていますが、現在以下がデフォルトで動作するようになっています。
これは今この記事を執筆しているvisual studio codeの環境で利用しているプラグインをそのまま導入したものになります（一部ローカルファイルを使用するプラグインは動作しなかったので除外）。

- [@textlint-ja/no-synonyms](https://github.com/textlint-ja/textlint-rule-no-synonyms)
- [@textlint-ja/textlint-rule-no-dropping-i](https://github.com/textlint-ja/textlint-rule-no-dropping-i)
- [@textlint-ja/textlint-rule-no-insert-dropping-sa](https://github.com/textlint-ja/textlint-rule-no-insert-dropping-sa)
- [abbr-within-parentheses](https://github.com/azu/textlint-rule-abbr-within-parentheses)
- [footnote-order](https://github.com/textlint-rule/textlint-rule-footnote-order)
- [ja-hiragana-keishikimeishi](https://github.com/lostandfound/textlint-rule-ja-hiragana-keishikimeishi)
- [no-mixed-zenkaku-and-hankaku-alphabet](https://github.com/textlint-ja/textlint-rule-no-mixed-zenkaku-and-hankaku-alphabet)
- [period-in-list-item](https://github.com/textlint-rule/textlint-rule-period-in-list-item)
- [prefer-tari-tari](https://github.com/textlint-ja/textlint-rule-prefer-tari-tari)
- [preset-ja-spacing](https://github.com/textlint-ja/textlint-rule-preset-ja-spacing)
- [preset-ja-technical-writing](https://github.com/textlint-ja/textlint-rule-preset-ja-technical-writing)
- [preset-jtf-style](https://github.com/textlint-ja/textlint-rule-preset-JTF-style)
- [textlint-rule-date-weekday-mismatch](https://github.com/textlint-rule/textlint-rule-date-weekday-mismatch)
- [textlint-rule-ja-no-inappropriate-words](https://github.com/textlint-ja/textlint-rule-ja-no-inappropriate-words)
- [textlint-rule-ja-no-orthographic-variants](https://github.com/textlint-ja/textlint-rule-ja-no-orthographic-variants)
- [textlint-rule-use-si-units](https://github.com/kn1cht/textlint-rule-use-si-units)
- [textlint-rule-write-good](https://github.com/textlint-rule/textlint-rule-write-good)

### 設定を変えたい場合

Textlintの設定画面を作っており、`textlintrc.json` を流し込めるようにしています。
上記プラグインの設定しか反映できないですが、使っているtextlintrcが動作しますのでご利用ください。

### インストール方法

[README.md](https://github.com/shivase/obsidian-textlint/blob/master/README.md)に記載していますが、コミュニティプラグインとしてまだ登録されていないので、pluginから検索してもまだ出てきません。

必要な資材をVaultのプラグイン配下に置きリロードしてください。

コミュニティプラグインへの登録は、一部修正必須なコードがあるので、それに対応してからになります。

## 最後に

実際に使う中での、ご意見や要望などありましたら気軽にコメントください！
私はヘビーユーザーではないため、利用している人のフィードバックを歓迎しています。
