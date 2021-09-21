---
title: "【Flutter】初の個人開発で『ネコ補給アプリ』をリリースした話"
emoji: "🐱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firebase", "TwitterAPI"]
published: true
---

こんにちは、[ダイゴ](https://twitter.com/Mamushi_journey)と申します。

先日、初めての個人開発アプリ **『にゃんバズ』** をリリースしました。
発案〜リリースを全て一人で行ったのは初めての経験で、iOS のリジェクト対応など苦労した点もありましたが、なんとかリリース出来ました。
本記事では、**にゃんバズ** を作った背景や、使用している技術などについてまとめていきます。

# なにを作ったか

にゃんバズは

**バズっている猫ツイートだけが流れてくる「ねこ補給アプリ」**

です。

![](https://storage.googleapis.com/zenn-user-upload/b25467225dd1478ff3330665.gif =400x)

Twitter ランドの、かわいいネコチャンたちを、思う存分補給することが出来ます。
Flutter で作ったので、iOS / Android どちらでもダウンロード出来ます。

▼ **iOS**

https://apps.apple.com/jp/app/%E3%81%AB%E3%82%83%E3%82%93%E3%83%90%E3%82%BA-%E3%81%AD%E3%81%93%E8%A3%9C%E7%B5%A6%E3%82%A2%E3%83%97%E3%83%AA/id1581512288

▼ **Android**

https://play.google.com/store/apps/details?id=io.github.daigowakabayashi.nyanbuzz

# なぜ作ったか

### ① 自分が猫不足だったから

私は昨年 10 月まで、大阪の実家（猫が 2 匹）で暮らしていたのですが、
エンジニア系 YouTuber である[KBOY さん](https://twitter.com/kboy_silvergym)に弟子入りするにあたって、大阪から札幌に移住しました。詳細は以下の記事にてまとめています。

https://note.com/mamushi_journey/n/n63d81adc9d48#tRZTV

札幌に来てからというもの、
画面越しに見ていた「あの KBOY さん」と毎日一緒に開発をしたり、シェアハウスで友だちも出来たりして、仕事もプライベートも刺激的で充実した毎日でした。

ただ、僕には**猫が不足していました**。

札幌に来て 1 年が経ちますが、野良猫は一度も見たことがありません。
猫カフェにも何度か行きましたが、やはり**慢性的な猫不足**は解消できませんでした。

実物の猫をコンスタントに補給するのは難しいと考え、
せめて、視覚的に猫を補給できるアプリを作ろうと思いました。

### ② Twitter のバズ猫は『可愛い＋面白い』から

単にねこ不足を解消するだけなら、他のねこ SNS アプリや、[もちまる日記](https://www.youtube.com/c/%E3%82%82%E3%81%A1%E3%81%BE%E3%82%8B%E6%97%A5%E8%A8%98)で良いのですが、
私は **『Twitter の猫』** が良かったのです。

その理由は、

- Twitter のバズ猫たちは **「かわいい」だけでなく「面白い」** から

です。

Twitter の猫たちは **「ねこ特有の面白さ」** を含んでいることが多い気がしたのです。

特に、 1 万いいねを超えるようなバズ猫ツイートは、絶妙に「可愛さ」と「面白さ」を両立させているものが多く、**『こういうツイート、見逃したくないなぁ』** と思い、にゃんバズのアプリ構想を思いつきました。

# どうやって作ったか

### 開発期間

**1 ヶ月半**ほどです。

平日 1.5 時間、土日 5 時間ほどが開発時間だったので
週 18 時間 × 6 = 108 時間ほどでリリースしました。

厳密には、土日で旅行に行った週などもあったので、100 時間弱、といった感じでしょうか。
Dart を書いてる時間より、Functions で TwitterAPI を叩く部分や、iOS のリジェクト対応をしている時間の方が多かった気がします。

### 全体像

![](https://storage.googleapis.com/zenn-user-upload/ac3dee02960c250cfaf053b6.png)

#### ① TwitterAPI の呼び出し

Cloud Functions の定期実行機能（Scheduler）を使って、15 分に 1 回、Twitter の SearchAPI を走らせます。

https://developer.twitter.com/en/docs/twitter-api/v1/tweets/search/api-reference/get-search-tweets

現在の TwitterAPI は、
`v2` と `v1.1` の 2 種類が提供されているのですが、`v2`は、まだ 「EarlyAccess」 とされており、昨年 8 月に公開されたばかりで情報も少ないので、今回は `v1.1` を使いました。

ちなみに、定期実行が「15 分ごと」である理由は、TwitterAPI の利用制限が、15 分単位で発生するためです。
実際は、「180 コール/ 15 分」まで無料なのですが、分単位で呼び出すと `Functions の料金もかかってくる可能性`があるのと、`そもそもにゃんバズには、1 分単位のリアルタイム性は求めていない`という点から、15 分に一回定期実行するように設定しています。

| Endpoint          | Requests / window per user |
| ----------------- | -------------------------- |
| GET search/tweets | 180                        |

https://developer.twitter.com/en/docs/twitter-api/v1/rate-limits

#### ② ツイートの取得

SearchAPI では、以下 3 つの条件がすべて当てはまるツイートを取得しています。

- **500 いいね以上**
- **画像付きツイート**
- **猫ワード（ねこ、猫、ネコ、ﾈｺ、にゃんこ、ニャンコ、ﾆｬﾝｺ）が含まれている**

**猫ワード**が必要な点に関しては、後述の`今後の課題`で挙げています。

#### ③ ツイート保存

② で取得したツイート群を、Firestore に保存していきます。

Firestore の追加メソッドとして、

- `DocumentReference.set();`
- `CollectionReference.add();`

の 2 種類があるのですが、
`add();`メソッドを使うとドキュメント ID が自動指定されてしまい、同じツイートが複数追加されてしまう可能性があります。

そこで、以下のように、「ドキュメント ID ＝ ツイート ID」とした`DocumentReference`に対して、`set({merge: true});`とすることで、
既に Firestore 上に存在するツイートを取得してしまった場合でも、`update()`処理のような振る舞いにすることができます。

```js
const batch = db.batch();
for (const tweet of searchData.statuses) {
  const tweetRef = db.collection("tweets").doc(tweet.id_str);
  batch.set(
    tweetRef,
    {
      id: tweet.id,
      id_str: tweet.id_str,
      name: tweet.user.name,
      screen_name: tweet.user.screen_name,
      profile_image_url_https: tweet.user.profile_image_url_https,
      created_at: tweet.created_at,
      favorite_count: tweet.favorite_count,
      retweet_count: tweet.retweet_count,
      text: tweet.full_text,
      images: tweet.extended_entities.media,
    },
    { merge: true }
  );
  console.log(`Tweet${searchData.statuses.indexOf(tweet)} : %j`, tweet);
}
await batch.commit();
```

https://firebase.google.com/docs/firestore/manage-data/add-data?hl=ja#add_a_document

#### ④ 表示

Firestore に保存されたツイート群を取得し、表示していきます。

画像の詳細表示には、[`photo_view`](https://pub.dev/packages/photo_view)というライブラリの、`PhotoViewGallery`クラスを使用しています。`builder()`メソッドを使うと、URL の配列を用意するだけで、いい感じにしてくれるのでオススメです。

![](https://storage.googleapis.com/zenn-user-upload/b38e54bef40b6b003dfb64a0.gif =400x)

```dart:遷移元にHeroウィジェットを使うとAnimationも付きます
PhotoViewGallery.builder(
  itemCount: images.length,
  scrollPhysics: const BouncingScrollPhysics(),
  builder: (BuildContext context, int index) {
    return PhotoViewGalleryPageOptions(
      imageProvider: NetworkImage(images[index]),
      heroAttributes: PhotoViewHeroAttributes(tag: images[index]),
    );
  },
),
```

# 現状の課題・今後追加したい機能

現状の大きな課題として、

**「猫ワード（ねこ、ネコ、にゃんこ等）」が含まれているツイートしか取得できていない**

という点が挙げられます。
例えば、このツイートは **「猫ワード」** が入っているのでにゃんバズでも表示されますが、

https://twitter.com/Kreuz_FF14/status/1434411646701555720

こんなツイートは、**「猫ワード」** が入っていないので、現在のにゃんバズでは取得できていません。

https://twitter.com/kaorinogita/status/1432578324438544391?s=21

せっかく可愛いのにもったいないですね。
逆に、**「猫ワード」** が入っている、 🔞 ツイートなども流れてくることがあります。
悪くはありませんが、別腹なので、ゆくゆくは FirebaseMLKit などを用いて、ねこが写っている画像しか取得しないようにしたいです。

他にも、今後追加したい機能として、

- **動画対応**
- **バズ通知機能**（1 万いいね以上でプッシュ通知、など）
- **Twitter ログイン**（にゃんバズ上で「いいね」等ができる）

などがあります。
もしユーザーが増えてきたら、

- **保護猫カフェや愛護団体などへの寄付機能**

なども実装して、少しでもこの世のネコチャン達を救えたりしたらいいな、なんて考えています。

# さいごに

今回の個人開発を通して、**モノづくりの楽しさ** を改めて感じることができました。

自分のアプリがストアに並んでいるの初めて見たときは興奮しましたし、
リリース後は、家族や友人からもポジティブな反応がもらえて嬉しかったです。

まだスタートラインに立ったばかりなので、
**最強のねこ補給アプリ**を目指してアプデしていきたいと思います。

最後までご覧頂き、ありがとうございました。

▼ アプリ

https://nyanbuzz.studio.site/

▼ お世話になっているコミュニティ

https://kboyflutteruniv.com/

https://community.camp-fire.jp/projects/view/280040
