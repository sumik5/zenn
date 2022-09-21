---
title: "daoに関係するエコシステム"
emoji: "🗂"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Web3", "DAO", "Ethereum"]
published: true
---

## DAO 関連のエコシステム

DAO 構築に関わることになりそうだったので、色々調べてみました
現状 DAO 運営で必要な各種機能を一つのサービスで管理していくことは不可能そう。各種 Web サービスを複数使って運用することになるので、よく使われていそうなサービスを検証程度ですが使ってみての各サービスの概要です。

## 先にまとめ

以下の組み合わせが良さそうかな？

- DaoHaus … DAO の管理として
- Dework … プロジェクトの管理、報酬受け渡しとして
- snapshot … 投票の仕組みとして
- coodinape … ユーザ間でのフィードバック＋褒賞の仕組みとして

## DAO Operating System

DAO を起動し運営していくために必要なベースとなる Web サービス

### aragon

https://aragon.org/

トークン、DAO の作成・管理が行えるサービス。

mainnet に DAO 作成するには `0.1` ETH の手数料がかかる。

DAO は用途ごとに 3 種類(Company,Membership,Reputation)用意されていて、トークンの譲渡しの可否や投票権の仕組みによって選択する形になっている。

Finance 機能を使うことで、DAO に各種トークンを資産として預けることが可能。必要に応じて誰かにトークンを受け渡したりできるようになる（それも投票にかけられて承認が必要）。

また各種機能がプラグイン化されており、必要な機能を後から追加していくことが可能であり、必要であれば aragonCLI を通して作成ができる。 → [https://hack.aragon.org/](https://hack.aragon.org/)

### DAOHaus

https://daohaus.club/

Guilds/Clubs/Ventures/Grants/Products から選んで DAO を作成する。色々な外部サービスと連携して各種機能を提供している。

資産管理部分は Gnosis Safe と連携。

投票は Snapshot と連携可能。基本的な投票機能自体は持っている。

### colony

https://colony.io/product

Gnosis Chain ネットワーク上の DAO 管理サービス。独自トークンも発行できる。

ステーブルコインである xDAI がベースであり、各種スマートコントラクトの実行に GAS 代を負担しないが、ethereum とは違うので注意

## DAO プロジェクト管理

DAO を運営するにあたって、プロジェクトのタスク管理などを行う Web サービス

### Dework

https://dework.xyz/

カンバン形式のプロジェクト管理が可能なサービス。タスク毎にトークンを報酬として払う仕組みがある。

notion のデータベースを取り込む機能はあるが、取り込むだけで双方向連携は無し。

Discord との連携が便利そう

投票の仕組みもあるが、一人一票制で重み付けはなし。

下に記載している Coodinape と連携ができる。

## 資産管理

DAO が保有する資産を管理する Web サービス

### Gnosis safe

https://gnosis-safe.io/

ETH/ERC20 のトークン、ERC721 の NFT アセットの資産管理を提供

予め管理者、決済時の必要決済者数を決めておくことで、トークンの転送に制限をかけることができるようになる（5 人のうち 3 人の承認を得たら OK とするなど）。

DAO で管理しているトークンを特定の一人に委ねてしまうと、秘密鍵の漏洩や紛失時に困ったことになるが、Gnosis safe を用いて複数人での管理にすることで安全に資産を管理できるようになる。

### syndicate

資金調達が簡易に行える機能を提供

[DAO オペレーティングシステムの SYNDICATE とは？特徴と主な取り組みを徹底解説 | DART's MEDIA - NFT に関する世界の最新情報をお届け](https://darts-nft.com/column/post-4261/)

### smart invoice

https://smartinvoice.xyz/

スマートコントラクトによる請求書払いを提供。
事前にクライアントにいくら送金するのか設定すると、その金額をサイト側が保管し、期日になるとクライアントに支払われる仕組み。

## 投票関連

DAO に投票の仕組みを提供する Web サービス

### snapshot

https://snapshot.org/

投票機能の提供に特化したサイト。投票の重み付けや、投票の多肢選択式などに対応

ERC20 や ERC721 形式のトークンをどこか別で生成する必要がある。

また、事前に ENS で `name.eth` 形式の名前を取得しておく必要がある。

### Tally

https://www.tally.xyz/

ERC20 トークンを利用した DAO の投票システムを簡易に作成できるサービス

`Governor Contract`や `Timelock Contract` と呼ばれるスマートコントラクトを準備する必要があるなど、ある程度スマートコントラクト構築の知識が必要。

自前で準備する必要があるが、その分細かい制御が可能な点はメリット

## Communication

### Coodinape

[https://coordinape.com/](https://coordinape.com/)

DAO の貢献者に対して報酬を与えたり、ユーザ間で `いいね` のようなアクションを送り合えるサイト（1 サイクルにつき 100 個を好きなように他のユーザに渡すような仕組みなっている）。

1 サイクル = 任意に決めるプロジェクトの周期（1 日〜1 ヶ月など）

Dework と連携が可能で、Dework でユーザが何のタスクをやったのかなどがわかるようになっているので、褒賞とかはこちらでやるといいかも

### Discourse

https://www.discourse.org/

フォーラム・掲示板機能。DAO 用ではないが、コミュニケーションツールとして使っているところがある。

### Collab.land

https://collab.land/

Discord で特定の NFT を持っているものに限定したチャンネルを作りたい時に利用する。

### 参考 Link

https://www.cloudot.co.jp/10547/
