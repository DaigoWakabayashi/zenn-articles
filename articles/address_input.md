---
title: "【Flutter】気持ちよい住所入力を目指して"
emoji: "💙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firebase", "Firestore", "FirebaseExtensions"]
published: false
---

この記事は、[Flutter 大学アドベントカレンダー 2023](https://qiita.com/advent-calendar/2023/flutteruniv) 1 日目の記事です。

## はじめに

ダイゴです。

とあるアプリを触っていた時に気持ちよくない住所入力を体験したので、Flutter でどうすれば気持ちよくできるかを考え、実装してみました。

どなたかの参考になれば幸いです。

### 環境

- [✓] Flutter (Channel stable, 3.16.0)
- [✓] 対象プラットフォームはモバイル（iOS・Android）

https://github.com/DaigoWakabayashi/flutter_address_input

### 目次

## 1. Flutter での住所入力

まず、気持ちよい住所入力とは

- 「とにかく速く」
- 「バリデーションは緩く」

### フォーカス管理

- onTapOutside

### バリデーション

### 郵便番号の autoFill

## 2. FirebaseExtension で住所検証（日本非対応・番外編）

本当はこちらを主題に持ってこようかと思っていたのですが、実装を進めた後に **日本では非対応であることに気づいた** （愚か）ので、グローバルプロジェクトを開発している方へ、そしていつか日本に対応した時に少しでも参考になるようにと、念の為記録を残しておきます。

:::message

- これから紹介する [Validate Address in Firestore](https://extensions.dev/extensions/googlemapsplatform/firestore-validate-address) は、日本の住所には対応していません。
- 対応国は [こちら](https://developers.google.com/maps/documentation/address-validation/coverage?hl=en)

:::

### 1. [Validate Address in Firestore](https://extensions.dev/extensions/googlemapsplatform/firestore-validate-address) とは

[Validate Address in Firestore](https://extensions.dev/extensions/googlemapsplatform/firestore-validate-address) とは、Firestore 上に保存された住所データの **有効性をチェック** してくれる FirebaseExtension です。

この拡張機能を使うことで、クライアント側でのバリデーションを最低限に抑えつつ、手軽に「不正な住所の混入」「タイプミスなどの誤入力」を防ぐことができ、例えば下記のようなベネフィットを得ることができます。

- 配送関連：配送の失敗や誤配送、注文のキャンセルや再配達のようなコストのかかる問題を減らす。
- 金融関連：新規口座開設者の認証に住所証明を利用できる。口座開設時に顧客の住所が存在することを確認することで、不正登録を検出できる。

![](https://storage.googleapis.com/zenn-user-upload/0b9845275b49-20231130.png)

かといって自作で完全な有効性検証を実装するのは、特に日本だと難易度が高いです。無理に厳しいバリデーションを加えてしまうと、例外的な住所に対応できなくなります。（この辺りは X(Twitter) などでも[度々話題](https://togetter.com/li/2161880)になっている）

- [日本の住所の正規化に本気で取り組んでみたら大変すぎて鼻血が出た。- Qiita](https://qiita.com/miya0001/items/598070abcdf0799daebc)
- [とにかく日本の住所のヤバさをもっと知るべきだと思います - note](https://note.com/inuro/n/n7ec7cf15cf9c)
- [うわっ…日本の住所表記、ヤバすぎ…？解決策をダラダラ語る会 - YouTube](https://www.youtube.com/watch?v=XF1wvbWF0Q8)

この Extension が日本に対応できていないのも、上記のような特殊な日本の住所事情が絡んでいるからかも知れませんね。
では、実際の導入方法を紹介します。

### 2. Extension のインストール

はじめに Extension のページにある ↓ のボタンから

![](https://storage.googleapis.com/zenn-user-upload/f36a00c3b6d6-20231130.png)

対象のプロジェクトを選び、拡張機能をインストールします。

![](https://storage.googleapis.com/zenn-user-upload/0bd86464e991-20231130.png)

諸々の必須機能を有効化したのち、最後に

1. Addresses Collection → なんでも良いですが今回は addresses に指定
2. Google Address Validation API Key → GCP の[Credentials](https://console.cloud.google.com/apis/credentials) から作成
3. Cloud Functions location → 近い方が良さそうなので asia-northeast1(Tokyo) リージョンに指定

![](https://storage.googleapis.com/zenn-user-upload/ba07003d4889-20231130.png)

を入力してから、数分するとインストールが完了します。

### 3. 使ってみる

**有効性が高い場合**

**有効性が低い場合**

**未対応国の場合**

## まとめ

注意点

- 住所データは基本持ちたくないので、要件が合うようなら Stripe の shipping とか使いましょう
- もし Firestore に保存するなら、「認証認可」「スキーマ検証」「バリデーション」を意識したセキュリティルールを書きましょう

FlutterGakkai

Flutter 大学
