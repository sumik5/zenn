---
title: "DeepL+Alfredでクリップボードの英語をブラウザで表示"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Alfred", "DeepL"]
published: true
---

普段 DeepL を愛用していますが、DeepL の標準アプリでは検索結果を残せませんし、色々機能が欲しいなぁと思ったので、DeepL の API と Alfred を使い、クリップボードの英語をブラウザで日本語表示する workflow を作りました
(DeepL の有料版にすればブラウザで使えるので、金払えばいいじゃんって話なんですが...)

![image](/images/003/result.gif)

https://github.com/shivase/alfred-worlflow-deepl-google-translation

### 追記

ver1.1 として、Google と DeepL の結果を同時に表示できるようにしました。(それぞれホットキーで個別に結果表示も可能です)

![deeplandgoogle](/images/003/deeplandgoogle.png)

### 必要なもの

- DeepL アカウント(API キーが必要なので)
- Alfred PowerPack(有料版が必要です)
- jq(homebrew とかでインストールしておいてもらえると)

インストール方法などは上記の Github の README をご参照下さい

### 使い方

クリップボードの値をそのまま使うので、翻訳したい文字をコピー(ctrl+c)して、設定したホットキーを押して下さい。自動的にブラウザで変換結果が表示されます

### 今後

色々やっつけで作ったので、日本語 → 英語の変換もしたいですし、不親切な部分の修正もしていきます

何か不備やこんな機能欲しいとかあれば言ってください！
