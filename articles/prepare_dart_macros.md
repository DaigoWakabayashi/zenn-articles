---
title: "【Dart】macros 登場に備える"
emoji: "🎯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dart", "Flutter"]
published: true
---

この記事は、[Flutter 大学アドベントカレンダー 2024](https://qiita.com/advent-calendar/2024/flutteruniv) 1 日目の記事です。

## はじめに

こんにちは、[ダイゴ](https://twitter.com/mamushi_journey) です。

これまで Dart は、[Null safety](https://dart.dev/null-safety), [Enhanced enum](https://dart.dev/language/enums#declaring-enhanced-enums), [Record](https://dart.dev/language/records), [Patterns](https://dart.dev/language/patterns), [Class-modifiers](https://dart.dev/language/class-modifiers) など、様々な言語仕様の進化を遂げてきました。

![](https://storage.googleapis.com/zenn-user-upload/37fe22b06c9b-20241130.png)
*[Dart3.0の概要と実務における活用](https://docs.google.com/presentation/d/1lt6Kfnrj5JABO_lFydQTkYBqPpitnB84jATo_GN5cO0/edit#slide=id.g24e63824e57_2_41)*


本記事では、Dart の次なる大きな進化となるであろう [macros](https://dart.dev/language/macros) について、その概要と実装方法をまとめています。

タイムライン的には、後述する JsonCodable macro の stable リリースが 2024年末、そして Dart の完全な言語機能としての stable リリースは 2025 年初頭を目指していると記載されており、そう遠くない未来でリリースされうる機能なので、今のうちにイメージを掴んでおきたいですね。

> We are working towards a stable release of the JsonCodable macro later this year (2024), and a stable release of the full language feature (namely, writing your own macros) early next year (2025).

https://dart.dev/language/macros#timeline

どなたかの参考になれば幸いです。

## 0. どんなことが出来るようになるの？

具体的な解説に入る前に、まずは macros によってどのようなことが出来るようになるのかをざっくりイメージとして掴もうと思います。

先日行われた [FlutterKaigi 2024](https://2024.flutterkaigi.jp/) での [ちゅーやんさん](https://x.com/chooyan_i18n?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor) のセッション冒頭デモが非常に分かりやすく、実務での活用イメージが掴みやすかったので、そのまま参照させていただきます。（ちゅーやんさん、いつもありがとうございます）

https://youtu.be/CE0NTGJLbS8?t=5609

動画を見ると、コードの変更と同時に、その内容に応じた別のコードがリアルタイムに生成されていることが分かります。
 
今日の Flutter 開発現場では、[build_runner](https://pub.dev/packages/build_runner) パッケージの `dart run build_runner <build or watch>` コマンドを起点にコード生成することが一般的かと思いますが、上記動画のようなスピード感で生成出来るとなると、開発生産性の観点でも非常に強力ですね。

macros がインパクトの大きな言語機能であることが分かったので、具体的に macros とは何なのか、どうやって使うのかなどについて解説していきます。

## 1. macros とはなにか

公式ドキュメントの macros トップページでは

> **The Dart macro system** is a major new language feature currently under development which adds support for **static meta-programming** to the Dart language.
> （訳）Dartマクロシステムは、Dart言語に静的メタプログラミングのサポートを追加する、現在開発中の主要な新しい言語機能である。

> A Dart macro is a user-definable piece of code that takes in other code as parameters and operates on it in real-time to create, modify, or add declarations.
> （訳）Dartマクロは、ユーザーが定義可能なコードの一部であり、他のコードをパラメータとして受け取り、リアルタイムでそれを操作して宣言を作成、修正、追加する。

https://dart.dev/language/macros

と記述されています。ちょっと何言ってるか分からないサンドウィッチマン富澤状態になってしまいましたが、なんとなく **static meta-programming** が用語として重要そうです。

この用語については、[dart-lang/language の motivation.md](https://github.com/dart-lang/language/blob/main/working/macros/motivation.md) に具体的な記述がありました。

> **Metaprogramming** refers to code that can do this—code that operates on other code as if it were data. It can take code in as parameters, reflect over it, inspect it, create it, modify it, and return it. ***Static* metaprogramming** means doing that work at compile-time.

https://github.com/dart-lang/language/blob/main/working/macros/motivation.md?plain=1#L133-L139

ざっくりまとめると **meta-programming** とは「コードを書くためのコード（≒ Dart ファイルを書くための Dart ファイル）」、そして **static** とは「コンパイル時にその生成がリアルタイムに行われること」を意味しています。まさに冒頭のデモのイメージですね。

macros は **「Dart コードをリアルタイムに生成するための Dart コード」** であることが把握出来ました。

次に

- macro の利用（既にある macro を利用する）
- macro の作成（独自に macro を作成する）

の 2 つの視点から、具体的な手順を解説していきます。

## 2. macro を利用する

まずは、macro それ自体を定義するのではなく、利用者としての macro を体験します。

今回は、ドキュメント記載にも記載されている [JsonCodable](https://pub.dev/packages/json#applying-the-jsoncodable-macro) という Json のシリアライズ・デシリアライズ用の macro を利用方法を紹介します。

### 2-1. 環境設定とプロジェクト作成

2024.12.01 現在では、macros は experimental（実験的）な機能であり、master チャンネルでしか利用出来ないので、下記コマンドでチャンネルを切り替えたうえでプロジェクトを作成します。

```sh
$ flutter channel master
$ flutter doctor -v
[✓] Flutter (Channel master, 3.27.0-1.0.pre.669, on macOS 15.1.1 24B91 darwin-arm64, locale en-JP)
    • Flutter version 3.27.0-1.0.pre.669 on channel master
    # ↓ Dart が 3.5.0-152 以降であることを確認
    • Dart version 3.7.0 (build 3.7.0-188.0.dev) 
$ flutter create macros_sample --empty
```

### 2-2. 依存関係の追加と analyze 設定の変更

次に、依存関係に [json](https://pub.dev/packages/json) パッケージを追加し

```sh
$ dart pub add json 
Resolving dependencies... 
Downloading packages... 
+ _macros 0.3.3 from sdk dart
  characters 1.3.0 (1.3.1 available)
  collection 1.19.0 (1.19.1 available)
+ json 0.20.3
+ macros 0.1.3-main.0
  matcher 0.12.16+1 (0.12.17 available)
  material_color_utilities 0.11.1 (0.12.0 available)
  meta 1.15.0 (1.16.0 available)
  test_api 0.7.3 (0.7.4 available)
Changed 3 dependencies!
6 packages have newer versions incompatible with dependency constraints.
Try `dart pub outdated` for more information.
```

`analysis_options.yaml` で experiment（実験的機能の有効化）設定を行い、pub get します。

```yaml
analyzer:
 enable-experiment:
   - macros
```

### 2-3. JsonCodable の利用

セットアップが完了したところで、JsonCodable マクロを使ってみます。
下記の単純な User クラスに

```dart
class User {
  final String name;
  final int age;
}
```

`@JsonCodable()` アノテーションを追加します。

```diff dart
+ import 'package:json/json.dart';

+ @JsonCodable()
class User {
  final String name;
  final int age;
}
```

下記のように実行可能な状態に書き換え

```diff dart
import 'package:json/json.dart';

+ void main() {
+   final userJson = {'name': 'ダイゴ', 'age': 26};
+   final user = User.fromJson(userJson);
+   print(user.toJson());
+ }

@JsonCodable()
class User {
  final String name;
  final int age;
}
```

`dart run --enable-experiment=macros lib/main.dart` を走らせると

```sh
$ dart run --enable-experiment=macros lib/main.dart
{name: ダイゴ, age: 26}
```

正常にシリアライズ・デシリアライズ出来ていることが確認出来ました。実際の VSCode でのコード Edit の様子はこんな感じです。

![](https://storage.googleapis.com/zenn-user-upload/ab643f5ccc1c-20241130.gif)

リアルタイムに生成が行われていますね。

## 3. macro を定義する

上記では、json パッケージに用意された macro を利用しましたが、次は独自の macro を定義してみようと思います。

調べてみた中で一番シンプルなサンプルとしては、K8i さんの [「タコでもわかるDartマクロ作成入門」](https://zenn.dev/yumemi_inc/articles/6d526d0546b9f7) が分かりやすかったのですが、今回はせっかくなので上述のコードに関連させて、User クラスに copyWith メソッドを生やす macro を定義してみようと思います。

### 3-1. 依存関係の追加

ドキュメントでは特に案内がないのですが、現時点では macro の作成に [macros](https://pub.dev/packages/macros) パッケージの追加が必要です。下記を追記し pub get します。

```diff yaml
environment:
  sdk: ^3.7.0-188.0.dev

dependencies:
  flutter:
    sdk: flutter
  json: ^0.20.3
+ macros: ^0.1.3-main.0 # 追加
```

### 3-2. macro クラスの作成

新しい dart ファイルを作成し、macro 修飾子をつけた `CopyWith` というクラスを作成します。

```dart
macro class CopyWith {}
```

次に、クラスに対する macro の作成に必要な `ClassDeclarationsMacro` という interface class を implement します。

```dart
import 'package:macros/macros.dart';

macro class CopyWith implements ClassDeclarationsMacro {
  @override
  Future<void> buildDeclarationsForClass(ClassDeclaration clazz, MemberDeclarationBuilder builder) {
    // TODO: implement buildDeclarationsForClass
    throw UnimplementedError();
  }
}
```

この ClassDeclarationsMacro のような implement 先は、生成対象となる Dart コードの種類（クラスなのかメンバなのか）等によって使い分けます。

全ての Macro 系 interface クラスは、下記に列挙されています。

https://github.com/dart-lang/sdk/blob/main/pkg/_macros/lib/src/api/macros.dart

### 3-3. macro クラスの実装

macro の基盤クラスが出来たところで、実際の生成コードを定義していきます。

まず前提イメージとして、上記の User クラスに copyWith メソッドを生やすならば、下記のような実装になるでしょう。

```dart
class User {
  const User({required this.name, required this.age});

  // 1
  final String name;
  final int age;

  User copyWith({String? name, int? age}) => // 2
      User(name: name ?? this.name, age: age ?? this.age); // 3
}
```
上記の `copyWith` メソッドを生成する（書く）ことを想定すると、ざっくりと 3 つのパートに分けることが出来ます。

1. 全フィールドの把握
2. それぞれのフィールドを nullable かつ optional な引数として受け取る
3. 現在のフィールド値をデフォルトとして新しいインスタンスを返す

それぞれのステップを macros を使って表現していきます。

先ほど定義した基礎となる `CopyWith` クラスに

```dart
import 'package:macros/macros.dart';

macro class CopyWith implements ClassDeclarationsMacro {
  @override
  Future<void> buildDeclarationsForClass(ClassDeclaration clazz, MemberDeclarationBuilder builder) {
    // TODO: implement buildDeclarationsForClass
    throw UnimplementedError();
  }
}
```

まずは「1. 全フィールドの把握」部分を実装します。

```diff dart
import 'package:macros/macros.dart';

macro class CopyWith implements ClassDeclarationsMacro {
  const CopyWith();

  @override
  Future<void> buildDeclarationsForClass(
    ClassDeclaration clazz,
    MemberDeclarationBuilder builder,
  ) async {
+   // 1. 対象クラスの全フィールドを取得
+   final allFields = await builder.fieldsOf(clazz);
  }
}
```

https://github.com/dart-lang/sdk/blob/5285faad677344009ae3c043b4643c4986a48284/pkg/_macros/lib/src/api/builders.dart#L117-L121

次に「2. それぞれのフィールドを nullable かつ optional な引数として受け取る」部分に変換し

```diff dart
import 'package:macros/macros.dart';

macro class CopyWith implements ClassDeclarationsMacro {
  const CopyWith();

  @override
  Future<void> buildDeclarationsForClass(
    ClassDeclaration clazz,
    MemberDeclarationBuilder builder,
  ) async {
    // 1. 対象クラスの全フィールドを取得
    final allFields = await builder.fieldsOf(clazz);
+   // 2. 全フィールドを Nullable な引数コードに変換
+   // 例: String? name
+   final namedParams = allFields
+           .map(
+             (field) => ParameterCode(
+               name: field.identifier.name,
+               type: field.type.code.asNullable,
+               keywords: const [],
+               defaultValue: null,
+             ),
+           ).toList();
  }
}
```

https://github.com/dart-lang/sdk/blob/5285faad677344009ae3c043b4643c4986a48284/pkg/_macros/lib/src/api/code.dart#L95-L106

最後に「3. 現在のフィールド値をデフォルト...」の部分も下記のように変換し

```diff dart
import 'package:macros/macros.dart';

macro class CopyWith implements ClassDeclarationsMacro {
  const CopyWith();

  @override
  Future<void> buildDeclarationsForClass(
    ClassDeclaration clazz,
    MemberDeclarationBuilder builder,
  ) async {
    // 1. 対象クラスの全フィールドを取得
    final allFields = await builder.fieldsOf(clazz);
    // 2. 全フィールドを Nullable な引数コードに変換
    // 例: String? name
    final namedParams = allFields
            .map(
              (field) => ParameterCode(
                name: field.identifier.name,
                type: field.type.code.asNullable,
                keywords: const [],
                defaultValue: null,
              ),
            ).toList();
+   // 3. 全フィールドをコピーするコードに変換
+   // 例: name: name ?? this.name
+   final copiedFields = allFields
+           .map(
+             (field) => RawCode.fromParts([
+               '${field.identifier.name}: ${field.identifier.name} ?? ',
+               field.identifier,
+             ]),
+           ).toList();
  }
}
```

https://github.com/dart-lang/sdk/blob/5285faad677344009ae3c043b4643c4986a48284/pkg/_macros/lib/src/api/code.dart#L38-L40


builder で `copyWith` というメソッド名や必要な文法と組み合わせて、生成対象のクラス内に宣言します。

```diff dart
import 'package:macros/macros.dart';

macro class CopyWith implements ClassDeclarationsMacro {
  const CopyWith();

  @override
  Future<void> buildDeclarationsForClass(
    ClassDeclaration clazz,
    MemberDeclarationBuilder builder,
  ) async {
    // 1. 対象クラスの全フィールドを取得
    final allFields = await builder.fieldsOf(clazz);
    // 2. 全フィールドを Nullable な引数コードに変換
    // 例: String? name
    final namedParams = allFields
            .map(
              (field) => ParameterCode(
                name: field.identifier.name,
                type: field.type.code.asNullable,
                keywords: const [],
                defaultValue: null,
              ),
            ).toList();
    // 3. 全フィールドをコピーするコードに変換
    // 例: name: name ?? this.name
    final copiedFields = allFields
            .map(
              (field) => RawCode.fromParts([
                '${field.identifier.name}: ${field.identifier.name} ?? ',
                field.identifier,
              ]),
            ).toList();
+   // copyWith メソッドを対象クラス内で宣言
+   builder.declareInType(
+     DeclarationCode.fromParts([
+       clazz.identifier,
+       ' copyWith({',
+       // 引数部分
+       ...namedParams.joinAsCode(', '),
+       '}) => ',
+       clazz.identifier,
+       '(',
+       // コピー部分
+       ...copiedFields.joinAsCode(', '),
+       ');',
+     ]),
+   );
  }
}
```

これで `copyWith` メソッドを生やす macro が作成出来ました。

実際に `User` クラスに `@CopyWith()` アノテーションをつけてみると、リアルタイムで augment コードが生成されており、

![](https://storage.googleapis.com/zenn-user-upload/62a3c3f3a796-20241201.gif)

```dart:生成元ファイル
import 'package:macros_sample/macro/copy_with.dart';

@CopyWith()
class User {
  const User({required this.name, required this.age});

  final String name;
  final int age;
}
```

```dart:生成されたファイル
part of 'package:macros_sample/main.dart';

import 'package:macros_sample/main.dart' as prefix0;
import 'dart:core' as prefix1;

augment class User {
prefix0.User copyWith({prefix1.String? name, prefix1.int? age}) => prefix0.User(name: name ?? this.name, age: age ?? this.age);
}
```

`copyWith` メソッドも期待した挙動が確認できました🎉

```dart
import 'package:macros_sample/macro/copy_with.dart';

void main() {
  final user = User(name: 'ダイゴ', age: 26);
  print(user.name);
  print(user.age);
  final copy1 = user.copyWith(name: 'イツコ');
  print(copy1.name); // イツコ
  print(copy1.age); // 26
  final copy2 = user.copyWith(age: 27);
  print(copy2.name); // ダイゴ
  print(copy2.age); // 27
}

@CopyWith()
class User {
  const User({required this.name, required this.age});
  final String name;
  final int age;
}
```

今回作ってみた macro の他にも、GitHub で [`lang:dart "macro class"`](https://github.com/search?q=lang%3Adart+%22macro+class%22&type=code) と検索すると、既にいろいろな方が結構面白い macro を作成していたり、[macros/example/lib](https://github.com/dart-lang/language/tree/ed924b8826913bb767084405a62b5fa09b74aa21/working/macros/example/lib) フォルダでも inherited_widget や auto_dispose など Flutter 関連の macro が実験的に用意されているので、自分で macro を作成する際は覗いてみると良いかも知れません。

## まとめ

今回は、Dart の macros 概要とその利用・定義方法について解説しました。

現状は挙動が不安定だったり（macro 修飾子をつけたコードのフォーマットが効かない等）とまだ実験的な機能ではあるものの、冒頭に紹介した通り 2025 年初頭での stable リリースを目指しているとのことなので、年内にイメージが掴めて良かったです。

また、冒頭の動画のまとめ部分でちゅーやんさんが仰っている事とも少し関連するのですが、macro はボイラープレートコードの削減や開発生産性の向上に寄与出来るものではあるものの、なんでもかんでも macro で省略・独自定義をしてしまうと、かえって生成元のコードに対してどんな実装が隠蔽されているのかチェックする必要が出てきたり、macro 自体のメンテナンスにコストがかかってしまったりと本末転倒な状態になってしまうことも想像出来るので、ある程度のバランス感覚が必要だなと感じました。

とはいえ、今までとは一風変わった世界観の新機能の登場・それによって開発フローが大きく変わる可能性があるとなると、やはりワクワクしますね。

今回のデモコードは GitHub 上で公開していますので、よかったら参考にして頂ければと思います。

https://github.com/DaigoWakabayashi/macros_sample

最後までご覧いただき、ありがとうございました。

## 参考

https://dart.dev/language/macros

https://github.com/dart-lang/language/tree/main/working/macros