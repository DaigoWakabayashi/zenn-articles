---
title: Flutter × FCMでプッシュ通知を実装する
emoji: "🐯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Flutter','Firebase','CloudMessaging']
published: true
---

## はじめに

こんにちは、[ダイゴ](https://twitter.com/Mamushi_journey)です。

Flutter で [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging?hl=ja) を用いた通知の実装をする機会があったので、備忘録として記事にしました。

本記事では、**パッケージのインポート** **〜** **テスト通知を受け取る** までの流れをまとめています。
どなたかの参考になれば幸いです。


:::message
前提として

1. [Apple Developer](https://developer.apple.com/jp/programs/) アカウント（99米ドル ≒ 11,800円 / 年）
2. iOS の実機デバイス（iOS シミュレータでは通知が動作しないので）

が必要です。
:::

## 1. パッケージのインポート

```yaml
dependencies:
  firebase_messaging: ^11.2.8
```
プロジェクトの `pubspec.yaml`ファイルに、任意の[`firebase_messaging`](https://pub.dev/packages/firebase_messaging)バージョンを追記し、`pub get` します。

## 2. iOS のセットアップ

ここが少し面倒です。

- **① Xcode の設定**
- **② APNs と FCM の繋ぎこみ**
   - APNs キーを Firebase に登録
   - App Identifier の生成
   - Provisioning Profile の登録

の順で解説していきます。

### ① Xcode の設定
まずは、Xcode の capability（アプリが持つ機能）に

- **「Push Notifications」**
- **「Background Modes」**

を追加していきます。

Xcode を開き、

![](https://storage.googleapis.com/zenn-user-upload/f33418f54615-20220228.png)

**①Runner** > **②Target Runner** > **③Signing & Capabilities** を選択します。

![](https://storage.googleapis.com/zenn-user-upload/dd3a497dd08c-20220228.png)

その後、**①「＋capability」ボタン** から、**②Push Notifications** を選択することで、通知の設定が有効になります。

![](https://storage.googleapis.com/zenn-user-upload/b7a70d90983f-20220228.png)

Background Modes も同様に、**①「＋capability」ボタン** から、**②Background Modes** を選択し有効化し、

![](https://storage.googleapis.com/zenn-user-upload/d1253df0f424-20220228.png)

詳細設定として、**Background fetch** と **Remote notifications** の両方のサブモードを有効にします。

![](https://storage.googleapis.com/zenn-user-upload/91e3e41673e4-20220228.png)

これで Xcode 側の設定は終了です。

### ② APNs と FCM の繋ぎこみ

APNs(Apple Push Notification service)と FCM（Firebase Cloud Messaging)の繋ぎこみをしていきます。

1. APNs キーを Firebase に登録する
2. App Identifier の生成
3. Provisioning Profile の生成

#### 1. APNs キーを Firebase に登録する

[AppleDeveloper のアカウントページ](https://developer.apple.com/account/#!/overview)を開き、**Certificates, Identifiers & Profiles**を選択します。

![](https://storage.googleapis.com/zenn-user-upload/8007e75ff6e4-20220228.png)

以下のようなページに飛ぶので、**Keys** から **Create Key** を選択し、

![](https://storage.googleapis.com/zenn-user-upload/3f942212f98f-20220228.png)

Key Name に適当な名前をつけて、APNsにチェックを入れます。

![](https://storage.googleapis.com/zenn-user-upload/38bf166c252c-20220228.png)

Continue を押すと確認画面が表示されるので、内容を確認し、Resister。

![](https://storage.googleapis.com/zenn-user-upload/05973915600d-20220228.png)

**① Key ID をコピー**しておき、**②ファイルをダウンロード**します。（再ダウンロードは出来ない点に注意しましょう）


![](https://storage.googleapis.com/zenn-user-upload/2f5d097803e9-20220228.png)
これで、APNs の Key を生成出来ました。


次に、生成した Key を Firebase プロジェクトに登録していきます。
[**Firebase コンソール**](https://console.firebase.google.com/u/0/) から、**プロジェクトの設定 > Cloud Messaging** を開きます。

![](https://storage.googleapis.com/zenn-user-upload/fe807fd46781-20220228.png)

**Apple アプリの構成** > **APNs 認証キー** > **アップロード** から

![](https://storage.googleapis.com/zenn-user-upload/aade6a4d2ee7-20220228.png)

- **① Key ファイル（先程生成したもの）**
- **② Key ID（先程コピーしたもの）**
- **③ Team ID（[AppleDeveloper](https://developer.apple.com/account/resources/certificates/list)ページ右上の8桁くらいのID）**

を入力し、アップロードします。

![](https://storage.googleapis.com/zenn-user-upload/accc2b271abf-20220228.png)

これで、APNs キーを Firebase プロジェクトに登録出来ました。

![](https://storage.googleapis.com/zenn-user-upload/c21aa349fdbd-20220228.png)

#### 2. App Identifier の生成

次に、App Identifier（バンドルIDやアプリの機能などをまとめた識別情報）を生成します。

[**Certificates, Identifiers & Profiles**](https://developer.apple.com/account/resources/certificates/list) > **Identifiers** > **＋ ボタン**を選択

![](https://storage.googleapis.com/zenn-user-upload/aedab4220535-20220228.png)

**App IDs** > **App** と選択し、説明（なんでもいい）と bundleID を入力します。

![](https://storage.googleapis.com/zenn-user-upload/8dd56ad6b1e1-20220228.png)

**capability** の部分で **Push Notifications** を設定し、内容を確認して Resister。

![](https://storage.googleapis.com/zenn-user-upload/f1123c47a8b2-20220228.png)

これで、App Identifier の登録が完了しました。

#### 3. Provisioning Profile の生成

最後に Provisioning Profile（AppID や Keyなどの情報をまとめたファイル）を生成し、アプリ側に登録していきます。

[**Certificates, Identifiers & Profiles**](https://developer.apple.com/account/resources/certificates/list) > **Profiles** > ＋ ボタンを選択

![](https://storage.googleapis.com/zenn-user-upload/bb8d69e178b5-20220228.png)


開発版であれば **iOS App Development**、本番用であれば **App Store** を選択します。

![](https://storage.googleapis.com/zenn-user-upload/991a2bb1eeea-20220228.png)


AppID は、先程生成した App ID を選択します。

![](https://storage.googleapis.com/zenn-user-upload/be7a94cc0c3c-20220228.png)

任意の Certificates（開発者の証明書）を選択し、

![](https://storage.googleapis.com/zenn-user-upload/f6723f626a1b-20220228.png)

デバイスも登録している場合は選択します。

![](https://storage.googleapis.com/zenn-user-upload/ed86141f7081-20220228.png)

Profile の名前を記入し、

![](https://storage.googleapis.com/zenn-user-upload/a9d4b63ae0ed-20220228.png)

内容を確認してから Download します。

![](https://storage.googleapis.com/zenn-user-upload/5a35079e2c74-20220228.png)

これで、通知の設定が入った Profile が生成出来たので、Xcode に登録します。

Xcode の **①Runner** > **②Target Runner** > **③Signing & Capabilities** を開き、**Provisioning Profile** の欄で、先程ダウンロードした Profile を選択します。


![](https://storage.googleapis.com/zenn-user-upload/5e15b35eef15-20220228.png)


少し長かったですが、これで一通りのセットアップが完了しました。

:::details 画像付き通知を実装したい場合
追加の設定が必要です。
今回は解説できませんでしたが、[公式Doc](https://firebase.flutter.dev/docs/messaging/apple-integration/#advanced-optional-allowing-notification-images)で詳しく解説されているので、そちらをご覧ください🙏
:::



## 3. 通知の送信

### 通知権限のリクエスト

ようやくアプリ側の実装を進めて行きます。
main.dart 内の関数で通知の権限をリクエストしてみます。


```dart
    // FCM の通知権限リクエスト
    final messaging = FirebaseMessaging.instance;
    await messaging.requestPermission(
      alert: true,
      announcement: false,
      badge: true,
      carPlay: false,
      criticalAlert: false,
      provisional: false,
      sound: true,
    );
```

すると、アプリのビルド時に権限要求のダイアログが表示されました。

![](https://storage.googleapis.com/zenn-user-upload/5340ef070c9c-20220228.png =250x)

権限のステータスは以下の 4 つで、`requestPermission`メソッドの返り値である、`NotificationSettings`の`authorizationStatus` プロパティから確認することが出来ます。

```dart
enum AuthorizationStatus {
  /// 承認
  authorized,

  /// 拒否
  denied,

  /// 未定（requestPermission するまではこの状態）
  notDetermined,

  /// 暗黙的な仮承認（iOS 12 以上の場合、requestPermission 時に provisional: true を指定することで、ダイアログ無しに通知を送信することが出来る）
  /// 詳細 → https://firebase.flutter.dev/docs/messaging/permissions/#provisional-authorization
  provisional,
}
```

https://firebase.flutter.dev/docs/messaging/permissions/


権限が承認したら、最後に、トークンを使ったテスト通知で動作を確認してみます。

### トークンを使ったテスト通知

トークン（通知を送るためのデバイスの識別子）を取得し、Firebase コンソールからテスト通知を送ってみます。
`getToken` メソッドを呼び出し、返り値の token を print してみます。

```dart
    // FCM の通知権限リクエスト
    final messaging = FirebaseMessaging.instance;
    await messaging.requestPermission(
      alert: true,
      announcement: false,
      badge: true,
      carPlay: false,
      criticalAlert: false,
      provisional: false,
      sound: true,
    );

    // トークンの取得
    final token = await messaging.getToken();
    print('🐯 FCM TOKEN: $token');
```

![](https://storage.googleapis.com/zenn-user-upload/4245a64ce299-20220228.png)

トークンが print されたので、こちらを元に Firebase コンソールから通知を送ります。
トークンをコピーしておき、 Firebase コンソールの Cloud Messaging タブを選択。

![](https://storage.googleapis.com/zenn-user-upload/0347b5e8fdf6-20220228.png)

テスト用の通知の文章を入力し、

![](https://storage.googleapis.com/zenn-user-upload/26e50d3f5fc4-20220228.png)

先程コピーした token を入力し、送信してみます。

![](https://storage.googleapis.com/zenn-user-upload/a3de03d9d666-20220228.png)


アプリを **Background** 状態にすることで、通知が表示されました 🎉

![](https://storage.googleapis.com/zenn-user-upload/7ac3bc1ad2c2-20220228.jpeg =250x)

**Foregroud** 状態で通知を表示するには少し設定が必要なので、[公式ドキュメント](https://firebase.flutter.dev/docs/messaging/notifications#foreground-notifications)を参考に実装をしてみて下さい🙏

| **状態**       | **説明**                                                                                                                                                                                                                   |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Foreground** | アプリケーションを開いているとき、表示中＆使用中の状態。                                                                                                                                                                   |
| **Background** | アプリケーションを開いているが、バックグラウンドにある（最小化されている）状態。ユーザーがデバイスの「ホーム」ボタンを押したり、アプリケーションスイッチャーで別のアプリケーションに切り替えたりした場合にこの状態になる。 |
| **Terminated** | 端末がロックされているとき、アプリケーションを起動していない状態。                                                                                                                                                         |


## さいごに

通知機能は、モバイルアプリにおける主要な機能であり、初学者の方でも個人アプリ等に組み込みたいケースが多いと思います。

一方で、実装のハードルが少し高い（特に iOS のセットアップ周り）ので、この記事が少しでもそういった方々の参考になれば嬉しいです。🙏

最後まで読んで頂き、ありがとうございました。

## 参考

https://firebase.google.com/docs/cloud-messaging

https://firebase.flutter.dev/docs/messaging/usage


## お世話になっているコミュニティ

https://kboyflutteruniv.com/

https://community.camp-fire.jp/projects/view/280040