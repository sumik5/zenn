---
title: "React(+Vite)をGithub PageへDeploy"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ 'react','vite','github','githubactions' ]
published: true
---

React（＋Vite）のSPAを、GitHub Actionsの `actions/deploy-pages@v1` を使ってDeployしたので、そのやり方を記載します。

## GitHub Actionsの設定

昔は `peaceiris/actions-gh-pages` を利用していたようですが、公式の `actions/deploy-pages@v1` で簡単にできるようになったようです（今回使うのが始めて）。

最終的なGitHub Actionの定義は次のとおり。

```yaml
name: deploy

on:
  push:
    branches: ['main']
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: 'pages'
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: 18
          # yarnのキャッシュを効かすために設定 (npmのかたはnpmで)
          cache: yarn
      - name: yarn install
        run: yarn install
      - name: yarn build
        run: yarn build
        # GIthubページは https://USERNAME.github.io/REPOSITORY_NAME/
        # となるので、envを追加してviteのベースパスを変えるように設定します。
        env:
          GITHUB_PAGES: true
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v1
        with:
          path: ./dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v1
```

## Viteの設定

上にコメントで記載しましたが、GitHub Pageは `https://USERNAME.github.io/REPOSITORY_NAME/` となるので、そのままReactを公開すると404になります。

そこでvite.configを弄ってBasePathがGitHubのときは変わるようにします。

```typescript:vite.config.ts
export default defineConfig({
  /* 中略 */
  base: process.env.GITHUB_PAGES
    ? 'REPOSITORY_NAME' // レポジトリ名を設定
    : './'
});
```

## React Routeの設定

Reactの方も、viteのBASE_URLを読み込むように変更。

```typescript
<BrowserRouter basename={import.meta.env.BASE_URL}>
  {children}
</Router>
```

以上で終わりです。気軽にSPAをホスティングでき便利になりましたね。
