---
title: "【Flutter】なぜウィジェットの import 先に「material.dart」を選ぶのか"
emoji: "🤔"
type: "tech"
topics: ["Flutter", "Dart"]
published: true
---

## はじめに

はじめまして、[ダイゴ](https://twitter.com/mamushi_journey)です。
Flutter で開発をしていると頻繁に、 ↓ のような `import` 選択をすることがあるかと思います。

![](https://storage.googleapis.com/zenn-user-upload/929f3197209c-20221018.png)

今までは **「とりあえず material.dart を選択しといたら良い」** と聞いたことがあったので、脳死で material を選択していたのですが、先日 **「なんで material.dart を選択するんだろう」「widgets.dart じゃだめなのか」** と疑問に思い、それをきっかけに Dart の ディレクティブ について少し理解が深まったので、記事にしました。

どなたかの参考になれば幸いです。

## なぜ material.dart を選ぶのか

結論、

**「（Material デザインに則った）Flutter アプリ開発に必要なファイルをすべて包含しているファイルだから」** 

です。

図で示すと、

- material.dart
- cupertino.dart
- widgets.dart

の 3つのファイルは以下のようなな関係になっていて、

![](https://storage.googleapis.com/zenn-user-upload/e0fe4498e9df-20221020.png)

material.dart を `import` すると青色部分のライブラリ（ファイル）が使えるようになります

![](https://storage.googleapis.com/zenn-user-upload/a8d1cca994a9-20221020.png)


ファイルを見ながら具体的に解説していきます。（文中、間違っている点等あればご指摘いただけると幸いです🙏）

## material.dart の中身

今まで脳死で選択していた material.dart の中身を見てみると、以下のようになっています。

![](https://storage.googleapis.com/zenn-user-upload/2de7ee891a0e-20221018.png)

![](https://storage.googleapis.com/zenn-user-upload/f27a54523c09-20221018.png)



ファイルに書かれている内容をざっくりまとめると
- material という **`library`** として公開している
- いろんなファイルを **``export``** している

という2点になります。

**`import`** はよく使うので「他のファイルを使えるようにするもの」となんとなく分かりますが、**`library`** と **`export`** が分かりません。

## `library` と `export`

以下の記事がとても参考になりました。

https://zenn.dev/littleforest/articles/a4fc5bc7c944d42f66d2

本記事向けにまとめるなら

#### `library`
- ライブラリ（`import`を用いて読み込むことのできるファイル）に名前をつけるために使う
- `library` と明示しなくても、Dart ファイルであれば自動的に 1 つのライブラリとして扱われる

#### `export`
- 外部に公開するライブラリ（ファイル）を指定するために使う
- メインのライブラリ（ファイル）に `export` を記述することで、そのライブラリを読み込むだけで `export` で指定されたライブラリ（ファイル）にもアクセスできる

といった感じでしょうか。
つまり、上述の material.dart ファイル内では
- `library`：material.dart ファイルを「material」というライブラリとして外部に公開する
- `export` ：Material デザインに則ったコンポーネントファイルたち（AppBar、ElevatedButton等）を「material」を読み込むだけでアクセスできるようにしている

といったことが行われています。

![](https://storage.googleapis.com/zenn-user-upload/3f44e8146964-20221019.png)


## widgets.dart じゃだめなのか

widgets.dart と聞くと全部の widget を `import` できそうですが、試しに widgets.dart を `import` してみると以下のようになります。

![](https://storage.googleapis.com/zenn-user-upload/3941e7c93b3a-20221019.png)

Material 関連の widget（MaterialApp, ThemeData 等）が参照できていません。
これは、widgets.dart 内では Material 系のコンポーネントが `export` されていないためです。図でいうと以下のような状態です。

![](https://storage.googleapis.com/zenn-user-upload/c58c7624a719-20221020.png)

widgets.dart では Material でも Cupertino でもない共通の widget（Text や GestureDetector 等）が `export` されています。

![](https://storage.googleapis.com/zenn-user-upload/eb9a9379e624-20221019.png)


そして material.dart と cupertino.dart をよく見てみると、両方とも widgets.dart を `export` しています。

![](https://storage.googleapis.com/zenn-user-upload/ac4657923adf-20221020.png)

![](https://storage.googleapis.com/zenn-user-upload/167134ff1844-20221020.png)

つまり、どちらか一方でも `import` すれば widgets.dart 内のクラスも参照できているというわけですね。（まさに冒頭の図のイメージ）


![](https://storage.googleapis.com/zenn-user-upload/a8d1cca994a9-20221020.png)


## まとめ

`export`, `import` は頭の中で「どっちが何だ？」みたいな感じになりがちでしたが、図にするとすんなり理解できました。

どちらも Flutter の内部ファイルを見に行った時とかによく見かけると思うので、よくわからなくなったら時はこの図を参考にしていただけると幸いです。

最後までご覧いただき、ありがとうございました。

## 参考記事

https://dart.dev/guides/language/language-tour#libraries-and-visibility

https://dart.dev/guides/libraries/create-library-packages


