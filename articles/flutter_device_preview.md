---
title: "【Flutter】device_preview を使って、様々なデバイス上でのレイアウトをサクッと確認する"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Flutter','Dart']
published: false
---

## はじめに

こんにちは、[ダイゴ](https://twitter.com/mamushi_journey)です。
様々なデバイス上でのレイアウト確認がサクッと出来るパッケージ、「[device_preview](https://pub.dev/packages/device_preview)」が気になっていたので、触ってみて記事にしました。
どなたかの参考になれば幸いです。

## 目次

1. **パッケージのインストール**
2. **セットアップ**
3. **主な機能紹介**

### 1. パッケージのインストール


pubspec.yaml ファイルに device_preview を追加し、`flutter pub get` 

```yaml
dependencies:
  device_preview: <任意のバージョン>
```

https://pub.dev/packages/device_preview

### 2. セットアップ

main 関数内のアプリウィジェットを、DevicePreview でラップします。

```dart
import 'package:device_preview/device_preview.dart';

void main() => runApp(
  DevicePreview(
    enabled: !kReleaseMode,
    builder: (context) => MyApp(), // Wrap your app
  ),
);

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      useInheritedMediaQuery: true,
      locale: DevicePreview.locale(context),
      builder: DevicePreview.appBuilder,
      theme: ThemeData.light(),
      darkTheme: ThemeData.dark(),
      home: const HomePage(),
    );
  }
}

```

MaterialApp 内で以下の 3つを指定することで、MediaQuery 内の値も DevicePreview 上のものが取れるようになっています。

- MaterialApp 内の
  - useInherited MediaQuery を true に変更
  - builder に DevicePreview.appBuilder を指定
  - locale に DevicePreview.locale(context) を指定

![](https://storage.googleapis.com/zenn-user-upload/6926f971b5b8-20220216.gif =250x)


### 3. 主な機能紹介

| デバイスの選択                                                                       | ダーク/ライトモード                                                                  |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| ![](https://storage.googleapis.com/zenn-user-upload/291bd3aeb8d6-20220215.gif =300x) | ![](https://storage.googleapis.com/zenn-user-upload/a92f895cd7e4-20220215.gif =300x) |
| iOS・Android・macOS・windowsなどのデバイスから選択が可能です。                       | 仮想デバイスのライトモード・ダークモードを切り替えることが出来ます。                 |

| 縦向き / 横向き                                                                      | 通常ビルドとの切り替え                                                                                   |
| ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| ![](https://storage.googleapis.com/zenn-user-upload/e24fe08dd344-20220215.gif =300x) | ![](https://storage.googleapis.com/zenn-user-upload/bac950b06d0a-20220215.gif =300x)                     |
| 仮想デバイスのオリエンテーション（縦向き・横向き）を切り替えることが出来ます。       | 右下のトグルスイッチで、通常のビルドモードと仮想デバイスでのプレビューモードを切り替えることが出来ます。 |


他にも location の変更や、キーボード表示時のプレビュー等の機能もあります。
特定の環境下でのレイアウト崩れなどの検証は、環境の再現に時間がかかったりするので、上記のように手軽に環境再現が出来ると楽ですね。

## さいごに


クロスプラットフォーム開発では、レイアウト等で配慮しなくてはいけない事項も必然的に増えてくるので、いちいちシュミレータをセットアップしてビルドする手間が省けるのはとても便利だなと感じました。


ただ、公式のREADMEにも記載されているのですが、DevicePreview は全てのデバイスの動きを完全にエミュレート出来るわけではない、という点は注意が必要です。

> There are some aspects of mobile devices that Device Preview will never be able to simulate. When in doubt, your best bet is to actually run your app on a real device.

あくまで開発時の参考程度にとどめ、怪しい部分は実機やシュミレータで確認するのが良さそうですね。
思ったより手軽に導入できたので、今後使っていきながら新たな発見があれば追記しようと思います。

最後までご覧頂き、ありがとうございました。

## 参考

https://pub.dev/packages/device_preview

https://github.com/aloisdeniel/flutter_device_preview