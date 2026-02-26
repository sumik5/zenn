---
title: "MacのIntelliJ上でProcessingを動かす"
emoji: "‍🎓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Processing","IntelliJ","Mac"]
published: true
---

## 始めに

私は電気通信大学の夜間（通称K課程）に通っているのですが、2年目のプログラミングの講義でProcessingというグラフィック描画に特化した言語を扱うことになりました。

![Processing画面](/images/015/000.png)

普段はIntelliJを使って開発をしているので、こういった素のメモ帳のようなもので開発するのが非常に苦痛で（Vimキーバインド使えないと生産性激落ち）、何とかIDEで開発しようとIntelliJにProcessingを導入したのでその手順を今回残すことにします。

なお、検索してみると[IntelliJ IDEAでProcessingを書く \#Java \- Qiita](https://qiita.com/shion1118/items/49b803b3217e642cfbd1)という記事が見つかるのですが、この設定ではMacは次のようなエラーメッセージが出てうまく起動しないので、同様に困っている人の参考になれば幸いです。

```java
java.lang.NoClassDefFoundError: com/apple/eawt/QuitHandler
 at java.base/java.lang.Class.getDeclaredMethods0(Native Method)
 at java.base/java.lang.Class.privateGetDeclaredMethods(Class.java:3402)
 at java.base/java.lang.Class.getMethodsRecursive(Class.java:3543)
 at java.base/java.lang.Class.getMethod0(Class.java:3529)
 at java.base/java.lang.Class.getMethod(Class.java:2225)
 at processing.core.PApplet.runSketch(PApplet.java:10707)
 at processing.core.PApplet.main(PApplet.java:10504)
 at processing.core.PApplet.main(PApplet.java:10486)
 at lesson2.polygon.main(polygon.java:8)
Caused by: java.lang.ClassNotFoundException: com.apple.eawt.QuitHandler
 at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:641)
 at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:188)
 at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:525)
```

## 前提

手順はMacを前提として記載しています。Windowsの方はフォルダー構成が違うのでうまく読み取ってください。たぶんフォルダー指定部分をちょっと変えればいけるはずです（もしくは上のQiitaのリンクで解決するかも）。

VS Codeでも同じような設定をすればよいはずですが、せっかく学生は無料でIntelliJが使えるので、一度有料のIDEの便利さを味わうとよいでしょう（布教）。

## 手順

### 新規プロジェクトの作成

まずは新規プロジェクトを作成してください。

プロジェクトはJavaで、JDK 17以降を指定してください。

![プロジェクト作成](/images/015/001.png)

ここはJavaを個別にインストールしていなければ、おそらく16になっています。デフォルトの16のままだと次のようにJavaが古いというエラーがでます（後から変えたい場合は、ファイル→プロジェクト構造から変更可能です）。16以外に選択肢がない場合は、別途OpenJDKなどをインストールしましょう。

```java
エラー: メイン・クラスlesson2.polygonのロード中にLinkageErrorが発生しました
 java.lang.UnsupportedClassVersionError: processing/core/PApplet has been compiled by a more recent version of the Java Runtime (class file version 61.0), this version of the Java Runtime only recognizes class file versions up to 60.0
```

### モジュールを追加する

ではプロジェクトでProcessingが動くよう、Processingのモジュールを追加します。

#### プロジェクト構造画面

まずは**ファイル**→**プロジェクト構造**を選んでください。

![プロジェクト構造を選択](/images/015/002.png)

#### JARを追加する

ここから次の画像のように**JAR またはディレクトリ**を選んでください。

![プロジェクト構造画面](/images/015/003.png)

![JARまたはプロジェクトを選択](/images/015/004.png)

するとFinderが表示されるので、「Application」→「Processing」→「Cotents」→「Java」の順番に指定して表示される「core.jar」を選択してください。

![core.jarを選択](/images/015/005.png)

うまくいけば次のようにcore.jarが表示されているはずです。これで準備完了です。

![core.jarの追加](/images/015/006.png)

### ソースコードを書く

では具体的にProcessingのソースコードを書いていきましょう。
次の画像のとおり、srcフォルダー配下に「test」というJavaプログラムを追加します。

![testコードを追加する](/images/015/007.png)

追加されたら次のコードを貼り付けてください。

```java
import processing.core.PApplet;

// test の部分はファイル名と同じにする
public class test extends PApplet {

  public static void main(String args[]) {
    // ここの名前もファイル名と同じにする
    PApplet.main("test");
  }

  public void settings(){
    // sizeメソッドだけはここに指定する
    size(640,480);
  }
  public void setup(){
  }
  public void draw(){
    // ここに実際のProcessingのコードを記載していく
    background(255);
    quad(400,100,300,200,600,200,550,50);
    triangle(250,250,200,450,600,450);
  }
}
```

### プログラムを実行する

「実行」→「実行」選ぶと選択肢がでるので、そこからtestを選んでください。

![実行](/images/015/009.png)

正常にセットアップできていれば、次のように画像が出力されるはずです。Good！

![結果](/images/015/010.png)

## おわり

これでIDEの補完機能などが使えるようになるので、かなり快適に作業ができますね。
講義では作ったコードを提出することになるので、settings部分に指定した「size（640,480）」の部分と、drawメソッドの部分をコピーし、実際のProcessingの画面で動作確認をしましょう。

```java
    size(640,480);
    background(255);
    quad(400,100,300,200,600,200,550,50);
    triangle(250,250,200,450,600,450);
```

VS CodeやIntelliJのようなIDE（統合開発環境）は実務では必須でしょう。とくにGitHub Copilotを使った開発体験は後戻りできないくらい快適ですので、時間を捻出するのがたいへんな夜間生はこういったツールを使いこなして効率よく作業するのは大事だと思っています。

では良き開発ライフを！
