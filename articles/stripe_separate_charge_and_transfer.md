---
title: "【StripeAPI】決済額を複数の送金先へ山分けする"
emoji: "💰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Stripe", "Firebase", "CloudFunctions", "Nodejs", "TypeScript"]
published: true
---

こんにちは、[ダイゴ](https://twitter.com/Mamushi_journey)と申します。

先日、StripeAPI を使った山分けの決済（ 顧客：送金先 = １：n ）を実装する機会があったので、
インプットした情報を忘れないうちに整理しておきたい、と思い記事にしました。

どなたかの参考になれば幸いです。

# 想定するサービス

プラットフォーム型サービスで、以下のような決済要件を持つものとします。

- 決済総額のうち、10%をプラットフォーム手数料として徴収
- 残り 90%を複数の送金先へ均等に山分けする

# 全体のイメージ

![](https://storage.googleapis.com/zenn-user-upload/a1f9d14974b20fd23e169fae.png)

### Customer

顧客。購入者でありお金を支払うユーザー。上の図で緑。

### Platform

その名の通りプラットフォームを提供するサービスそのもの。上の図の青。

### Connected Account

プラットフォームを利用してサービスを提供し、入金を受ける方。上の図のピンク。

# 実装の流れ

1. 顧客（Customer）の作成
2. カード情報の登録
3. 送金先アカウント（ConnectAccount）の作成
4. 決済（Charge）の成立
5. 送金先（Transfer）の作成

ひとつずつ解説していきます。

# 1. 顧客（Customer）の作成

まずは、お金を支払う側である**Customer**を作成します。

なお、今回は CloudFunctions(TypeScript)を用いたコード例を示していきます。

https://stripe.com/docs/api/customers/create

```ts
const stripe = require("stripe")("プラットフォームのSecret Key", {
  typescript: true,
});

export const createCustomer = functions
  .region("asia-northeast1")
  .https.onCall(async (data, context) => {
    return stripe.customers.create();
  });
```

create メソッドを呼び出すと、Stripe 上で[Customer](https://stripe.com/docs/api/customers/object)オブジェクトが生成されます。
オブジェクト内の`id(cus_〇〇〇〇)`を、サービス側のユーザーデータに保存することで、Stripe 上の Customer とサービス側のユーザーを紐付けることが出来ます。

# 2. カード情報の登録

次に、先程作成した Customer にカード情報をもたせます。
カード情報を安全な形で登録するために、[カード情報を Token 化](https://stripe.com/docs/api/tokens/create_card)したものを Customer に登録します。

```ts
export const createCardInfo = functions
  .region("asia-northeast1")
  .https.onCall((data, context) => {
    const customerId = data.customerId;
    const cardToken = data.cardToken;
    return stripe.customers.createSource(customerId, { source: cardToken });
  });
```

以下の記事で、カード情報をセキュアに管理するための実装方法が詳しく紹介されています。

https://qiita.com/y_toku/items/7e51ef7e69d7cbbfb3ca#%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E3%82%92%E5%88%A9%E7%94%A8%E3%81%97%E3%81%A6%E6%B1%BA%E6%B8%88%E3%82%92%E3%81%99%E3%82%8B-or-%E3%82%AB%E3%83%BC%E3%83%89%E6%83%85%E5%A0%B1%E3%82%92%E5%AE%89%E5%85%A8%E3%81%AA%E5%BD%A2%E3%81%A7%E4%BF%9D%E5%AD%98%E3%81%99%E3%82%8B%E6%B5%81%E3%82%8C

# 3. 送金先アカウント（ConnectAccount）の作成

アカウントには、

- [Standard](https://stripe.com/docs/connect/standard-accounts)
- [Express](https://stripe.com/docs/connect/express-accounts)
- [Custom](https://stripe.com/docs/connect/custom-accounts)

の 3 種類が存在します。

https://stripe.com/docs/connect/accounts

↑ の公式ドキュメントで、日本語で解説されているので解説は省略しますが、
簡単にいうと、**Standard** → **Express** → **Custom**の順で、カスタマイズ性が高くなり、よりユーザーに Stripe を意識させずに実装することが可能です。
今回は、一番カスタマイズ性の高い**Custom**アカウントでの実装例を示していきます。

```ts
/// ConnectAccountの作成
export const createConnectAccount = functions
  .region("asia-northeast1")
  .https.onCall((data, context) => {
    return stripe.accounts.create(
      {
        type: "custom",
        country: "JP",
        business_type: "individual",
        capabilities: {
          card_payments: { requested: true },
          transfers: { requested: true },
        },
      },
      {
        idempotencyKey: data.idempotencyKey,
      }
    );
  });
```

[`business_type`](https://stripe.com/docs/api/accounts/object#account_object-business_type)は、`individual`か`company`から選ぶことができ（`non_profit`、`government_entity` は US のみ）、どちらを選ぶかによって認証フローが少し変わってきます。今回は、個人であることを想定して`individual`を設定しました。

[`capabilities`](https://stripe.com/docs/api/accounts/object#account_object-capabilities)は、アカウントに持たせる機能を指定するパラメータです。今回は、`card_payments`（カード決済）と`transfers`（送金）を指定しました。

上記のコードで、送金先となる [Account](https://stripe.com/docs/api/accounts/object) オブジェクトが作成される訳ですが、実はこれだけだと送金先として指定ことができません。

送金を受けるには、[`accounts.update`](https://stripe.com/docs/api/accounts/update)メソッドを使って

1. 本人確認情報の提出
2. 利用規約への同意

を行い、[`charges_enabled`](https://stripe.com/docs/api/accounts/object#account_object-charges_enabled)と[`payouts_enabled`](https://stripe.com/docs/api/accounts/object#account_object-payouts_enabled)を true にする必要があります。
実はここが一番実装が大変だったりもするのですが、長くなりそうなので、また別の記事にまとめようと思います。

https://stripe.com/docs/connect/identity-verification

# 4. 決済（Charge）の作成

カード情報を登録した Customer と、本人確認を終えた Account が用意できたら、いよいよ本題である決済（Charge）を作成していきます。
Stripe には[3 種類の決済種別](https://stripe.com/docs/connect/charges#types)があり、どれを使うかによって、サービスの資金フローが大きく変わります。

## ① Direct charge

**「顧客と子アカウント間で直接決済を行い、決済額の一部をプラットフォームアカウントへ送金する」** という決済方式です。
この場合、子アカウントが決済に対しての責任を持つことになるため、決済手数料の負担や返金対応は子アカウント自身が行います。

https://stripe.com/docs/connect/direct-charges

## ② Destination charge

**「プラットフォームに支払いを作成すると同時に、送金先アカウント（単一）へいくら配分するかを決める」** という決済方式です。
この場合、プラットフォーム側が決済に対しての責任を持つため、決済手数料の負担や返金対応はプラットフォームが行います。

https://stripe.com/docs/connect/destination-charges

## ③ Separate Charges and Transfers

その名の通り、**「決済と送金を分けて行う」** 決済方式です。
プラットフォームに対して決済が行われ、プラットフォームの任意のタイミングで、任意のアカウント（複数可）に送金する、という資金フローを実現できます。
この場合も Destination charge と同様に、プラットフォーム側が決済に対しての責任を持つため、決済手数料の負担や返金対応はプラットフォームが行います。

https://stripe.com/docs/connect/charges-transfers

今回の決済要件は、

- 支払い総額の 10%をプラットフォーム手数料として徴収
- 残った 90%を**複数**の子アカウントで均等に山分け

という要件であるため、[Separate Charges and Transfers](https://stripe.com/docs/connect/charges-transfers)を採用します。
**「決済と送金を分けて行う」** ので、まずはプラットフォームに対しての決済を作成します。

```ts
/// 決済を作成
export const createCharge = functions
  .region("asia-northeast1")
  .https.onCall(async (data, context) => {
    const customer = data.customerId; // 顧客
    const amount = data.amount; // 支払い総額
    return stripe.charges.create(
      {
        customer: customer,
        amount: amount,
        currency: "jpy",
      },
      {
        idempotencyKey: data.idempotencyKey,
      }
    );
  });
```

第 2 引数として渡している`idempotencyKey`は、べき等性を担保するためのパラメータです。
べき等とは、ある操作を 1 回行っても複数回行っても結果が同じであることをいう概念です。つまり`idempotencyKey`をつければ、その key がついたリクエストを複数行ったとしても１回だけの処理にしてくれるということです。

すべての stripe のメソッドに対して、追加の引数として渡すとべき等にしてるので、特に決済などの重複してはいけない処理には必須となるパラメータです。

https://qiita.com/kboy/items/6d8ce83084a0f49ab0d2

https://stripe.com/docs/api/idempotent_requests

# 5. 送金（Transfer）の作成

最後に、上記で作成した決済データを使って、送金を作成します。
支払い総額の 10%を、手数料としてプラットフォームに残し、残りの 90%を、子アカウントで山分けします。

```ts
/// 送金を作成
export const createTransfer = functions
  .region("asia-northeast1")
  .https.onCall(async (data, context) => {
    const chargeId = data.charge.id; // 送金元の決済ID
    const amount = data.charge.amount; // 送金元の決済総額
    const feeAmount = Math.floor((amount * 1) / 10); // プラットフォーム手数料
    const transferAmount = amount - feeAmount; // 山分けする金額（支払い総額 - プラットフォーム手数料）
    const targetAccountIds = data.targetAccountIds; // 子アカウントIDの配列
    const singleAmount = Math.floor(transferAmount / targetAccountIds.length); // 子アカウントあたりの送金金額
    for (const accountId of targetAccountIds) {
      await stripe.transfers.create(
        {
          amount: singleAmount,
          currency: "jpy",
          destination: accountId,
          source_transaction: chargeId, // ココがポイント
        },
        {
          idempotencyKey: data.idempotencyKey,
        }
      );
    }
  });
```

ポイントは、[`source_transaction`](https://stripe.com/docs/api/transfers/create#create_transfer-source_transaction)です。

本来、決済（Charge）と送金（Transfer）と紐付けるには、

- 決済・送金の create 時に、同一の[`transfer_group`](https://stripe.com/docs/api/transfers/object#transfer_object-transfer_group)を指定

しなければいけないのですが、
この場合、プラットフォーム側に「利用可能な残高」が必要です。日本の場合、決済が行われてから「利用可能な残高」となるまで 4 営業日かかります。

そのため、プラットフォーム側に残高がない（サービスローンチ直後など）状態で送金を作成しようとすると、**「プラットフォームに利用可能な残高がない」** という旨のエラーが多発します。

このエラーを防ぐのが[`source_transaction`](https://stripe.com/docs/api/transfers/create#create_transfer-source_transaction)です。
[`source_transaction`](https://stripe.com/docs/api/transfers/create#create_transfer-source_transaction)で、送金元の決済 ID を指定することで、プラットフォームに利用可能な残高がなくても、該当の決済が利用可能になった時、自動的に送金がされるようになります。

以下の記事でも、詳しく解説されていますので、参考にしてみると良いかも知れません。

https://qiita.com/y_toku/items/7bfa42793801dfc5415d#source_transaction-%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%A6%E6%B1%BA%E6%B8%88%E3%81%A8%E9%80%81%E9%87%91%E3%82%92%E7%B4%90%E4%BB%98%E3%81%91%E3%82%A8%E3%83%A9%E3%83%BC%E3%82%92%E6%B8%9B%E3%82%89%E3%81%99

# まとめ

今回は、決済額を複数の送金先へ山分けする実装について紹介しました。

山分けのロジック自体はそこまで複雑ではないのですが、
**子アカウントの type 選定**や、**3 つの決済種別とその選定** あたりが、情報が多く、少し複雑なところかと思うので、本記事がどなたかの参考になれば幸いです。

最後まで読んで頂き、ありがとうございました！

# 参考

- Stripe 公式ドキュメント

https://stripe.com/docs/api

- Stripe Connect 101

https://qiita.com/y_toku/items/7bfa42793801dfc5415d#stripe-connect-%E3%81%A8%E3%81%AF
