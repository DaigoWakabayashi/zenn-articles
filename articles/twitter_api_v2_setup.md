---
title: "【2022年】TwitterAPI v2 の仕様まとめ・セットアップ方法"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Twitter',]
published: true
---

## はじめに

こんにちは、[ダイゴ](https://twitter.com/mamushi_journey)です。

2021年11月から、Twitter のデフォルトAPIが [v1.1](https://developer.twitter.com/en/docs/twitter-api/v1) から [v2](https://developer.twitter.com/en/docs/twitter-api) に変更されました。

v2 に関する日本語記事が少ない＆アクセスレベルや利用申請方法も変更されているため、本記事では、v2 の仕様まとめ・セットアップ方法について解説していきます。

どなたかの参考になれば幸いです。

https://developer.twitter.com/en/docs/twitter-api


## 目次

1. Twitter API v2 のアクセスレベル
2. Developer アカウント登録（Essential利用）
3. Elevated プランへのアップグレード方法


## 1. Twitter API v2 のアクセスレベル

v2 のアクセスレベルは以下の 3つです。
![](https://storage.googleapis.com/zenn-user-upload/fdcfefe12f08-20220402.png)


- Essential
  - サインアップするだけですぐ使える
  - 無料
  - [tweet caps](https://developer.twitter.com/en/docs/twitter-api/tweet-caps) は50万ツイート / 月
  - 作れる環境は 1つ
- Elevated
  - 申請が必要
  - 無料
  - 200万ツイート / 月
  - 環境は 3つ作成可能
  - v1.1 のAPIにもアクセス可能
- Academic Research
  - 学術的な使い方する場合に使う
  - 申請が必要
  - 無料

v1.1 との大きな違いは、利用申請が簡単になったことです。

以前までは、すべてのアクセスレベルで英語による利用申請が必要でしたが、v2 から Essential プランの場合は申請が不要になり、サインアップするだけですぐに使えるようになりました。

Essential プランでも、取得制限は 50万ツイート / 月 なので、「とりあえず触ってみたい」「簡単な Bot を作りたい」といったユースケースの場合はこちらで十分かと思います。

より詳しい各プランの違いは、[公式Doc](https://developer.twitter.com/en/docs/twitter-api/getting-started/about-twitter-api#v2-access-level) で詳細に記述されているので、そちらをご覧ください。


https://developer.twitter.com/en/docs/twitter-api/getting-started/about-twitter-api#v2-access-level

本記事では、開発者アカウントへの登録（Essential利用）と Elevated API へのアップグレード方法について解説していきます。

## 2. Developer アカウント登録（Essential利用）

Twitter アカウントを作成しておきます。

https://twitter.com/



[TwitterAPIの公式ページ](https://developer.twitter.com/en/docs/twitter-api/getting-started/getting-access-to-the-twitter-api)から、APIを使用したいアカウントにログインした状態で Sign Up します。


![](https://storage.googleapis.com/zenn-user-upload/72e99a18a1cc-20220401.png)


Developer Portal （開発者用の画面）に遷移します。
電話番号をTwitterアカウントに登録しろと言われる場合は、Twitterアカウントの設定画面から登録しておきます。

![](https://storage.googleapis.com/zenn-user-upload/a202e244225a-20220401.png)

いくつかの簡単な質問に回答し、利用規約に同意します。

![](https://storage.googleapis.com/zenn-user-upload/61dc98c11dd7-20220401.png)



認証メールから認証を行うと、App（APIを利用するアプリケーション）の作成画面が表示されます。
私の場合、TwitterAPIを使ったモバイルアプリを作成する予定だったので、そちらのアプリ名を入力しました。


![](https://storage.googleapis.com/zenn-user-upload/f3b6e4251b89-20220401.png)

すると、**ものすごく重要なKey達** が生成されるので、安全な場所にすべてコピーしておきましょう。

![](https://storage.googleapis.com/zenn-user-upload/5355652152e9-20220401.png)


このフローはスキップしても大丈夫ですが、試しに curl で実行してみて、レスポンスが帰ってきたらOKです。

```
% curl -X GET -H "Authorization: Bearer <コピーしたToken>" "https://api.twitter.com/2/tweets/20" 
{"data":{"id":"20","text":"just setting up my twttr"}}
```


![](https://storage.googleapis.com/zenn-user-upload/221d6d3c86fa-20220401.png)

これで、TwitterAPI v2 にアクセスできるようになりました。

ユースケースによってはこちらのプランで十分ではありますが、Essential では環境が一つしか作れないという制限があり、個人的に Dev Prod の環境分けを行いたかったので、次に Elevated プランの申請方法について解説します。


## 3. Elevated プランへのアップグレード方法

[Developer Portal](https://developer.twitter.com/en/portal/products) から、申請フローを開始します。

![](https://storage.googleapis.com/zenn-user-upload/966b93f0e231-20220401.png)

基本情報を入力し、

![](https://storage.googleapis.com/zenn-user-upload/54643d7aff59-20220401.png)


Twitter API の用途を入力していきます。（In English とあるので英語で記入した方が良さそう）

こちらは人によって違うと思うので、参考までに自分の申請文言を載せておきます。（DeepLにまかせる→自分で軽く添削、が早くてオススメです）

- **How will you use the Twitter API or Twitter Data?**

```
There are two things we want to make.

The first is an iOS/Android application that retrieves and lists tweets about cats.

The second is a bot that retrieves popular tweets about cats and replies to them.
```

- **Are you planning to analyze Twitter data?**

→ No

- **Will your App use Tweet, Retweet, Like, Follow, or Direct Message functionality?**

→ Yes

```
We will use the reply function.

For example, to a cat tweet with over 10,000 likes, "That's cute!" We plan to send a reply to the effect that

If it is difficult to use the reply function, we will consider using the quote retweet function.
```

- **Do you plan to display Tweets or aggregate data about Twitter content outside Twitter?**

→ Yes

```
We plan to create an iOS / Android app that lists cat-related tweets.

We plan to create a tweet list screen and a detail screen.
The detail screen will also include a function to jump to the Twitter mobile app.
```

- **Will your product, service, or analysis make Twitter content or derived information available to a government entity?**

→ No


内容を確認し、利用規約に同意すると、Elevated API へのアクセスが承認されました。

恐らくここからTwitter側で審査して、ダメだった場合はリジェクトの連絡が来るのだと思います。（自分の場合は何もありませんでした）


![](https://storage.googleapis.com/zenn-user-upload/4b663ff69a06-20220401.png)

せっかくアップグレード出来たので、Dev・Prod の環境それぞれのアプリを作成していきます。

Add App を選択し、

![](https://storage.googleapis.com/zenn-user-upload/f766dee37dd9-20220401.png)

作りたい環境を選択します。（初期アプリは Develop になっていました）

![](https://storage.googleapis.com/zenn-user-upload/3d52a2ded2d4-20220401.png)

アプリ名を設定します。Prod環境なので、[アプリ名] | Production としました。

![](https://storage.googleapis.com/zenn-user-upload/8f1a9f884c2a-20220401.png)

Production 環境用の **ものすごく大事なKey達** が生成されます。

![](https://storage.googleapis.com/zenn-user-upload/a09268b572b6-20220401.png)


環境分けが完了しました。

![](https://storage.googleapis.com/zenn-user-upload/e8319117b056-20220401.png)


## さいごに

今回の v1.1 → v2 の変更で、API の仕様が大幅に変わっており、v2 はまだ情報量も少ないので、これからも Twitter API 関連の情報発信をしていけたらと思います。

最後までご覧いただき、ありがとうございました。


