---
title: "textlintプラグインの作り方(例：オンドゥル語変換) プラグイン作成編"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "textlint","github","typescript","git" ]
published: true
---

本記事は[textlintプラグインの作り方(例：オンドゥル語変換) 準備編](https://zenn.dev/shivase/articles/006-how-to-create-new-textlint-plugin-1)の続きにあたります。

* [準備編](https://zenn.dev/shivase/articles/006-how-to-create-new-textlint-plugin-1)
* プラグイン作成編 ← 本記事
* [リリース編](https://zenn.dev/shivase/articles/008-how-to-create-new-textlint-plugin-3)

## 本記事概要

実際にプラグインのコードを書き、コマンドラインでのtextlintを実行して、正常にオンドゥル変換できるかを確認します。

## プラグインの作成

### カタカナ→オンドゥル語変換する関数の作成

オンドゥル語への変換は、今回はカナの単純な一文字ずつ変換とし、 `lib/ondulish.ts` を作成して、文字変換自体の処理はそちらに任せます。

```bash
mkdir lib
touch lib/ondulish.ts
```

実際のロジックはこちら。サンプルなので単純に。
https://github.com/shivase/textlint-rule-ondul-style/blob/v1.0.0/src/lib/ondulish.ts

### 形態素解析の導入

変換対象の文字列を単語に分解するためのモジュール、形態素解析のkuromojiを使います。日本語の読みをカナで出力してくれるため、それを上の `ondulish.ts` に渡します。

textlintではそのwrapper関数である `kuromojin` が用意されているので、そちらをインストール。

```bash
yarn add kuromojin
```

### メインロジック作成

`index.ts` を今回は次のようにかきました。

```typescript:index.ts
import { TextlintRuleModule } from '@textlint/types';
import { tokenize } from 'kuromojin';
import ondulish from './lib/ondulish';

export interface Options {
  // if the Str includes `allows` word, does not report it
  allows?: string[];
}

const reporter: TextlintRuleModule<Options> = (context) => {
  const { Syntax, RuleError, report, getSource } = context;
  // const allows = options.allows ?? [];

  return {
    async [Syntax.Str](node) {
      const text = getSource(node); // 文字列を取得

      const tokens = await tokenize(text); // kuromojiで形態素解析し、単語に分ける
      tokens.forEach((token) => {
        // 単語が日本語の場合pronunciationにカタカナ読みが入るので、その時のみ処理
        // token.readingにもカナが入るが、発音を正しく？オンドゥルするためにpronunciation選択
        if (token.pronunciation) {
          const ondul = ondulish(token.pronunciation); // オンドゥル語に変換
          if (token.reading !== ondul) {
            const message = `${token.surface_form} => ${ondul}`; // 出力メッセージへ整形
            // メッセージを出力するためのRuleErrorクラスの作成
            const ruleError = new RuleError(message, {
              index: token.word_position - 1,
            });
            // メッセージ出力
            report(node, ruleError);
          }
        }
      });
    },
  };
};
export default reporter;
```

:::message
`Syntax.Str` の部分など、本当は変えたほうがよい部分はあるのですが、今回はなるべくシンプルにすべてをオンドゥル変換することとします。
:::

### ダミーファイル作成

変換対象のファイルを作成しておきます。

```bash
touch ondulish_target.txt
```

```txt:ondulish_target.txt
本当に裏切ったんですか
```

### LET'S ONDULISH！

まずはコンパイルします。

```bash
$ yarn build
yarn run v1.22.19
rimraf lib
textlint-scripts build
Successfully compiled 2 files with Babel (829ms).
✨  Done in 7.91s.
```

そしてONDULISH！

```bash
# うまくいかない時は --debug をつけると何かわかるかもしれません
$ yarn textlint --rulesdir ./lib ondulish_target.txt
(中略)

textlint-rule-ondul-style/ondulish_target.txt
  1:1   error  本当に => ｵﾝﾄﾞｩーﾙ  index
  1:4   error  裏切っ => ﾙﾗｷﾞッ    index
  1:7   error  た => ﾀﾞ            index
  1:8   error  ん => ﾝ             index
  1:9   error  です => ﾃﾞｨｽ        index
  1:11  error  か => ｶ             index
```

よいかんじですね！

kuromojinでtokenizeすると、次のような情報を持ったtokenが渡されます。適時自分の作りたいプラグインに合わせて使うとよいでしょう。

https://github.com/azu/kuromojin

```typescript
tokenize(text).then(tokens => {
    console.log(tokens)
    /*
    [ {
        word_id: 509800,          // 辞書内での単語ID
        word_type: 'KNOWN',       // 単語タイプ(辞書に登録されている単語ならKNOWN, 未知語ならUNKNOWN)
        word_position: 1,         // 単語の開始位置
        surface_form: '黒文字',    // 表層形
        pos: '名詞',               // 品詞
        pos_detail_1: '一般',      // 品詞細分類1
        pos_detail_2: '*',        // 品詞細分類2
        pos_detail_3: '*',        // 品詞細分類3
        conjugated_type: '*',     // 活用型
        conjugated_form: '*',     // 活用形
        basic_form: '黒文字',      // 基本形
        reading: 'クロモジ',       // 読み
        pronunciation: 'クロモジ'  // 発音
      } ]
    */
});
```

忘れずcommitしておきましょう。

```bash
$ git add .
$ git commit -m "feat: オンドゥル変換まで実装"
yarn run v1.22.19
$ /textlint-rule-ondul-style/node_modules/.bin/lint-staged
✔ Preparing lint-staged...
✔ Running tasks for staged files...
✔ Applying modifications from tasks...
✔ Cleaning up temporary files...
✨  Done in 8.22s.
```

### Fix対応のための改良

これでリリースの準備はできていますが、 `fix` に対応させ、自動的に修正するまでやりたいですよね。ここからはそのための処理です。

index.tsを変更し、fixerに関する処理を追加します。 `RuleError` に `fix` を追加し、修正対象の単語の開始位置・終了位置と、変換文字列が入るようにしています。

```typescript
const ruleError = new RuleError(message, {
   index: tokenFromIndex,
   fix: fixer.replaceTextRange(
     [tokenFromIndex, tokenToIndex],
     ondul,
   ),
),
```

export部分も修正し、 `fixer` を追加します。

```typescript
export default {
  linter: reporter,
  fixer: reporter,
};
```

最終的な差分は以下です。

```diff typescript:index.ts
 const reporter: TextlintRuleModule<Options> = (context) => {
-  const { Syntax, RuleError, report, getSource } = context;
+  const { Syntax, RuleError, report, fixer, getSource } = context;
   // const allows = options.allows ?? [];
 
   return {
@@ -23,9 +23,15 @@ const reporter: TextlintRuleModule<Options> = (context) => {
           const ondul = ondulish(token.pronunciation); // オンドゥル語に変換
           if (token.reading !== ondul) {
             const message = `${token.surface_form} => ${ondul}`; // 出力メッセージへ整形
+            const tokenFromIndex = token.word_position - 1;
+            const tokenToIndex = tokenFromIndex + token.surface_form.length;
             // メッセージを出力するためのRuleErrorクラスの作成
                          const ruleError = new RuleError(message, {
-              index: token.word_position - 1,
+              index: tokenFromIndex,
+              fix: fixer.replaceTextRange(
+                [tokenFromIndex, tokenToIndex],
+                ondul,
+              ),
             });
             // メッセージ出力
             report(node, ruleError);
           }
        }
      });
    },
   };
 };
-export default reporter;
+
+export default {
+  linter: reporter,
+  fixer: reporter,
+};
```

おそらく `tokenToIndex` はこのロジックだと悪い気はしますが、今回はよいかな。

### Let's Ondulish

さてFixしてみましょう。まずは `fix` なしで出力してみます。

```bash
$ yarn build
$ yarn textlint --rulesdir ./lib ondulish_target.txt
yarn run v1.22.19
/textlint-rule-ondul-style/node_modules/.bin/textlint --rulesdir ./lib ondulish_target.txt
ホントーニ
ウラギッ
タ
ン
デス
カ

/textlint-rule-ondul-style/ondulish_target.txt
  1:1   ✓ error  本当に => ｵﾝﾄﾞｩーﾙ  index
  1:4   ✓ error  裏切っ => ﾙﾗｷﾞッ    index
  1:7   ✓ error  た => ﾀﾞ            index
  1:8   ✓ error  ん => ﾝ             index
  1:9   ✓ error  です => ﾃﾞｨｽ        index
  1:11  ✓ error  か => ｶ             index

✖ 6 problems (6 errors, 0 warnings)
✓ 6 fixable problems.
Try to run: $ textlint --fix [file]

error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

`✓ 6 fixable problems.` と出ましたね。修正が可能になりました。 `-fix` を追加して再度実行します。

```bash
$ yarn textlint --rulesdir ./lib ondulish_target.txt --fix
yarn run v1.22.19
/textlint-rule-ondul-style/node_modules/.bin/textlint --rulesdir ./lib ondulish_target.txt --fix
ホントーニ
ウラギッ
タ
ン
デス
カ

/textlint-rule-ondul-style/ondulish_target.txt
  1:1   ✔   本当に => ｵﾝﾄﾞｩーﾙ  index
  1:4   ✔   裏切っ => ﾙﾗｷﾞッ    index
  1:7   ✔   た => ﾀﾞ            index
  1:8   ✔   ん => ﾝ             index
  1:9   ✔   です => ﾃﾞｨｽ        index
  1:11  ✔   か => ｶ             index

✔ Fixed 6 problems

✨  Done in 2.73s.

$ cat ./ondulish_target.txt
ｵﾝﾄﾞｩーﾙﾙﾗｷﾞッﾀﾞﾝﾃﾞｨｽｶ
```

無事変換できました！あとはcommitしてテストに進みましょう。

```bash
$ git add .
$ git commit -m "feat: fixerの追加"
yarn run v1.22.19
$ /textlint-rule-ondul-style/node_modules/.bin/lint-staged
✔ Preparing lint-staged...
✔ Running tasks for staged files...
✔ Applying modifications from tasks...
✔ Cleaning up temporary files...
✨  Done in 8.41s.
[v1.0.0 69e2329] feat: fixerの追加
 2 files changed, 13 insertions(+), 4 deletions(-)
 delete mode 100644 ondulish_target.txt
```

## テスト

では実際にテストコードを修正します。
と言っても、今回はすごく単純なのでひとつテストケースを作るだけにしました。

```typescript:index-text.ts
import TextLintTester from 'textlint-tester';
import rule from '../src';

const tester = new TextLintTester();
// ruleName, rule, { valid, invalid }
tester.run('rule', rule, {
  invalid: [
    {
      text: `本当に裏切った`,
      errors: [
        {
          message: '本当に => ｵﾝﾄﾞｩﾙﾙ',
          line: 1,
          column: 1, //文字の開始位置は1
        },
        {
          message: '裏切っ => ﾙﾗｷﾞｯ',
          line: 1,
          column: 4, //「本当に」の次の文字なので4
        },
        {
          message: 'た => ﾀﾞ',
          line: 1,
          column: 7, //たは7文字目になる
        },
      ],
    },
  ],
});
```

Fix機能があるので、 `column` がただしく設定されているかをちゃんとチェックしています。

### テスト実行

`yarn test` を実行しましょう。

```bash
$ yarn test
yarn run v1.22.19
$ textlint-scripts test


  rule
ホントウニ
ウラギッ
タ
    ✔ 本当に裏切った (1150ms)


  1 passing (1s)

✨  Done in 7.17s.
```

ではコミットします。

```bash
git add .
git commit -m "feat: test 追加"
```

### リリースに備えて

次で最後のリリースになるのですが、このままだと今のコミットログは次のような感じで、そのままマージするにはちょっと微妙です（commitやりなしたりしたので実際のログと手順にずれがあります）。そこでコミットを1つにまとめる処理をしておきます。

```bash
$ git log --oneline
69e2329 (HEAD -> v1.0.0, origin/v1.0.0) feat: fixerの追加
82eecfd fix: dependenciesが間違ってた
ed8109f feat: オンドゥル変換まで実装
2271df6 add: カタカナ変換
084a4a6 fix: huskyが実行されなかったので修正
e66bfce feat: linterを準備
97c1a2e fix: lintエラーを回避
e0080b8 (main) feat: initial commit
```

```bash
# 97c1a2e から 69e2329 を一つにするので、HEADから7つをまとめます
$ git rebase -i HEAD~7

# editorでpickの部分をsquashに変更(sでもOK)
pick 97c1a2e fix: lintエラーを回避
squash e66bfce feat: linterを準備
squash 084a4a6 fix: huskyが実行されなかったので修正
squash 2271df6 add: カタカナ変換
squash ed8109f feat: オンドゥル変換まで実装
squash 82eecfd fix: dependenciesが間違ってた
squash 69e2329 feat: fixerの追加

# 変更を確定すると、新たに画面が出るのでコミットメッセージを追加

# This is a combination of 7 commits.
# This is the 1st commit message:

feat: 初期リリース

# 確定後のログ
[detached HEAD fb52f9c] feat: 初期リリース
 Date: Fri Oct 14 19:11:31 2022 +0900
 11 files changed, 1874 insertions(+), 93 deletions(-)
 create mode 100644 .eslintignore
 create mode 100644 .eslintrc.yml
 create mode 100755 .husky/pre-commit
 create mode 100644 .prettierignore
 create mode 100644 .prettierrc
 create mode 100644 src/lib/ondulish.ts
Successfully rebased and updated refs/heads/v1.0.0.

~/repo/shivase/textlint-rule-ondul-style v1.0.0  4m 22
```

1つになったか確認してみましょう。

```bash
$ git log
commit fb52f9c50fd3a1684b2a0ebf5c8b371858685e7c (HEAD -> v1.0.0)
Author: shivase <keitti05@gmail.com>
Date:   Fri Oct 14 19:11:31 2022 +0900

    feat: 初期リリース

commit e0080b86d540770ef6373103a3cbc426dc5ace66 (main)
Author: shivase <keitti05@gmail.com>
Date:   Fri Oct 14 18:30:00 2022 +0900

    feat: initial commit

$ git push origin v.1.0.0
Enumerating objects: 24, done.
Counting objects: 100% (24/24), done.
Delta compression using up to 2 threads
Compressing objects: 100% (12/12), done.
Writing objects: 100% (16/16), 31.61 KiB | 4.52 MiB/s, done.
Total 16 (delta 3), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (3/3), completed with 3 local objects.
To https://github.com/shivase/textlint-rule-ondul-style.git
 + 69e2329...fb52f9c v1.0.0 -> v1.0.0 (forced update)
```

これでリリース準備ができました！

次はテストを書いて、リリースまでします。

[リリース編へ](https://zenn.dev/shivase/articles/008-how-to-create-new-textlint-plugin-3)

## プラグイン参考にしたリポジトリ

Fixerの設定
[textlint-ja/textlint-rule-ja-no-abusage: よくある日本語の誤用をチェックするtextlintルール](https://github.com/textlint-ja/textlint-rule-ja-no-abusage)

kuromojinの使い方
[textlint-ja/textlint-rule-no-dropping-i: い抜き言葉をチェックするtextlintルール](https://github.com/textlint-ja/textlint-rule-no-dropping-i)
