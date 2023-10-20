---
title: "【Flutter】カメラ機能を実装する"
emoji: "📷"
type: "tech"
topics: ["Flutter", "iOS", "Android"]
publication_name: "flutteruniv_dev"
published: true
---

## はじめに

こんにちは、[ダイゴ](https://twitter.com/Mamushi_journey)です。

初学者の頃からずっとお世話になっている[Flutter 大学](https://flutteruniv.com/)で、今年の 4 月から、 zoom での質問回答を行う講師として活動を始めました。

先日の質問 zoom で、カメラ機能関連の質問があり、「そういえばカメラ機能の実装したことないな」と思い、簡単なカメラアプリを作成し、記事にしました。

サンプルプロジェクトも公開しているので、こちらも合わせて参考にしていただけると幸いです。

https://github.com/DaigoWakabayashi/flutter_camera_example

## この記事でやっていること

![](https://storage.googleapis.com/zenn-user-upload/c3fe91fea82b-20220414.gif)

- カメラを起動
- 写真を撮る
- 画像として表示する

## 目次

1. パッケージの追加
2. 利用可能なカメラを取得
3. プレビューを表示する（`CameraController`・`CameraPreview` ）
4. 写真を取る
5. 画像を表示（`Image` Widget）

https://docs.flutter.dev/cookbook/plugins/picture-using-camera

## 1. パッケージの追加

まずは、カメラ機能に関連する 3 つのパッケージをインポートします。

- [camera](https://pub.dev/packages/camera)
  デバイス上のカメラを操作するためのツール。

- [path_provider](https://pub.dev/packages/path_provider)
  画像の保存場所を見つけ出してくれる。

- [path](https://pub.dev/packages/path)
  画像の保存場所を作成してくれる。

上記のパッケージを、`pubspec.yaml` ファイルに追加し、`flutter pub get` します。（fvm を使っている場合は prefix をつけましょう）

```yaml
dependencies:
  flutter:
    sdk: flutter
  camera: ^0.9.4+19
  path_provider: ^2.0.9
  path: ^1.8.0
```

[camera](https://pub.dev/packages/camera)パッケージを正しく動作させるためには

- Android の minSdkVersion を 21 以上に設定
- iOS の `ios/Runner/Info.plist` に権限要求時の文言を追加

する必要があるので、各々設定していきます。

#### Android の設定

`android/app/build.gradle` の minSdkVersion を 21 に設定します。

```diff
    defaultConfig {
        // TODO: Specify your own unique Application ID (https://developer.android.com/studio/build/application-id.html).
        applicationId "io.github.daigowakabayashi.camera"
-        minSdkVersion flutter.minSdkVersion
+        minSdkVersion 21
        targetSdkVersion flutter.targetSdkVersion
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
    }
```

#### iOS の設定

`ios/Runner/Info.plist` に権限要求（〇〇に「カメラ」の利用を許可しますか？のダイアログ）時の文言を追加します。

```diff
 <?xml version="1.0" encoding="UTF-8"?>
 <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
 <plist version="1.0">
 <dict>
   ~~~~~~~~~~~ 中略 ~~~~~~~~~~~
+	<key>NSCameraUsageDescription</key>
+	<string>カメラを使う理由・用途を記述（ここをちゃんと書かないと、Appleのレビューでリジェクトされます）</string>
+	<key>NSMicrophoneUsageDescription</key>
+	<string>マイクを使う理由・用途を記述（ここをちゃんと書かないと、Appleのレビューでリジェクトされます）</string>
 </dict>
 </plist>
```

↑ は、こんな感じで表示されます。

![](https://storage.googleapis.com/zenn-user-upload/186202038ab4-20220414.png =250x)

## 2. 利用可能なカメラを取得

コードを書いていきます。

まず、ビルドされている端末で利用できるカメラを取得します。
`availableCameras()` という関数を使うと、利用可能なカメラを取得することができます。

```dart
import 'package:camera/camera.dart';
import 'package:flutter/material.dart';

Future<void> main() async {
  // main 関数内で非同期処理を呼び出すための設定
  WidgetsFlutterBinding.ensureInitialized();

  // デバイスで使用可能なカメラのリストを取得
  final cameras = await availableCameras();

  // 利用可能なカメラのリストから特定のカメラを取得
  final firstCamera = cameras.first;

  // 取得できているか確認
  print(firstCamera);

  runApp(const MyApp());
}
```

CameraDescription が print されていれば OK です。

```terminal
// iOS 実機
flutter: CameraDescription(com.apple.avfoundation.avcapturedevice.built-in_video:0, CameraLensDirection.back, 90)
// Android 実機
flutter: CameraDescription(0, CameraLensDirection.back, 90)
```

ちなみに私の環境では、iOS Simulator で Camera が取得できず、`cameras.first` 部分で `StateError (Bad state: No element)` のエラーが発生しました。

Android Emulator は試していませんが、エミュレータ等でカメラが起動できたとしても何が映るんだという感じなので、実機でデバッグすることをおすすめします。

## 3. プレビューを表示する

プレビューを表示するには、`CameraController`（撮影などのメソッドを持つクラス）の初期化が必要です。

写真撮影用の画面を StatefulWidget で定義し、initState 内で、`CameraController`の作成・初期化を行います。

```dart
/// 写真撮影画面
class TakePictureScreen extends StatefulWidget {
  const TakePictureScreen({
    Key? key,
    required this.camera,
  }) : super(key: key);

  final CameraDescription camera;

  @override
  TakePictureScreenState createState() => TakePictureScreenState();
}

class TakePictureScreenState extends State<TakePictureScreen> {
  late CameraController _controller;
  late Future<void> _initializeControllerFuture;

  @override
  void initState() {
    super.initState();

    _controller = CameraController(
      // カメラを指定
      widget.camera,
      // 解像度を定義
      ResolutionPreset.medium,
    );

    // コントローラーを初期化
    _initializeControllerFuture = _controller.initialize();
  }

  @override
  void dispose() {
    // ウィジェットが破棄されたら、コントローラーを破棄
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    // NEXT：プレビュー画面を表示
    return SizedBox();
  }
}

```

そして、`CameraPreview` ウィジェットに、先程初期化した controller を渡すことで、プレビューを表示することができます。

```dart

  @override
  Widget build(BuildContext context) {
    // FutureBuilder で初期化を待ってからプレビューを表示（それまではインジケータを表示）
    return FutureBuilder<void>(
      future: _initializeControllerFuture,
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.done) {
          return CameraPreview(_controller);
        } else {
          return const Center(child: CircularProgressIndicator());
        }
      },
    );
  }
```

プレビューが表示できました。（私の今の作業環境が写っています）

![](https://storage.googleapis.com/zenn-user-upload/da1927bf033c-20220414.jpeg =250x)

## 4. 写真を取る

CameraController の `takePicture` メソッドで、写真を撮影することが出来ます。

FloatingActionButton を設置し、onPressed 内で `takePicture` を呼び出します。

```dart
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: FutureBuilder<void>(
          future: _initializeControllerFuture,
          builder: (context, snapshot) {
            if (snapshot.connectionState == ConnectionState.done) {
              return CameraPreview(_controller);
            } else {
              return const CircularProgressIndicator();
            }
          },
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          // 写真を撮る
          final image = await _controller.takePicture();
          // path を出力
          print(image.path);
        },
        child: const Icon(Icons.camera_alt),
      ),
    );
  }
```

ボタンを押すといつもの「カシャッ」というシャッター音が鳴り、以下のような path が出力されます。

```
flutter: /var/mobile/Containers/Data/Application/xxxxxxxxxxxxxxxxxx/Documents/camera/pictures/CAP_xxxxxxxxxxxxxxx.jpg
```

最後に、こちらの Path を使って画像を表示してみます。

## 5. 画像を表示（`Image` ウィジェット）

撮影した写真を表示する画面を作成し、

```dart
// 撮影した写真を表示する画面
class DisplayPictureScreen extends StatelessWidget {
  const DisplayPictureScreen({Key? key, required this.imagePath})
      : super(key: key);

  final String imagePath;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('撮れた写真')),
      body: Center(child: Image.file(File(imagePath))),
    );
  }
}

```

写真撮影後に上記の画面へ遷移するようにします。

```dart
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          // 写真を撮る
          final image = await _controller.takePicture();
          // 表示用の画面に遷移
          await Navigator.of(context).push(
            MaterialPageRoute(
              builder: (context) => DisplayPictureScreen(imagePath: image.path),
              fullscreenDialog: true,
            ),
          );
        },
        child: const Icon(Icons.camera_alt),
      ),
```

これでカメラ撮影と表示が出来ました 🎉

![](https://storage.googleapis.com/zenn-user-upload/c3fe91fea82b-20220414.gif)

:::details コード全体（124 行）

```dart
import 'dart:io';

import 'package:camera/camera.dart';
import 'package:flutter/material.dart';

Future<void> main() async {
  // main 関数内で非同期処理を呼び出すための設定
  WidgetsFlutterBinding.ensureInitialized();
  // デバイスで使用可能なカメラのリストを取得
  final cameras = await availableCameras();
  // 利用可能なカメラのリストから特定のカメラを取得
  final firstCamera = cameras.first;
  runApp(MyApp(camera: firstCamera));
}

class MyApp extends StatelessWidget {
  const MyApp({
    Key? key,
    required this.camera,
  }) : super(key: key);

  final CameraDescription camera;

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Camera Example',
      theme: ThemeData(),
      home: TakePictureScreen(camera: camera),
    );
  }
}

/// 写真撮影画面
class TakePictureScreen extends StatefulWidget {
  const TakePictureScreen({
    Key? key,
    required this.camera,
  }) : super(key: key);

  final CameraDescription camera;

  @override
  TakePictureScreenState createState() => TakePictureScreenState();
}

class TakePictureScreenState extends State<TakePictureScreen> {
  late CameraController _controller;
  late Future<void> _initializeControllerFuture;

  @override
  void initState() {
    super.initState();

    _controller = CameraController(
      // カメラを指定
      widget.camera,
      // 解像度を定義
      ResolutionPreset.medium,
    );

    // コントローラーを初期化
    _initializeControllerFuture = _controller.initialize();
  }

  @override
  void dispose() {
    // ウィジェットが破棄されたら、コントローラーを破棄
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: FutureBuilder<void>(
          future: _initializeControllerFuture,
          builder: (context, snapshot) {
            if (snapshot.connectionState == ConnectionState.done) {
              return CameraPreview(_controller);
            } else {
              return const CircularProgressIndicator();
            }
          },
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          // 写真を撮る
          final image = await _controller.takePicture();
          // 表示用の画面に遷移
          await Navigator.of(context).push(
            MaterialPageRoute(
              builder: (context) => DisplayPictureScreen(imagePath: image.path),
              fullscreenDialog: true,
            ),
          );
        },
        child: const Icon(Icons.camera_alt),
      ),
    );
  }
}

// 撮影した写真を表示する画面
class DisplayPictureScreen extends StatelessWidget {
  const DisplayPictureScreen({Key? key, required this.imagePath})
      : super(key: key);

  final String imagePath;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('撮れた写真')),
      body: Center(child: Image.file(File(imagePath))),
    );
  }
}
```

:::

## さいごに

なんだかネイティブ依存の機能実装難しそう、、と思っていましたが、
パッケージ使うと意外と簡単にカメラ機能を実装できるもんだなぁと感じました。

`CameraController` 内部を見ている感じだと、写真撮影の他にも

- ズーム調節
- ピント設定
- カメラ情報のリアルタイムリッスン（Stream）

などもすぐ出来そうな感じでした。

便利なパッケージを開発してくださってる方々に感謝ですね。（ぜひ [Star🌟](https://github.com/flutter/plugins/tree/main/packages/camera/camera)を！）

サンプルプロジェクトも公開しているので、ぜひ色々と試してみていただけると幸いです。

https://github.com/DaigoWakabayashi/flutter_camera_example

最後までご覧頂きありがとうございました。

## 参考

https://docs.flutter.dev/cookbook/plugins/picture-using-camera
