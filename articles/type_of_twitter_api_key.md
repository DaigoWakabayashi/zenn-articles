---
title: "TwitterAPI Key の種類まとめ"
emoji: "🐣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["twitterAPI"]
published: true
---

TwitterAPI について調べていく中で、consumerKey やら AppKey やら CustomerKey やら「なにがなになんだ」ってなったので調べると、公式 Doc に書いてあったので残しておく

```
用語の説明
以下のガイドで、同じものを呼ぶさまざまな用語を確認できます。

クライアントの認証情報:

App Key === API Key === Consumer API Key === Consumer Key === Customer Key === oauth_consumer_key
App Key Secret === API Secret Key === Consumer Secret === Consumer Key === Customer Key === oauth_consumer_secret
Callback URL === oauth_callback

一時的な認証情報:

Request Token === oauth_token
Request Token Secret === oauth_token_secret
oauth_verifier

トークンの認証情報:

Access token === Token === resulting oauth_token
Access token secret === Token Secret === resulting oauth_token_secret

```

参考

https://developer.twitter.com/ja/docs/authentication/oauth-1-0a/obtaining-user-access-tokens
