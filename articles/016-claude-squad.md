---
title: "tmux上でClaude Codeを多重起動し、それぞれに役割を分担させ協調させる環境を自動構築するアプリを作りました"
emoji: "🤖"
type: "tech"
topics: ["claude", "claudecode", "tmux"]
published: false
---

## 始めに

私はtmux環境上でClaude Codeを多重起動し、それぞれのAgentに役割をもたせて連携させていましたが、段々と毎回起動するのが面倒だったので**Claude Squad**というツールを今回作成いたしました。

[Claude Squad](https://github.com/catenas-g/claude-squad)

これは、tmux上で複数のClaude Code AIエージェントを並列実行し、チーム開発のようにプロジェクトを進めてくれるツールになります。

といっても作って使い始めてもうかなり経ち、公開したのも先週ですので、もしかしたら今のClaude CodeにAgent機能が追加されたので、それをうまく使えばもっと良い方法できるかもしれません。

:::message
Claude Codeを私はこれを使って12多重くらいで動かしていますが（2つのClaude Squadを起動）、Claude Codeがダマで制限をかけて話題になっています。
MAXプランでないとトークン死しますし、そのMAXでも制限にひっかかりやすくなるので、そこだけご注意ください。
:::

## 動作イメージ

![動作イメージ画像](/images/016/screen_shot.gif)

## Claude Squadとは

Claude Squadは、Product Owner、Manager、複数のDeveloperという役割分担をもつAIエージェントチームをtmux上で動かす統合開発環境システムです。

### システム構成

Claude Squadは2つの主要コンポーネントで構成されています：

1. **claude-squad**: AIエージェントセッションを起動・管理するメインシステム
2. **send-agent**: 起動中のエージェントにメッセージを送信するクライアントツール

### 起動されるエージェントの役割

- **po （Product Owner）**: プロジェクト全体の統括と戦略決定（左上pane）
- **manager**: プロジェクトマネージャーとしてチーム管理と作業分担（左下pane）
- **dev1〜dev4**: 実際の開発作業を行う実行エージェント（右側pane）

## 動作原理

### エージェント間連携のしくみ

Claude Squadの核心は、各エージェントが明確な役割分担を持ち、`send-agent` コマンドを通じて相互にメッセージを送信しあうしくみになっています。

#### 基本的な作業フロー

1. Product Owner: ユーザーからの要求を受け取り、プロジェクト方針を決定
2. Manager: POからの指示をもとに、各開発エージェントに適切な作業を分担
3. Developer: 割り当てられた専門領域の作業を実行
4. 統合： Managerが各成果物を統合し、POに最終承認を求める

#### エージェント設計の特徴

各エージェントは厳格な役割制限が設けられています。

#### Product Ownerの制約

- 直接的なコーディングや作業実行を禁止
- ファイル変更系ツール（Write、Edit、bash等）の使用を制限
- 情報収集と戦略的判断に特化

#### Manager の責務

- 作業分担と進捗管理
- 品質確認と統合作業
- POと開発チーム間の橋渡し

#### Developer の柔軟性

- プロジェクトに応じた専門役割（フロントエンド、バックエンド、データアナリスト等）
- 実際のコード作成とテスト実行
- 作業完了時のManagerへの報告

## 利用方法

基本的には[README.md](https://github.com/catenas-g/claude-squad/blob/main/README.md)を見ていただければと思います。
各役割をチューニングしたいとかあると思いますので、その場合はREADMEをご確認ください。

### 前提条件

- tmux
- Claude Code CLI（認証済み）

### 環境構築

中身はGo言語で作っているので、Windowsでも動くはずですが、動作確認まではしていません。おそらくMakefileを弄る必要があると思います。

```bash
# リポジトリのクローン
git clone https://github.com/catenas-g/claude-squad.git
cd claude-squad

# システムへのインストール
make install

# 設定の初期化（日本語版）
claude-squad --init ja

# システム診断
claude-squad --doctor

# 設定確認
claude-squad --show-config
```

### セキュリティ設定

Claude Squadは `dangerously-skip-permissions` をONにして動作するため、エージェントに適切な制約を設ける必要があります。

`~/.claude/claude-squad/settings.json` で `allow` と `deny` の設定を行ってください。

**これをしないと大暴れしますのでご注意ください。**

[settings.jsonの例](https://github.com/catenas-g/claude-squad/blob/main/docs/settings.json)

### 実行

```bash
# プロジェクトディレクトリに移動
cd [プロジェクトフォルダ]

# セッション名を指定して起動
claude-squad myproject
```

## Link

このしくみは、次のページを参考に、自分で楽に使いやすいように改善したものになります。

[Claude Codeで複数のAIエージェントが協調作業！並列実行チーム構築ガイド](https://chatgpt-lab.com/n/na59171855b1e)
