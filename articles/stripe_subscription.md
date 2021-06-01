---
title: "【Stripe API×Cloud Functions】プラットフォーム型サービスにサブスクリプションを導入する"
emoji: "💰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Stripe','Firebase','CloudFunctions','Nodejs']
published: false
---

# はじめに

2021年の3月にリリースした
現役エンジニアとプログラミング学習者のマッチングプラットフォーム、『[CodeBoy](https://codeboy.jp/)』ですが、本日より「月額プラン」に対応しました！🙌（以前までは「単発プラン」のみでした）

本記事では、CodeBoyのような「プラットフォーム型サービス」で、サブスクリプションを実装するにあたって詰まったこと、気をつけたことなどを共有できればと思います。


# CodeBoyの技術構成

#### フロントエンド

- Flutter Web 

#### バックエンド

- Firebase
  - Authentication
  - Firestore
  - Storage
  - Hosting
  - Functions
  - Extensions(Trigger Email)

#### 決済システム

- Stripe



それぞれについて詳しく知りたい、という方は[KBOYさんの記事](https://qiita.com/kboy/items/eb34dc0cd150f4890bad)をみていただけると全体像が把握できると思います。

# サービスの全体像

![](https://storage.googleapis.com/zenn-user-upload/bb01d06badf3a1221be65746.png)



### Customer
顧客。購入者でありお金を支払うユーザー。上の図で緑。
CodeBoyでは、「生徒」にあたります。


### Platform
その名の通りプラットフォームを提供するサービス。上の図の青。
CodeBoyでは、「CodeBoyというサービスそのもの」にあたります。


## Connected Account
プラットフォームを利用してサービスを提供し、入金を受ける方。上の図のピンク。
CodeBoyでは、「講師」にあたります。


# サブスクリプション関連のStripeリソース

| リソース | 概要 | 
| ---- | ---- | 
| **Product** | 複数の料金を保存できる場所。CodeBoyでは「ConnectAccount（講師）：Product（商品）」が「1:1」の関係になるように設定している。 | 
| **Price** | 価格。Products に対して複数設定可能で、請求金額と請求間隔が設定できる。CodeBoyでは「月額プラン」にあたる。 | 
| **Subscription** | 月額の自動決済。状態を持ち、支払いに関連して状態遷移する。CodeBoyでは「月額プランを購入」したタイミングで生まれる。 | 

他にもInvoiceやPaymentIntentsなどの多くのリソースがありますが、今回は「サブスクリプションが成立するまでの流れ」について取り上げるので、以上3つを扱います。

# サブスクリプションが成立するまでの流れ

1. Productの作成
2. Product内にPriceを格納
3. 顧客情報とPriceをもとに、Subscriptionを作成

ひとつずつ解説していきます。

# 1. Productの作成

まずはProduct（複数のPriceを格納できる箱）を用意します。
必須パラメータは`name`だけです。

```ts

// MARK: - Productを作成する
export const createStripeProduct = functions.region("asia-northeast1").https.onCall((data, context) => {
    const stripeAccountId = data.stripeAccountId; 

    return stripe.products.create({
        name: stripeAccountId, // Productの名前（必須）
    }, {
        idempotencyKey: data.idempotencyKey, // べき等性の担保
    });
});

```

https://stripe.com/docs/api/products/create

CodeBoyにおいて「複数のPriceを格納するもの」は講師なので、nameには講師のConnectAccountのidを指定しました。（Price内にもConnectAccountIdは入っているので、実際のところなんでも良いと思いますが、Stripeの管理画面でソートしやすいものにしたほうが良いと思います）
return された Product オブジェクト内にある id は productId として講師データ内に持たせておきます。

また、createメソッドの第2引数として渡している`idempotencyKey`は、べき等性を担保するためのパラメータです。
べき等とは、ある操作を1回行っても複数回行っても結果が同じであることをいう概念です。つまり`idempotencyKey`をつければ、そのkeyがついたリクエストを複数行ったとしても１回だけの処理にしてくれるということです。

すべてのstripeのメソッドに対して、追加の引数として渡すとべき等にしてくれます。
特に決済などの重複してはいけない処理には必須かと思われます。

https://stripe.com/docs/api/idempotent_requests


# 2. Product内にPriceを作成

次に、先程作ったProduct内にPrice（価格）を格納していきます。


```ts
// MARK: - Priceを作成する
export const createStripePrice = functions.region("asia-northeast1").https.onCall((data, context) => {
    const amount = data.amount;
    const productionId = data.product;

    return stripe.prices.create({
        unit_amount: amount, // 料金
        currency: 'jpy', // 通貨
        recurring: { interval: 'month' }, // 請求間隔
        product: productionId, // 保存するProductのid
    }, {
        idempotencyKey: data.idempotencyKey,
    });
});
```

https://stripe.com/docs/api/prices/create

今回指定したのは以下の4つです、

- 料金
- 通貨
- 請求間隔
- 保存するProductId

請求間隔は「[`day`, `week`, `month`, `year`](https://stripe.com/docs/api/prices/create#create_price-recurring-interval)」から選択することができるので、`month`を指定しています。（ニーズがあればプラン作成時に選択できるようにするのも面白いかも知れません）



# 3. 顧客情報とPriceをもとに、Subscriptionを作成

ProductとPriceが準備できたので、最後にSubscriptionを作成します。
Subscription自体の解説に入る前に、StripeConnectにおける、3つの決済種別について簡単に触れておきます。

### 決済の種別

Stripeには[3種類の決済種別](https://stripe.com/docs/connect/charges#types)があり、どれを使うかによって、サービスの資金フローが大きく変わります。

- [Direct charge](https://stripe.com/docs/connect/direct-charges)
- [Destination charge](https://stripe.com/docs/connect/destination-charges)
- [Separate Charges and Transfers](https://stripe.com/docs/connect/charges-transfers)

https://qiita.com/y_toku/items/7bfa42793801dfc5415d#%E8%B3%87%E9%87%91%E3%83%95%E3%83%AD%E3%83%BC%E3%82%92%E6%B1%BA%E3%82%81%E3%82%8B%E6%B1%BA%E6%B8%88%E4%BD%9C%E6%88%90%E6%96%B9%E6%B3%95

↑の記事で、3種類それぞれについて非常にわかりやすく解説されているので、ここでの解説は省きますが、
CodeBoyのようにサービス側が資金フローに責任を持つ場合は[Destination charge](https://stripe.com/docs/connect/destination-charges)による決済が妥当なので、以下の例はDestination chargeにおけるサブスクリプションであることに注意して下さい。（chargeの種類によって少しコードの書き方も違います）

### サブスクリプションの作成

```ts
// MARK: - Subscriptionを作成する
export const createStripeSubscription = functions.region("asia-northeast1").https.onCall((data, context) => {
    const customerId = data.customerId;
    const priceId = data.priceId;
    const stripeAccountId = data.stripeAccountId;

    return stripe.subscriptions.create({
        customer: customerId, // 顧客（生徒）
        items: [
            {
                price: priceId, // 料金
            },
        ],
        application_fee_percent: 10, // プラットフォーム手数料
        transfer_data: {
            destination: stripeAccountId, // 料金を受け取るアカウント（講師）
        },
    }, {
        idempotencyKey: data.idempotencyKey,
    });
});
```

https://stripe.com/docs/api/subscriptions/create

#### customer

`customer`には顧客であるユーザーのidを指定します。サービスの内容にもよりますが、CodeBoyではサインアップと同時に`customer`をcreateし、そのidをFirestoreのユーザードキュメント内に保存しています。

#### items

`items`では、税率やメタデータなどの詳細設定が可能です。
CodeBoyの場合、特別な設定は特に必要ないので、priceの情報だけを渡します。

#### application_fee_percent

ここでプラットフォーム手数料をパーセンテージで指定します。
CodeBoyは10％なので10を指定。

#### transfer_data

料金の受取先アカウントをConnectAccountのidで指定します。


Stripe管理画面の「顧客」→「定期支払い」から、サブスクリプションが作成されていることが確認出来ます。


![](https://storage.googleapis.com/zenn-user-upload/cdddc928de78d22a75b51c68.png)


# まとめ

以上がサブスクリプション作成までの流れです。
作成以降の管理（支払いWebHookイベントに応じた取引データの作成・解約など）については、少しの間サブスクリプションを運用してみてから、別の記事にまとめようと思います。

当方エンジニア歴がまだ1年にも満たないひよっこエンジニアなので、もし間違った認識をしていたり、ベターな実装方法があれあばご指摘いただけると嬉しいです！🙏

最後までご覧いただき、ありがとうございました！

https://codeboy.jp/