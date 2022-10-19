---
title: "【Flutter】なぜ import 先に「material.dart」を選ぶのか"
emoji: "🤔"
type: "tech"
topics: ["Flutter",]
published: false
---

## はじめに

はじめまして、[ダイゴ](https://twitter.com/mamushi_journey)です。
Flutter で開発をしていると頻繁に、 ↓ のような import 選択をすることがあるかと思います。

![](https://storage.googleapis.com/zenn-user-upload/929f3197209c-20221018.png)

今までは **「とりあえず material 選択しといたら良い」** と聞いたことがあったので、脳死で material を選択していたのですが、先日 **「なんで material を選択するんだろう」「widgets じゃだめなのか」** と疑問に思い、それをきっかけに Dart の ディレクティブ について理解が深まったので、記事にしました。

どなたかの参考になれば幸いです。

## なぜ material を選ぶのか

結論、

**「（Material デザインに則った）Flutter アプリ開発に必要なファイルをすべて包含しているファイルだから」** 

です。

図で示すとこんな感じです。

=================== 図 ===================

具体的に解説していきます。（文中、間違っている点等あればご指摘いただけると幸いです🙏）

## material.dart の中身

今まで脳死で選択していた material.dart ファイルの中身を見てみると、以下のようになっています。

![](https://storage.googleapis.com/zenn-user-upload/2de7ee891a0e-20221018.png)

![](https://storage.googleapis.com/zenn-user-upload/f27a54523c09-20221018.png)



ファイルに書かれている内容をざっくりまとめると
- material という **library** として公開している
- いろんなファイルを **export** している

という2点になります。

import はよく使うので「他のファイルを使えるようにするもの」となんとなく分かりますが、**library** と **export** が分かりません。
それぞれについて調べてみましょう。

## library と export

以下の記事がとても参考になりました。

https://zenn.dev/littleforest/articles/a4fc5bc7c944d42f66d2

本記事向けにまとめるなら

#### library
- ライブラリ（`import`を用いて読み込むことのできるファイル）に名前をつけるために使う
- `library` と明示しなくても、Dart で書かれたファイルは自動的に 1 つのライブラリとして扱われる

#### export
- 外部に公開するライブラリ（ファイル）を指定するために使う
- メインのライブラリ（ファイル）に `export` を記述することで、メインのライブラリを読み込むだけで `export` で指定されたライブラリ（ファイル）にもアクセスできる

といった感じでしょうか。
つまり、上述の material.dart ファイル内では
- `library` で、material.dart ファイルを「material」というライブラリとして外部に公開する
- `export` で、Material デザインに則ったコンポーネント群（AppBar、ElevatedButton等）を「material」を読み込むだけでアクセスできるようにしている

といったことが行われています。

![](https://storage.googleapis.com/zenn-user-upload/3f44e8146964-20221019.png)


## widgets.dart じゃだめなのか

material.dart で行われていることはだいたい把握できましたが、当初疑問に思っていた **「widgets.dartじゃだめなのか」** という疑問はまだ解決してません。

![](https://storage.googleapis.com/zenn-user-upload/89af65bdd3ca-20221019.png)

widgets.dart と聞くと全部の widget を import できそうですが、試しに widgets.dart を import してみると以下のようになります。

![](https://storage.googleapis.com/zenn-user-upload/3941e7c93b3a-20221019.png)

Material 関連の widget（`MaterialApp`, `ThemeData` 等）が参照できていません。
widgets.dart 内では Material 系のコンポーネントが export されていないためです。
内部を見てみると、widgets.dart 内では Material でも Cupertino でもない共通の widget（`Text` や `GestureDetector` 等）が export されています。

![](https://storage.googleapis.com/zenn-user-upload/eb9a9379e624-20221019.png)


そして実は、material.dart の末尾では widgets.dart が export されているため、material を import すれば widgets.dart 内のクラスも参照できているというわけですね。

![](https://storage.googleapis.com/zenn-user-upload/d7dede7ad347-20221019.png)

そのため、


## まとめ

## 参考記事

https://dart.dev/guides/language/language-tour#libraries-and-visibility

https://dart.dev/guides/libraries/create-library-packages


