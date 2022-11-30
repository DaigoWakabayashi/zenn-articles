---
title: "【FlutterFire × Stripe】flutter_stripe パッケージで楽々カード決済"
emoji: "💳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Flutter','Firebase','Stripe','TypeScript','Nodejs']
published: false
---

この記事は、[Flutter 大学アドベントカレンダー 2022](https://qiita.com/advent-calendar/2022/flutteruniv) 1日目の記事です。

## はじめに

はじめまして、ダイゴです。

今回は [**flutter_stripe**](https://pub.dev/packages/flutter_stripe) パッケージを触ってみました。

========================== サンプル ==========================


## 目次

1. パッケージのインストール・両OSでのセットアップ
2. Stripe アカウントの作成 & Publishable Key の発行
3. サーバー側の実装（Cloud Functions）
4. クライアント側の実装（Flutter）
```
[✓] Flutter (Channel stable, 3.3.5, on macOS 13.0)
    • Flutter version 3.3.5 on channel stable
    • Dart version 2.18.2
```

## 1. パッケージのインストール・両OSでのセットアップ

まずはパッケージのセットアップをしていきます。

### pubspec.yaml

プロジェクトの pubspec.yaml に、 [flutter_stripe](https://pub.dev/packages/flutter_stripe) を追加して `pub get` します。

```diff yaml
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.2
+ flutter_stripe: ^7.0.0
```

### iOSの設定

Xcode を開いて
- Runner > Project Runner > Deployment Target > iOS Deployment Target
- Runner > Targets Runner > Minimum Deployments > Minimum Deployments > iOS 

を 12.0 以上に指定します。

### Androidの設定

1. **Android SDK のバージョン指定**

`android/app/build.gradle` を以下のように指定します。

```diff gradle
android {
-   compileSdkVersion flutter.compileSdkVersion
+   compileSdkVersion 33
    ndkVersion flutter.ndkVersion

    ~~~~~~~~~~~ 中略 ~~~~~~~~~~~

    defaultConfig {
        applicationId "com.example.flutter_stripe_example"
-       minSdkVersion flutter.minSdkVersion
+       minSdkVersion 21
        targetSdkVersion flutter.targetSdkVersion
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
    }
```

ちなみに、flutter.〇〇SdkVersion の値は `flutter_sdk/packages/flutter_tools/gradle/flutter.gradle` で確認可能なので、そちらの値が ↑ 以上になっていれば flutter.〇〇SdkVersion のままで問題ありません。

2. **Kotlin のバージョン指定**

`android/build.gradle` にある kotlin_version を 1.5.0 に以上に指定します。

```diff gradle
buildscript {
+   ext.kotlin_version = '1.6.21' // ← 1.5.0 以上に指定
    repositories {
        google()
        mavenCentral()
    }
```

3. **style.xml の変更**

`android/app/src/main/res/values/styles.xml` と `android/app/src/main/res/values-night/styles.xml` を以下のように変更します。

```diff xml
 <?xml version="1.0" encoding="utf-8"?>
 <resources>
    ~~~~~~~~~~~ 中略 ~~~~~~~~~~~
-    <style name="NormalTheme" parent="@android:style/Theme.Light.NoTitleBar">
+    <style name="NormalTheme" parent="Theme.MaterialComponents">
         <item name="android:windowBackground">?android:colorBackground</item>
     </style>
 </resources>
```

4. **gradle のバージョン確認**

`android/build.gradle` のビルドツールのバージョンが新しいものになっているか確認し、古ければ修正します。（最新は[こちら](https://mvnrepository.com/artifact/com.android.tools.build/gradle?repo=google)から確認できますが、現時点では 7 以上になっていれば問題はなさそうです）

```diff gradle
    dependencies {
+       classpath 'com.android.tools.build:gradle:7.1.2' // もし 4 系とかになっていたら修正
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}
```

5. **MainActivity.kt の変更**

`android/app/src/main/kotlin/<アプリidのpath>/MainActivity.kt` で、FlutterActivity → FlutterFragmentActivity に変更します。

```diff kt
  package com.example.flutter_stripe_example
  
  import io.flutter.embedding.android.FlutterActivity
+ import io.flutter.embedding.android.FlutterFragmentActivity

- class MainActivity: FlutterActivity() {
+ class MainActivity: FlutterFragmentActivity() {
  }
```

## 2. Stripe アカウントの作成 & Publishable Key の発行

### Stripe アカウントの作成

以下のページにアクセスし、アカウント作成 or ログインします。

https://dashboard.stripe.com/login?redirect=%2Fdashboard

ダッシュボードから新規の Stripe アカウント（≒ アプリのアカウント）を発行します。

![](https://storage.googleapis.com/zenn-user-upload/7c6cde6eaf0c-20221105.png)

![](https://storage.googleapis.com/zenn-user-upload/634ed8bc864d-20221105.png)

すると、ダッシュボードから PublishableKey が確認できます。

![](https://storage.googleapis.com/zenn-user-upload/693d237aa62e-20221105.png)


:::message
- Stripe の新規アカウント発行時は「テスト環境」となっており、テストのクレジットカードを使って擬似的な決済を走らせることができる状態です。
- 実際にお金を動かすには [本番環境への利用申請](https://dashboard.stripe.com/account/onboarding/business-structure) が必要ですが、本記事では省略しています。
:::

## 3. サーバー側の実装（Cloud Functions）

flutter_stripe で想定されている決済手法には
- [PaymentSheet（と PaymentIntent）](https://docs.page/flutter-stripe/flutter_stripe/sheet)
- [CardForm（と PaymentIntent）](https://docs.page/flutter-stripe/flutter_stripe/card_field)

の2つがあるのですが、公式が推奨している & サーバー側の実装が比較的簡単な [PaymentSheet](https://docs.page/flutter-stripe/flutter_stripe/sheet) による決済を実装していきます。

どちらにせよ、サーバー側で [PaymentIntent](https://stripe.com/docs/api/payment_intents) のエンドポイントを作成する必要があるので、今回は Cloud Functions を使った上記のエンドポイント作成例を紹介します。

### Cloud Functions の環境構築

少し雑になってしまうのですが、Cloud Functions の実行環境については、公式のドキュメントを参考に構築してください。

https://firebase.google.com/docs/functions/get-started

サンプルプロジェクトでは ESlint と Prettier で静的解析およびフォーマットを行っていますが、必須ではないのと設定が複雑なので、参考資料を載せておきます。

https://zenn.dev/big_tanukiudon/articles/c1ab3dba7ba111#eslint-%E5%B0%8E%E5%85%A5-%EF%BC%86-%E5%AE%9F%E8%A1%8C

https://maku.blog/p/yfow6dk/

### PaymentIntent.create のエンドポイント作成

では関数の作成に取り掛かります。
まずは node の [stripe](https://www.npmjs.com/package/stripe) モジュールをインストールし、

```shell
npm install stripe --save
```

Secret Key を使って以下のようなエンドポイントを作成します。

```ts
import * as functions from "firebase-functions";

const stripe = require("stripe")(process.env.STRIPE_SECRET_KEY);

/// PaymentIntent の作成
export const createPaymentIntent = functions.https.onCall(async (_, __) => {
  try {
    // 新しい Customer を作成（既存の場合は id を渡せばOK）
    const customer = await stripe.customers.create();
    // Ephemeral Key (一時的なアクセス権を付与するキー)を作成
    // https://stripe.com/docs/payments/accept-a-payment?platform=ios&ui=payment-sheet#add-server-endpoint
    const ephemeralKey = await stripe.ephemeralKeys.create(
      { customer: customer.id },
      { apiVersion: `2020-08-27` }
    );
    // PaymentIntent の作成
    const paymentIntent = await stripe.paymentIntents.create({
      amount: 1000,
      currency: `jpy`,
      customer: customer.id,
      automatic_payment_methods: {
        enabled: true,
      },
    });
    // アプリ側で必要な値を返却
    return {
      paymentIntent: paymentIntent.client_secret,
      ephemeralKey: ephemeralKey.secret,
      customer: customer.id,
    };
  } catch (error) {
    console.error(`error: %j`, error);
    return {
      title: `エラーが発生しました`,
      message: error,
    };
  }
});
```
上記に登場する Stripe リソースは以下のような役割があります。
- [Customer](https://stripe.com/docs/api/customers)
  - Stripe 上の顧客、お金を支払う人（ユーザー）
  - カード情報などの決済手段を持つことができる
- [EphemeralKey](https://stripe.com/docs/payments/accept-a-payment?platform=ios&ui=payment-sheet#add-server-endpoint)
  - Customer（機密情報）へのアクセスを一時的に許可する Key
- [PaymentIntent](https://stripe.com/docs/api/payment_intents/object)
  - Stripe 上の決済データ
  - 決済金額や通貨などを指定できる

今回の例では決済ごとに Customer オブジェクトを作成していますが、実際のサービスであればクライアント側から受け取った CustomerId などを使って PaymentIntent を作成することになるかと思います。

## 4. クライアント側の実装（Flutter）
最後に Flutter（クライアント）側の実装をしていきます。
main 関数内で Stripe パッケージの publishableKey と、Cloud Functions を呼び出せるように Firebase の初期化を行います。（Firebase の初期化は [FlutterFire CLI](https://firebase.google.com/docs/flutter/setup?platform=ios) を使えば楽です）

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Firebase の初期化
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);

  // ダッシュボードの公開可能キー
  Stripe.publishableKey = const String.fromEnvironment('STRIPE_PK_DEV');

  runApp(const App());
}
```
### PaymentSheet の表示

準備が整ったので、PaymentSheet を表示します。

initPaymentSheet メソッドに必要な情報を渡すと、決済ボトムシートが表示されるようになります。

全体のコードは以下になります。

```dart
class PaymentSheetPage extends HookWidget {
  const PaymentSheetPage({super.key});

  @override
  Widget build(BuildContext context) {
    final payment = useState<PaymentIntent?>(null);
    return Scaffold(
      appBar: AppBar(title: Text(runtimeType.toString())),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('金額：${payment.value?.amount.toString()}'),
            Text('ステータス：${payment.value?.status.name.toString()}'),
            Text('日時：${payment.value?.created.toString()}'),
            ElevatedButton(
              onPressed: () async {
                try {
                  // 1. Cloud Functions 上で PaymentIntent を作成
                  final callable = FirebaseFunctions.instance
                      .httpsCallable('createPaymentIntent');
                  final result = await callable.call();
                  final data = result.data;

                  // 2. PaymentSheet を初期化
                  await Stripe.instance.initPaymentSheet(
                    paymentSheetParameters: SetupPaymentSheetParameters(
                      customFlow: true,
                      merchantDisplayName: 'Flutter Stripe Example',
                      paymentIntentClientSecret: data['paymentIntent'],
                      customerEphemeralKeySecret: data['ephemeralKey'],
                      customerId: data['customer'],
                    ),
                  );

                  // 3. PaymentSheet を表示
                  await Stripe.instance.presentPaymentSheet();

                  // 4. 決済を確定
                  await Stripe.instance.confirmPaymentSheetPayment();

                  // 5. 決済内容を取得
                  final paymentIntent = await Stripe.instance
                      .retrievePaymentIntent(data['paymentIntent']);
                  payment.value = paymentIntent;
                } on StripeException catch (e) {
                  final error = e.error;
                  switch (error.code) {
                    case FailureCode.Canceled:
                      log('キャンセルされました', error: e);
                      break;
                    case FailureCode.Failed:
                      log('エラーが発生しました', error: e);
                      break;
                  }
                } on FirebaseFunctionsException catch (e) {
                  log('エラーが発生しました', error: e);
                } catch (e) {
                  log('不明なエラーが発生しました', error: e);
                }
              },
              child: const Text('Show PaymentSheet'),
            ),
          ],
        ),
      ),
    );
  }
}

```

コメントにある通りといえばそれまでなのですが、3 と 4 が `Future<void>` な点は、個人的には注意すべき点かなと思います。（どうしてこういった設計になっているのかわかる方いたら教えて下さい）

実際に動かしてみると、Stripe 上で決済が成功しているのを確認できました。

![](https://storage.googleapis.com/zenn-user-upload/6f2060cbc89e-20221201.png)

## まとめ

今回は flutter_stripe パッケージを触ってみました。

本当は [payment_intent_succeeded](https://stripe.com/docs/api/events/types#event_types-payment_intent.succeeded) をリッスンする WebHook のエンドポイントを作って、決済リスト表示みたいなことも解説できればと思ったのですが、時間が足りずミニマムな例になってしまいました。また時間のあるときに触ってみようと思います。

セキュアなカード決済機能を手軽に実装できるのは有り難いですね。
コミッターへの寄付も出来るみたいなので、お世話になった際は感謝を伝えにいこうと思います。

https://opencollective.com/flutter_stripe

最後までご覧いただき、ありがとうございました。

https://github.com/DaigoWakabayashi/flutter_stripe_example


## 参考

https://stripe.com/docs

https://docs.page/flutter-stripe/flutter_stripe/sheet



