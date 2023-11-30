---
title: "【Flutter】気持ちよい住所入力を目指して"
emoji: "💙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firebase", "Firestore", "FirebaseExtensions"]
publication_name: "flutteruniv_dev"
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

:::message

- 住所データは可能であれば持ちたくないので、要件が合うようなら外部サービスへの保存とかも検討する（Stripe の customer.shipping など）
- もし Firestore に保存するなら「認証認可」「スキーマ検証」「バリデーション」を意識したセキュリティルールを書く

:::

## 2. FirebaseExtension で住所検証（日本非対応・番外編）

本当はこちらを主題に持ってこようかと思っていたのですが、実装の途中で **日本では非対応であることに気づいた** （愚か）ので、グローバルプロジェクトを開発している方へ、そしていつか日本に対応した時に少しでも参考になるようにと、念の為記録を残しておきます。

:::message

- これから紹介する [Validate Address in Firestore](https://extensions.dev/extensions/googlemapsplatform/firestore-validate-address) は、日本の住所には対応していません。
- 対応国は [こちら](https://developers.google.com/maps/documentation/address-validation/coverage?hl=en)

:::

### Validate Address in Firestore とは

[Validate Address in Firestore](https://extensions.dev/extensions/googlemapsplatform/firestore-validate-address) とは、GoogleMapsPlatform の [Address Validation API](https://mapsplatform.google.com/maps-products/address-validation) を使って、Firestore 上に保存された **住所データの有効性を自動でチェック・補完** してくれる FirebaseExtension です。

![](https://storage.googleapis.com/zenn-user-upload/0b9845275b49-20231130.png)

この拡張機能を使うことで、クライアント側でのバリデーションを最低限に抑えつつ、手軽に「不正な住所の混入」「タイプミスなどの誤入力」を防ぐことができ、例えば下記のような恩恵を得ることができます。

- 配送関連：配送の失敗や誤配送、注文のキャンセルや再配達のようなコストのかかる問題を減らす。
- 金融関連：新規口座開設者の認証に住所証明を利用できる。口座開設時に顧客の住所が存在することを確認することで、不正登録を検出できる。

難易度の高い住所の有効性検証を手軽に実装できるのは良さそうですね。特に日本だと難易度が高いことは知られているようで、過去に X(Twitter) などでも[話題](https://togetter.com/li/2161880)になっています。

自分もこの記事を書く中で初めて見てみたのですが、ぜひ興味のある方は、奥の深い住所の森を覗いてみてはいかがでしょうか。

- [日本の住所の正規化に本気で取り組んでみたら大変すぎて鼻血が出た。- Qiita](https://qiita.com/miya0001/items/598070abcdf0799daebc)
- [とにかく日本の住所のヤバさをもっと知るべきだと思います - note](https://note.com/inuro/n/n7ec7cf15cf9c)
- [うわっ…日本の住所表記、ヤバすぎ…？解決策をダラダラ語る会 - YouTube](https://www.youtube.com/watch?v=XF1wvbWF0Q8)

30 カ国以上に対応しているこの API が日本に対応できていないのも、上記のような特殊な日本の住所事情が絡んでいるからかも知れませんね。では、海外プロダクト向けに実際の導入方法を紹介します。

### Extension のインストール

はじめに Extension のページにある ↓ のボタンから

![](https://storage.googleapis.com/zenn-user-upload/f36a00c3b6d6-20231130.png)

対象のプロジェクトを選び、拡張機能をインストールします。

![](https://storage.googleapis.com/zenn-user-upload/0bd86464e991-20231130.png)

諸々の必須機能を有効化したのち、最後に 3 項目を入力します。

1. Addresses Collection
2. Google Address Validation API Key
3. Cloud Functions location

2 の API Key は GCP の [Credentials](https://console.cloud.google.com/apis/credentials) ページ ↓ から作成し

![](https://storage.googleapis.com/zenn-user-upload/0909357d7b45-20231130.png)

値をコピーしたのちに、フィールドに入れて

![](https://storage.googleapis.com/zenn-user-upload/65decb54a24b-20231130.png)

拡張機能をインストールします。今回は下記になりました。

![](https://storage.googleapis.com/zenn-user-upload/0d83e002fe0c-20231130.png)

数分するとインストールが完了します。

### 使ってみる

インストールが完了したので、対応国であるアメリカの住所を入れてみて、どういった動きになるのか試してみます。

**有効性が高い場合**

まず、サンプルにある有効な住所を入れてみます。

![](https://storage.googleapis.com/zenn-user-upload/6b60be0f8254-20231130.png)

すると、下記のような addressValidity という Map データがすぐに生成されました。

![](https://storage.googleapis.com/zenn-user-upload/3f9446d47059-20231130.png)

内部の構造は [ValidationResult](https://developers.google.com/maps/documentation/address-validation/reference/rest/v1/TopLevel/validateAddress?hl=en#validationresult) 型になっており、入力された住所の各要素に対して検証が行われています。

```json:ValidationResult
{
  "result": {
    // 検証結果まとめ
    "verdict": {},
    // APIによって決定された住所の詳細
    "address": {},
    // 入力住所に対して生成されたジオコード
    "geocode": {},
    // 住所が事業所か住居かなどを示す情報
    "metadata": {},
    // 米国郵政公社からの住所に関する情報(アメリカ・プエルトリコのみ対応)
    "uspsData": {}
  },
  // リクエストごとに割り当てられる ID
  "responseId": "ID"
}
```

重要なのは verdict と address です。

[**Verdict**](https://developers.google.com/maps/documentation/address-validation/reference/rest/v1/TopLevel/validateAddress?hl=en#Verdict)

- 全体に対する検証結果の集まり
- [Granularity(enum)](https://developers.google.com/maps/documentation/address-validation/reference/rest/v1/TopLevel/validateAddress?hl=en#Granularity) もしくは bool で表現される（Granularity → 直訳は粒度）
- 使い道としては、このフィールドを元にプロダクト要件に応じた「有効」「無効」基準を設定するなど

```json:Verdict
{
    "inputGranularity": enum (Granularity),
    "validationGranularity": enum (Granularity),
    "geocodeGranularity": enum (Granularity),
    "addressComplete": boolean,
    "hasUnconfirmedComponents": boolean,
    "hasInferredComponents": boolean,
    "hasReplacedComponents": boolean
}
```

[**Address**](https://developers.google.com/maps/documentation/address-validation/reference/rest/v1/TopLevel/validateAddress?hl=en#Address)

- 自動整形された住所の詳細
- 住所のスペルミスのある部分の修正、誤った部分の置き換え、欠落している部分の推測などを含む
- 使い道としては、有効性が低い項目の修正サジェストなど

```json:Address
{
  "formattedAddress": string,
  "postalAddress": {
    object (PostalAddress)
  },
  "addressComponents": [
    {
      object (AddressComponent)
    }
  ],
  "missingComponentTypes": [
    string
  ],
  "unconfirmedComponentTypes": [
    string
  ],
  "unresolvedTokens": [
    string
  ]
}
```

Firestore を用いた際のフローとしては、

1. 住所入力し、完了ボタンを押す
2. address を追加処理
3. 追加した address ドキュメントを監視
4. 検証結果に応じて、有効性の高い住所へと整形（無効な住所であれば再入力、もしくは有効性が低いパラメータのみアラート表示など）
5. プロダクトに応じた [**Verdict**](https://developers.google.com/maps/documentation/address-validation/reference/rest/v1/TopLevel/validateAddress?hl=en#Verdict) の基準を満たしたら終了

のようなものにすると良さそうだなぁと思いました。

素の Address Validation API を用いた実装例については GoogleMapPlatform の Architecture Center の例も参考になりそうです ↓

https://developers.google.com/maps/architecture/ecommerce-checkout-address-validation?hl=en#implementing_address_validation_api

**未対応国の場合**

最後に、日本の住所を US 形式でいれてみると

![](https://storage.googleapis.com/zenn-user-upload/5709e5328f74-20231130.png)

400 Bad Request になりました（そりゃそう）

![](https://storage.googleapis.com/zenn-user-upload/cadafa02e2f1-20231130.png)

## まとめ

少し残念な後半になってしまいましたが「こんなのあるんだ〜」程度に思ってもらえたら嬉しいです。手軽に導入できて便利だなと思ったので、もし日本対応したらより詳しく記事にしようと思います。笑

また、来年初旬に FlutterGakkai というカンファレンスの 第 5 回を開催予定です。
connpass 登録者数 700 名以上・累計参加者数 900 名以上と、日本では FlutterKaigi に次ぐ規模のカンファレンスになりつつあるので、Flutter に興味のある方はぜひ X(Twitter) や connpass をチェックしてもらえると幸いです。

https://twitter.com/FlutterGakkai/status/1729421310869020955

最後まで読んでいただき、ありがとうございました。

https://flutteruniv.com/

## 参考

https://mapsplatform.google.com/maps-products/address-validation/

https://developers.google.com/maps/architecture/ecommerce-checkout-address-validation?hl=en
