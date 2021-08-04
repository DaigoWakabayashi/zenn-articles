---
title: "【GoogleMobileAds】Flutterでバナー広告をリスト表示する"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Firebase", "AdMob"]
published: true
---

# 実装したもの

ListView 内で表示できるバナー広告を実装しました。

![](https://storage.googleapis.com/zenn-user-upload/fefac416e08d6e6b0ec73b7e.gif)

# Flutter の AdMob 事情

Flutter で用意されている admob パッケージは大きく 3 つあります。

- firebase_admob

Firebase が用意しているパッケージ。
広告を Widget 扱いすることができず、画面上部・下部に固定されてしまう。
2021 年 8 月 2 日現在では DISCONTINUED となっており、今後の積極的な開発は期待できない。

https://pub.dev/packages/firebase_admob

- admob_flutter

OSS で作られているパッケージ。
広告をひとつの Widget として扱うことができるので、ユーザーにストレスを与えずに広告を実装することができる。
コントリビューター不足で、あまり積極的に開発が行われていない。

https://pub.dev/packages/firebase_admob

- google_mobile_ads

2021 年 2 月に Google が出したパッケージ。
広告を Widget として扱うことができ、かつ開発も積極的に行われている。
FlutterDeveloper 待望のパッケージ。

https://pub.dev/packages/google_mobile_ads

今回は、3 つ目の google_mobile_ads を使った広告実装についてまとめていきます。

# 前提

- Flutter 1.22.0 以上
- アプリ未リリース

# 実装手順

0. AdMob の登録
1. パッケージの導入
2. iOS/Android での設定
3. 広告の実装

## 0. AdMob の登録

https://admob.google.com/home/

上記にアクセスし、必要事項を記入します。

![](https://storage.googleapis.com/zenn-user-upload/e54af83cd56f4cbe6e960f34.png)

AdMob のユーザー登録が完了しました。

![](https://storage.googleapis.com/zenn-user-upload/daffec8d7610993e136fd983.png)

#### iOS アプリの設定

サイドバーの「アプリ」→「アプリを登録して利用を開始」

![](https://storage.googleapis.com/zenn-user-upload/0120ad207a6724e59c2838c1.png)

まずは iOS、アプリ未リリースを選択

![](https://storage.googleapis.com/zenn-user-upload/1694f73563d283dce0c6fbe9.png)

アプリ名（bundleId とかじゃなくなんでもいい）とユーザー指標にチェックを入れて「アプリを追加」

![](https://storage.googleapis.com/zenn-user-upload/d33b888799123df6b1227254.png)

iOS アプリが追加されました。

![](https://storage.googleapis.com/zenn-user-upload/9b42dc650174150723866e56.png)

#### Android アプリの設定

「アプリを追加」から Android アプリの設定も同様に行います。

![](https://storage.googleapis.com/zenn-user-upload/b6fe049334dcc87a992104ba.png)

ここで Android を選択します。それ以外は同じです。

![](https://storage.googleapis.com/zenn-user-upload/254e2d049e99ebb8d0df8935.png)

iOS、Android 両方のアプリが作成できました。

![](https://storage.googleapis.com/zenn-user-upload/bd153f9ab857ad8c786d6f8d.png)

#### アプリ ID の確認

「アプリ」→「すべてのアプリを表示」で

![](https://storage.googleapis.com/zenn-user-upload/a24ebaa2cc933682db28f656.png)

それぞれのアプリ ID を確認することができます。

![](https://storage.googleapis.com/zenn-user-upload/1af18a039dcb4ef30910ffb9.png)

#### 広告ユニットの作成

次に、広告ユニットを作成します。
サイドバーの「アプリ」→「広告ユニット」

![](https://storage.googleapis.com/zenn-user-upload/39a1f508a598895546a95f56.png)

様々な広告フォーマットがありますが、今回は ListView 内に広告を挿入したいので、「バナー」を選択

![](https://storage.googleapis.com/zenn-user-upload/9782225d76037775621cf3e9.png)

広告ユニット名を記入します。

![](https://storage.googleapis.com/zenn-user-upload/2a406ec4c25fe186b2ccad25.png)

広告ユニットが作成され、ID が生成されました。

![](https://storage.googleapis.com/zenn-user-upload/aca78c168a2cf419b309a128.png)

アプリ ID と混同しそうですが、別物です。

```
アプリID・・・それぞれのアプリに一つだけ存在
広告ユニットID・・・作成した広告ごとに存在（複数生成できる）
```

iOS アプリの広告ユニットはできたので、Android アプリでも同様に広告ユニットを生成しておきましょう。
AdMob の登録は以上です。

## 1. パッケージ導入

Flutter で実装していきます。以下のパッケージを pubspec.yaml に追加し、pub get します。

https://pub.dev/packages/google_mobile_ads/install

```yml:pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  ・
  ・
  ・
  google_mobile_ads: ^0.13.2+1

```

## 2. iOS/Android での設定

iOS は、ios/Runner/Info.plist を以下のように更新します。

```plist:Info.plist

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    ~~~~~~~~~
    <key>GADApplicationIdentifier</key>
    <string>ca-app-pub-aaaaaaaaaaaaa~00000000000000</string>
    <key>SKAdNetworkItems</key>
      <array>
        <dict>
          <key>SKAdNetworkIdentifier</key>
          <string>cstr6suwn9.skadnetwork</string>
        </dict>
      </array>
    ~~~~~~~~~
</dict>
</plist>

```

Android は、android/app/src/main/AndroidManifest.xml 内を以下のように更新します。

```xml:AndroidManifest.xml
<manifest xmlns:android=""
    package="">
   <application
        android:label=""
        android:icon="">
       <!-- Sample AdMob App ID: ca-app-pub-3940256099942544~3347511713 -->
        <meta-data
           android:name="com.google.android.gms.ads.APPLICATION_ID"
           android:value="ca-app-pub-aaaaaaaaaaa~0000000000"/>
    </application>
</manifest>

```

## 3. 広告の実装

#### パッケージの初期化

まずは main 関数内で、初期化処理を行います。

```dart
void main() {

  WidgetsFlutterBinding.ensureInitialized();
  MobileAds.instance.initialize(); // 追加
  runApp(const MyApp());
}
```

#### 広告の初期化（バナー広告）

次に、広告ウィジェットの初期化です。
BannerAd クラスをインスタンス化した後、.load();メソッドを呼び出すことでバナー広告を読み込みます。

```dart
    // バナー広告をインスタンス化
    BannerAd　myBanner = BannerAd(
      adUnitId: getTestAdBannerUnitId(),
      size: AdSize.banner,
      request: const AdRequest(),
      listener: const BannerAdListener();
    );
    // バナー広告の読み込み
    myBanner.load();
```

`getTestAdBannerUnitId()`は、プラットフォーム（iOS/Android）に合わせてデモ用広告 ID を返します。

```dart
  // プラットフォーム（iOS / Android）に合わせてデモ用広告IDを返す
  String getTestAdBannerUnitId() {
    String testBannerUnitId = "";
    if (Platform.isAndroid) {
      // Android のとき
      testBannerUnitId = "ca-app-pub-3940256099942544/6300978111"; // Androidのデモ用バナー広告ID
    } else if (Platform.isIOS) {
      // iOSのとき
      testBannerUnitId = "ca-app-pub-3940256099942544/2934735716"; // iOSのデモ用バナー広告ID
    }
    return testBannerUnitId;
  }
```

テストモードを使わずに多くの広告をクリックすると、無効なアクティビティとしてアカウントが警告を受ける恐れがあります。
開発時には、必ずデモ用の広告ユニットを使うようにしましょう。

| 広告の種類            | iOS                                    | Android                                |
| --------------------- | -------------------------------------- | -------------------------------------- |
| バナー                | ca-app-pub-3940256099942544/2934735716 | ca-app-pub-3940256099942544/6300978111 |
| リワード              | ca-app-pub-3940256099942544/1712485313 | ca-app-pub-3940256099942544/5224354917 |
| ネイティブ アドバンス | ca-app-pub-3940256099942544/3986624511 | ca-app-pub-3940256099942544/2247696110 |

上記以外のデモ用広告 ID は以下で確認できます。

- iOS

https://developers.google.com/admob/ios/test-ads?hl=ja#sample_ad_units

- Android

https://developers.google.com/admob/android/test-ads?hl=ja#sample_ad_units

#### バナー広告 Widet を使って表示

`load（）`を呼び出した後、`AdWidget`を用いることで、広告を表示することができます。
`AdWidget`は StatefulWidget を継承しており、他の Widget たちと同様に UI に組み込むことが可能です。

```dart
AdWidget(ad: myBanner),
```

![](https://storage.googleapis.com/zenn-user-upload/81403d05573e6290799d96f7.png)

バナー広告の表示が出来ました。

#### ListView 内に広告を挿入

最後に、 ListView 内に広告を挿入する例を紹介しておきます。
ListView.builder の index が 5 の倍数のときに広告を挿入する
というコードにしてみました。

```dart:timeline_page.dart
    /// リスト表示部分
    child: ListView.builder(
      itemCount: tweets.length,
      itemBuilder: (context, index) {
        final tweet = tweets[index];
        return Column(
          children: [
            // indexが 5 の倍数の場合
            index % 5 == 0
                ? Container(
                    color: Colors.white,
                    height: 64.0,
                    width: double.infinity,
                    child: AdWidget(ad: myBanner),
                  )
                : const SizedBox(),
            // ツイート部分
            InkWell(
              onTap: () async {
                await Navigator.push<dynamic>(
                  context,
                  MaterialPageRoute<dynamic>(
                    builder: (context) =>
                        TweetDetailPage(tweet),
                  ),
                );
              },
              child: TweetCardWidget(
                tweet: tweet,
                tag: kTimeLineHeroTag,
              ),
            ),
          ],
        );
      },
    ),

```

バナー広告をリスト内に挿入することができました。
よくある「バナーが画面上部や下部に固定されているヤツ」に比べると、ユーザーへのストレスも多少は軽減されるのではないでしょうか。

![](https://storage.googleapis.com/zenn-user-upload/fefac416e08d6e6b0ec73b7e.gif)

これから AdMob の実装する方の参考になれば幸いです。
最後まで読んでいただき、ありがとうございました。

# 参考

- google_mobile_ads

https://pub.dev/packages/google_mobile_ads

- Google AdMob スタートガイド

https://developers.google.com/admob/flutter/quick-start

- Google Mobile Ads for Flutter の最新 0.13.0 使ってみた

https://zenn.dev/kentaroh/articles/729a52a3e90bca
