---
title: "【Flutter】flutter_stripe パッケージ × Cloud Functions で楽々クレジット決済"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Flutter','Stripe',]
published: false
---

## はじめに

はじめまして、ダイゴです。

https://docs.page/flutter-stripe/flutter_stripe


## 環境

```shell
[✓] Flutter (Channel stable, 3.3.5, on macOS 13.0)
    • Flutter version 3.3.5 on channel stable
    • Dart version 2.18.2
```

## 1. パッケージのインストール・両OSでのセットアップ
### pubspec.yaml

プロジェクトの pubspec.yaml に、 [flutter_stripe](https://pub.dev/packages/flutter_stripe) を追加して、`pub get` します。

```diff yaml
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.2
+ flutter_stripe: ^6.0.0
```

### iOSの設定

Xcode を開いて
- Runner > Project Runner > Deployment Target > iOS Deployment Target
- Runner > Targets Runner > Minimum Deployments > Minimum Deployments > iOS 

を 12.0 以上に指定してください。

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

flutter_stripe パッケージはこの Key さえあれば使える状態になります。

:::message
- Stripe の新規アカウント発行時は「テスト環境」となっており、テストのクレジットカードを使って擬似的な決済を走らせることができる状態です。
- 実際にお金を動かすには[本番環境への利用申請](https://dashboard.stripe.com/account/onboarding/business-structure)が必要ですが本記事では省略しています。
:::

## カード決済

flutter_stripe には、3つのカード決済手段があります。

決済手段        |  説明  | 
------------- | --------------------------------------- | 
[**Payment sheet**](https://docs.page/flutter-stripe/flutter_stripe/sheet) | Stripeが推奨する決済手段。ローカライズ、アニメーション、エラー処理などが実装されている。  | 
[**Cardfield**](https://docs.page/flutter-stripe/flutter_stripe/card_field) | カードの入力フィールド（一行）。カスタム性高く実装が可能。 | 
[**Card form**](https://docs.page/flutter-stripe/flutter_stripe/card_field)  | CardFieldと機能は同等だが、複数行での入力が可能。 | 

## カード決済機能の作成


## 参考

