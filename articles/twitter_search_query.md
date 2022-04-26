---
title: "【TwitterAPI v2】検索クエリ・オペレータ一覧"
emoji: "🐣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['TwitterAPI']
published: true
---

## オペレータ一覧

:::message
学術研究利用のアクセスレベルである [**Academic Research**](https://developer.twitter.com/en/docs/twitter-api/getting-started/about-twitter-api#v2-access-level) のオペレータは除いています。
:::

#### Standalone 

単独で使用することができるオペレータ。（他のオペレータと併用も可能）

| **Operator** | **意味** | **例** |
| --- | --- | --- |
| ``keyword`` |  特定のキーワードを含む | `ねこ OR ネコ` | 
| ``emoji`` |  特定の絵文字を含む | `(😺 OR 🙀) 🐾` |
| ``""`` | 文字列の完全一致 | `"Twitter API" OR -"recent search"` |
| ``#`` | 特定のハッシュタグを含む | `#ねこ OR #ねこ好きさんと繋がりたい`|
| ``@`` | 特定のユーザー名エンティティ`@`を含む | `v2 @twitter`
| ``from:`` | 特定ユーザーによるツイート | `from:twitterdev OR from:twitter` |
| ``to:`` | 特定ユーザーへのリプライツイート | `to:twitterdev OR to:twitter` |
| ``url:`` | 特定のURLを含む | `url:"https://developer.twitter.com"` |
| ``retweets_of:`` | 指定したユーザーのリツイート | `retweets_of:twitterdev` |
| ``context:`` | 特定のドメインID/属性IDのペアを持つツイート（[詳細](https://developer.twitter.com/en/docs/twitter-api/annotations/overview)） | `context:47.1139229372198469633` |
| ``entity:`` | 特定のエンティティ文字列の値を持つツイート（[詳細](https://developer.twitter.com/en/docs/twitter-api/annotations/overview)）| `entity:"Barcelona"` |
| ``conversation_id:`` | 特定スレッド（`conversation`）内のツイート | `conversation_id:1334987486343299072` |

#### Conjunction-required

単独でクエリに使用することはできない。（単独で使用すると大量のツイートにマッチしてしまうため）少なくとも1つのStandalone演算子がクエリに含まれている場合にのみ使用することができるオペレータ。

| **Operator** | **意味** | **例** |
| --- | --- | --- |
| `is:retweet` | リツイート（引用リツイートを除く） | `data @twitterdev -is:retweet` |
| `is:reply` | リプライ | `from:twitterdev is:reply` |
| `is:quote` | 引用リツイート | `"sentiment analysis" is:quote` |
| `is:verified` | Twitterで作者が確認された？ツイート | `#nowplaying is:verified` |
| `has:hashtags` | 少なくとも1つのハッシュタグを含むツイート | `from:twitterdev -has:hashtags` |
| `has:links` | ツイート本文にリンクやメディアを含むツイート | `from:twitterdev announcement has:links` |
| `has:mentions` | 他のユーザーへのメンションを含むツイート | `#nowplaying has:mentions` |
| `has:media` | 写真、GIF、動画などのメディアオブジェクトを含むツイート | `(kittens OR puppies) has:media` |
| `has:images` | 画像と認識されたURLを含むツイート | `#meme has:images` |
| `has:videos` | Twitterに直接アップロードされたビデオを含むツイート | `#icebucketchallenge has:videos` |
| `lang:` | Twitterが特定の言語として分類したツイート | `TOKYO lang:ja` |



## bool 演算子・グルーピング


| 演算子 | 説明 |
| --- | --- |
| `AND` | スペースを挟んで連続する演算子は、boolean 演算の「AND」ロジックとなり、両方の条件を満たしたツイートが返される。 |
| `OR` | ORを挟んで連続する演算子はOR論理となり、どちらかの条件を満たしたツイートが返される。 |
| `-`  | キーワード（または任意の演算子）の前にダッシュ（-）を付けると、そのキーワードを否定（NOT）することができる。すべての演算子は否定することができるが、否定された演算子を単独で使用することはできない。 |
| `()` | 括弧を使用して演算子をグループ化することができる。ANDが最初に適用され、次にORが適用される。 |


:::message
括弧の中にまとめられた集合を否定してはならない。
例： skiing -(snow OR day OR noschool) → skiing -snow -day -noschool 
:::


## 評価順序

ANDとORの機能を組み合わせる場合、

1. ANDロジックで接続された演算子を評価
2. ORロジックで接続された演算子を評価

の順序でクエリが評価される。

例えば、

`apple OR iphone ipad` は `apple OR (iphone ipad)` と評価され、
`ipad iphone OR android` は、`(iphone ipad) OR android` と評価される。



## 備考
#### クエリの長さ制限
- 使用する[アクセスレベル](https://developer.twitter.com/en/docs/twitter-api/getting-started/about-twitter-api#v2-access-level)によって、文字数に制限がある。
  - Essential or Elevated → 512文字まで
  - Academic  → 1024文字まで

#### 大文字・小文字の区別

すべての演算子は、大文字と小文字を区別せずに評価される。
例えば、catというクエリは、cat, CAT, Cat を含むツイートにマッチする。

#### 引用リツイートのクエリマッチング

SearchAPI を使用する場合、演算子は **引用元のツイートの内容にはマッチしない** が、**引用ツイート本体に含まれる内容** にはマッチする。


:::message
ただし、[filtered stream](https://developer.twitter.com/en/docs/twitter-api/tweets/filtered-stream) は、引用された元のツイートと引用ツイートの両方のコンテンツにマッチすることに注意が必要。
:::


## 参考

https://developer.twitter.com/en/docs/twitter-api/tweets/search/integrate/build-a-query