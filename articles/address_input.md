---
title: "【Flutter】気持ちよい住所入力を目指して"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firebase", "Firestore", "FirebaseExtensions"]
published: false
---

この記事は、[Flutter 大学アドベントカレンダー 2023](https://qiita.com/advent-calendar/2023/flutteruniv) 1 日目の記事です。

## はじめに

ダイゴです。

どなたかの参考になれば幸いです。

### 環境・対象プラットフォーム

```shell
[✓] Flutter (Channel stable, 3.16.0)
[✓] Xcode - develop for iOS and macOS (Xcode 15.0)
```

### サンプルコード

### 目次

## 1. 実装方針

- 「とにかく速く」
- 「バリデーションは緩く」

### i. フォーカス管理とバリデーション

### ii. 郵便番号の autoFill

## 2. FirebaseExtension で住所バリデーション（日本非対応・番外編）

本当はこちらを主題に持ってこようかと思っていたのですが、実装を進めた後に **日本では非対応であることに気づいた** ので、
日本人が同じ轍を踏まないよう、そして、海外の方のためになるよう、そして、いつか日本に対応した時に少しでも参考になるようにと、念の為、記録を残しておきます。

:::message alert

- この拡張機能は、日本では対応していません。
- 対応国は[こちら](https://developers.google.com/maps/documentation/address-validation/coverage?hl=en)

:::

### 1. [Validate Address in Firestore](https://extensions.dev/extensions/googlemapsplatform/firestore-validate-address) とは

[Validate Address in Firestore](https://extensions.dev/extensions/googlemapsplatform/firestore-validate-address) とは、Firestore 上に保存された住所データの **有効性をチェック** してくれる FirebaseExtension です。

この拡張機能を使うことで、クライアント側でのバリデーションを最低限に抑えつつ、手軽に「不正な住所の混入」を防ぐことができます。

### 2. Extension のインストール

はじめに Extension のページにある ↓ のボタンから

画像

画像

対象のプロジェクトを選び、拡張機能をインストールします。
諸々の必須機能を有効化したのち、最後に

1. Addresses Collection → なんでも良いですが今回は addresses に指定
2. Google Address Validation API Key → GCP の[Credentials](https://console.cloud.google.com/apis/credentials) から作成
3. Cloud Functions location → 近い方が良さそうなので asia-northeast1(Tokyo) リージョンに指定

を入力してから、数分するとインストールが完了します。

### 3. 対応国の住所を入れる

**有効性が高い場合**

**有効性が低い場合**

**未対応国の場合**

## まとめ

注意点

- 住所データは基本持ちたくないので、要件が合うようなら Stripe の shipping とか使いましょう
- もし Firestore に保存するなら、「認証認可」「スキーマ検証」「バリデーション」を意識したセキュリティルールを書きましょう

FlutterGakkai

Flutter 大学
