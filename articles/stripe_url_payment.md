---
title: "【Stripe】最近話題のURL決済を試してみた"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['stripe',]
published: true
---


# アカウントの作成


StripePaymentLinksの[トップページ](https://stripe.com/jp/payments/payment-links)にアクセスする

![](https://storage.googleapis.com/zenn-user-upload/3a6b5b60c2ccc3a234f5a07a.png)

「今すぐ始める」から必要事項を記入してアカウント作成

![](https://storage.googleapis.com/zenn-user-upload/9704d9e252f4e81c6ba4f3b4.png)

アカウントが作成されました

![](https://storage.googleapis.com/zenn-user-upload/f3a5ff942e9c9993c95ff3e2.png)


# 本番環境の利用申請

本番環境の利用には申請が必要らしいのでサイドバーの「本番環境の申請」からやってみる


![](https://storage.googleapis.com/zenn-user-upload/39e2a3f0146cfdfe55c51e76.png)


まずは住所を入力

![](https://storage.googleapis.com/zenn-user-upload/a0d6834874dc4fec5a098403.png)

次に申請者の情報

![](https://storage.googleapis.com/zenn-user-upload/936615a007a014931f9d2502.png)

次に事業の詳細（ウェブサイトを持っていない場合はSNSのアカウントページなどでも大丈夫みたいですが、詳細情報を後から求められる場合もあるみたいなので、できればウェブサイトを記載したほうがよさそうです）

![](https://storage.googleapis.com/zenn-user-upload/f90287414bfb2b86b8d6575e.png)

次に改正割販法（なんだそれは）に関連する質問に答えます
それなに？という方は[こちら](https://stripe.com/jp/guides/installment-sales-act)を参考にされると良いと思います。
簡単に言うと『クレジットカード番号等の「適切な管理」や「不正利用対策の義務化」など消費者保護を強化しよう』というものみたいです。





![](https://storage.googleapis.com/zenn-user-upload/6e91a7892c1c4c5001d83a72.png)

サポート情報を入力します（事業名をカタカナやらローマ字やらで入力するだけです）



![](https://storage.googleapis.com/zenn-user-upload/7c6c6cb8fbadb13cd87d2549.png)



口座情報を入力します

![](https://storage.googleapis.com/zenn-user-upload/138876154804bfac5f2c9b1a.png)


最後に確認

![](https://storage.googleapis.com/zenn-user-upload/07acfb91556054eba0bda6e9.png)

簡単なステップで確認ができました

![](https://storage.googleapis.com/zenn-user-upload/ee2ed63268843e139c59b63a.png)


# 支払いリンクの作成

サイドバーの「商品」→「支払いのリンク」から「支払いリンクを作成」


![](https://storage.googleapis.com/zenn-user-upload/75f048d7074e22fa49081de1.png)


作成画面に遷移しました
右側のプレビュー画面で決済ページの画面を確認できるようです

![](https://storage.googleapis.com/zenn-user-upload/784c306c04ded759b01679d9.png)

商品の

- 名前
- サムネイル
- 説明
- 金額（単発/月額）

を入力します

Stripeの[最小請求額](https://stripe.com/docs/currencies#%E6%9C%80%E5%B0%8F%E3%81%8A%E3%82%88%E3%81%B3%E6%9C%80%E5%A4%A7%E8%AB%8B%E6%B1%82%E9%87%91%E9%A1%8D)が50JPYだったので、51円に設定しました


右上の「リンクを作成」ボタンを押すと
![](https://storage.googleapis.com/zenn-user-upload/bf641a41969abcfa8f159d8d.png)


支払いのリンクが作成されました


![](https://storage.googleapis.com/zenn-user-upload/848d8752bb101ece32092ff9.png)

試しに決済してみると

![](https://storage.googleapis.com/zenn-user-upload/40f95007a20b9a2a9a59a5fc.png)


出来ました！（すごい）

![](https://storage.googleapis.com/zenn-user-upload/55d9ec4bb3d014a160f26efd.png)


Stripeのコンソールからも支払いが成功しているのが確認出来ます

![](https://storage.googleapis.com/zenn-user-upload/58c2e3ba9c93bd6d6d4cdee4.png)

## 備考


「記事の中にURL入れたら試しで決済してくれるんじゃね！？」という考えがよぎりましたが、いつもお世話になっているZennの開発者の方々へのリスペクトを込めてリンクはあえて載せませんでした。もし参考になったという方は「サポートする」からお願いします。
Zennの使いやすさが発信へのハードルをいつも下げてくれています。ありがとうございます。
まだまだひよっこエンジニアですが、Zenn同様に素敵なサービスを作れるように精進します！
