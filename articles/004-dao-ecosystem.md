---
title: "daoに関係するエコシステム"
emoji: "🗂"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Web3", "DAO", "Ethereum"]
published: true
---

## DAO 関連のエコシステム

業務でDAOを調べる必要があったので、いろいろ関連するサービスを調査してみました。

現状DAO運営で必要な各種機能を1つのサービスで管理していくことは不可能そう。各種Webサービスを複数使って運用することになるので、よく使われていそうなサービスを検証程度ですが使ってみての各サービスの概要です。

## 先にまとめ

以下の組み合わせが良さそうかな？

- DaoHaus : DAOの管理として
- Dework : プロジェクトの管理、報酬受け渡しとして
- snapshot : 投票のしくみとして
- coodinape : ユーザー間でのフィードバック＋褒賞のしくみとして

## DAO Operating System

DAOを起動し運営していくために必要なベースとなるWebサービス。

### aragon

https://aragon.org/

トークン、DAOの作成・管理が行えるサービス。

mainnetにDAO作成するには `0.1` ETHの手数料がかかる。

DAOは用途ごとに3種類（Company,Membership,Reputation）用意されており、トークン譲渡の可否や投票権のしくみによって選択する形になっている。

Finance機能を使うことで、DAOに各種トークンを資産として預けることが可能。必要に応じて誰かにトークンを受け渡したりできるようになる（それも投票にかけられて承認が必要）。

また各種機能がプラグイン化されており、必要な機能を後から追加していくことが可能であり、必要であればaragonCLIを通して作成ができる。 → [https://hack.aragon.org/](https://hack.aragon.org/)

### DAOHaus

https://daohaus.club/

Guilds/Clubs/Ventures/Grants/Productsから選んでDAOを作成する。いろいろな外部サービスと連携して各種機能を提供している。

資産管理部分はGnosis Safeと連携。

投票はSnapshotと連携可能。基本的な投票機能自体は持っている。

### colony

https://colony.io/product

Gnosis Chainネットワーク上のDAO管理サービス。独自トークンも発行できる。

ステーブルコインであるxDAIがベースであり、各種スマートコントラクトの実行にGAS代を負担しないが、ethereumとは違うので注意。

## DAO プロジェクト管理

DAOを運営するにあたって、プロジェクトのタスク管理などを行うWebサービス。

### Dework

https://dework.xyz/

カンバン形式のプロジェクト管理が可能なサービス。タスクごとにトークンを報酬として払うしくみがある。

notionのデータベースを取り込む機能はあるが、取り込むだけで双方向連携はなし。

Discordとの連携が便利そう。

投票のしくみもあるが、一人一票制で重み付けはなし。

下に記載しているCoodinapeと連携ができる。

## 資産管理

DAOが保有する資産を管理するWebサービス。

### Gnosis safe

https://gnosis-safe.io/

ETH/ERC20のトークン、ERC721のNFTアセット資産管理を提供。

あらかじめ管理者、決済時の必要決済者数を決めておくことで、トークンの転送に制限をかけることができるようになる（5人中3人の承認を得たらOKとするなど）。

DAOで管理しているトークンを特定の一人に委ねてしまうと、秘密鍵の漏洩や紛失時に困るが、Gnosis safeを用いて複数人での管理にすることで安全に資産を管理できるようになる。

### syndicate

資金調達が簡易に行える機能を提供。

[DAO オペレーティングシステムの SYNDICATE とは？特徴と主な取り組みを徹底解説 | DART's MEDIA - NFT に関する世界の最新情報をお届け](https://darts-nft.com/column/post-4261/)

### smart invoice

https://smartinvoice.xyz/

スマートコントラクトによる請求書払いを提供。まえもってクライアントにいくら送金するのか設定すると、その金額をサイト側が保管し、期日になるとクライアントに支払われるしくみ。

## 投票関連

DAOに投票のしくみを提供するWebサービス。

### snapshot

https://snapshot.org/

投票機能の提供に特化したサイト。投票の重み付けや、投票の多肢選択式などに対応。

ERC20やERC721形式のトークンをどこか別で生成する必要がある。

また、事前にENSで `name.eth` 形式の名前を取得しておく必要がある。

### Tally

https://www.tally.xyz/

ERC20トークンを利用したDAOの投票システムを簡易に作成できるサービス。

`Governor Contract` や `Timelock Contract` と呼ばれるスマートコントラクトを準備する必要があるなど、ある程度スマートコントラクト構築の知識が必要。

自前で準備する必要があるが、そのぶん細かい制御が可能な点はメリット。

## Communication

### Coodinape

[https://coordinape.com/](https://coordinape.com/)

DAOの貢献者に対して報酬を与え、ユーザー間で `いいね` のようなアクションを送り合えるサイト（1サイクルにつき100個を好きなように他のユーザーに渡すようなしくみなっている）。

1サイクル = 任意に決めるプロジェクトの周期（1日〜1ヵ月など）

Deworkと連携が可能で、Deworkでユーザーが何のタスクをやったのかなどがわかるようになっているので、褒賞とかはこちらでやるとよいかな。

### Discourse

https://www.discourse.org/

フォーラム・掲示板機能。DAO用ではないが、コミュニケーションツールとして使っているところがある。

### Collab.land

https://collab.land/

Discordで特定のNFTを持っているものに限定したチャンネルを作りたいときに利用する。

### 参考 Link

https://www.cloudot.co.jp/10547/
