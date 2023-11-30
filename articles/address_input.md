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

こんにちは、[ダイゴ](https://twitter.com/mamushi_journey) です。

とあるアプリを触っていた時に気持ちよくない住所入力を体験したので、Flutter でどうすれば気持ちよくできるかを考え、下記ようなサンプルを実装してみました。

![](https://storage.googleapis.com/zenn-user-upload/7972e8870005-20231201.gif)

後半では住所の有効性チェックについて記述しようとしたものの意図せず番外編となってしまいましたが、どなたかの参考になれば幸いです。

### 環境

- [✓] Flutter (Channel stable, 3.16.0)
- [✓] 対象プラットフォームはモバイル（iOS・Android）

https://github.com/DaigoWakabayashi/flutter_address_input

## 1. 気持ちよい住所入力

本記事では

- 郵便番号からの自動入力
- フォーカスの管理
- 入力可能な値の絞り込み

の 3 つの工夫について、サンプルコードを紹介しながら解説します。

### 郵便番号からの自動入力

郵便番号からの自動入力とは、入力された 7 桁の郵便番号を元に API を叩き住所を自動で埋める、といった実装です。

複数の外部 API サービスがあるようでしたが、一番ポピュラーかつ無料らしい [zipcloud の郵便番号検索 API](http://zipcloud.ibsnet.co.jp/doc/api) を使用してみました。

https://qiita.com/megane_/items/c8e4062697eaa785a103

該当するコードはこちら。[Flutter 大学の逆引き辞典](https://zenn.dev/pressedkonbu/books/flutter-reverse-lookup-dictionary/viewer/022-postal_code_to_address)に少し手を加えたような実装です。

```dart: 検索関数
Future<Address?> _searchAddress(String zipcode) async {
  final response = await http.get(
    Uri.parse('https://zipcloud.ibsnet.co.jp/api/search?zipcode=$zipcode'),
  );
  // 正常なレスポンスのみ処理
  if (response.statusCode != 200) {
    return null;
  }
  // パースして結果の配列を取得
  final body = jsonDecode(response.body) as Map<String, dynamic>;
  final results = body['results'] as List?;
  if (results == null || results.isEmpty) {
    return null;
  }
  // 複数の住所のうち、先頭の住所を使う
  final json = results.first as Map<String, dynamic>;
  final address = Address.fromJson(json);
  return address;
}
```

```dart:呼び出し側
class AddPage extends HookWidget {
  const AddPage({super.key});

  @override
  Widget build(BuildContext context) {

    final zipcodeController = useTextEditingController();
    final address1State = useState<Prefecture?>(null);
    final address2Controller = useTextEditingController();
    final address3Controller = useTextEditingController();
    final address4Controller = useTextEditingController();

    return Scaffold(
      body: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 16),
        child: Column(
          children: [
            const Gap(16),
            TextFormField(
              autofocus: true,
              controller: zipcodeController,
              decoration: const InputDecoration(labelText: '郵便番号'),
              onChanged: (zipcode) async {
                // 7 桁になったら住所検索
                if (zipcode.length != 7) return;
                final result = await _searchAddress(zipcode);
                // ヒットすれば address3 まで入力
                if (result != null) {
                  address1State.value =
                      Prefecture.values.byCode(result.prefcode);
                  address2Controller.text = result.address2;
                  address3Controller.text = result.address3;
                } else {
                  // しなければ SnackBar 表示（ダイアログ等でフローを止めないようにする）
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(content: Text('該当する住所が見つかりませんでした')),
                  );
                }
              },
            ),
            const Gap(8),

```

:::details 住所クラス

```dart
/// 住所にあたるモデルクラス
/// プロパティ名は API のレスポンスに合わせている
class Address {
  /// 郵便番号
  /// 7桁の数字、ハイフンなし
  final String zipcode;

  /// 都道府県コード
  final String prefcode;

  /// 都道府県名
  final String address1;

  /// 市区町村名
  final String address2;

  /// 町域名
  final String address3;

  /// 建物名
  final String? address4;

  Address({
    required this.zipcode,
    required this.prefcode,
    required this.address1,
    required this.address2,
    required this.address3,
    required this.address4,
  });

  factory Address.fromJson(Map<String, dynamic> json) {
    return Address(
      zipcode: json['zipcode'],
      prefcode: json['prefcode'],
      address1: json['address1'],
      address2: json['address2'],
      address3: json['address3'],
      address4: json['address4'],
    );
  }
}

```

:::

### フォーカスの管理

フォーカスとは、特定の入力フィールドに対してキーボードが出て、入力可能な状態のことです。

これが何も設定されていないアプリでは、住所 ① 入力 → 次のフィールドタップ → 住所 ② 入力 → 次のフィールドタップ... といった具合に、入力とフィールドタップを繰り返さなければいけません。ちょっとだけ気持ちよくないです。

単純に上から下へ（正確には [FocusTraversal](https://api.flutter.dev/flutter/widgets/FocusTraversalGroup-class.html) 順）とフォーカスを当てていく分には [`FocusScope.of(context).nextFocus()`](https://api.flutter.dev/flutter/widgets/FocusNode/nextFocus.html) メソッドで良いのですが、今回は自動入力
の対応もあるので、入力フィールドごとに FocusNode を作成して、フォーカスのリクエストを送ります。

今回のサンプルでは、

1. キーボードの完了ボタンで上から下に自動でフォーカスが当たっていく 2.自動入力
   があった時は、適切なフィールドまでフォーカスを飛ばす
2. もしテキストフィールド外をタップした場合はフォーカスを外す（onTapOutSide）

を実装しています。

```dart
class AddPage extends HookWidget {
  const AddPage({super.key});

  @override
  Widget build(BuildContext context) {
    // 入力値
    final zipcodeController = useTextEditingController();
    final address1State = useState<Prefecture?>(null);
    final address2Controller = useTextEditingController();
    final address3Controller = useTextEditingController();
    final address4Controller = useTextEditingController();
    // フォーカス
    final address2FocusNode = useFocusNode();
    final address3FocusNode = useFocusNode();
    final address4FocusNode = useFocusNode();
    // zipcodeFocusNode → autoFocus:true で対応できるため不要
    // address1FocusNode → 自動入力が成功した場合は address3 へ・失敗した場合は zipCode でのフォーカスを留めるため不要

    return Scaffold(
      body: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 16),
        child: Column(
          children: [
            const Gap(16),
            TextFormField(
              autofocus: true,
              controller: zipcodeController,
              decoration: const InputDecoration(labelText: '郵便番号'),
              keyboardType: TextInputType.number,
              inputFormatters: [
                LengthLimitingTextInputFormatter(7),
                FilteringTextInputFormatter.digitsOnly,
              ],
              onChanged: (zipcode) async {
                // 7 桁になったら住所検索
                if (zipcode.length != 7) return;
                final result = await _searchAddress(zipcode);
                // ヒットすれば address3 まで入力
                if (result != null) {
                  address1State.value =
                      Prefecture.values.byCode(result.prefcode);
                  address2Controller.text = result.address2;
                  address3Controller.text = result.address3;
                  address3FocusNode.requestFocus();
                } else {
                  // しなければ SnackBar 表示（ダイアログ等でフローを止めないようにする）
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(content: Text('該当する住所が見つかりませんでした')),
                  );
                }
              },
            ),
            const Gap(8),
            DropdownButtonFormField<Prefecture>(
              value: address1State.value,
              decoration: const InputDecoration(labelText: '都道府県'),
              items: Prefecture.values
                  .map((e) => DropdownMenuItem(value: e, child: Text(e.ja)))
                  .toList(),
              onChanged: (value) {
                address1State.value = value;
                address2FocusNode.requestFocus();
              },
            ),
            const Gap(8),
            TextFormField(
              controller: address2Controller,
              focusNode: address2FocusNode,
              decoration: const InputDecoration(labelText: '市区町村'),
              onEditingComplete: () => address3FocusNode.requestFocus(),
            ),
            const Gap(8),
            TextFormField(
              controller: address3Controller,
              focusNode: address3FocusNode,
              decoration: const InputDecoration(labelText: '番地'),
              onEditingComplete: () => address4FocusNode.requestFocus(),
            ),
            const Gap(8),
            TextFormField(
              controller: address4Controller,
              focusNode: address4FocusNode,
              decoration: const InputDecoration(labelText: '建物名（任意）'),
              onEditingComplete: () => FocusScope.of(context).unfocus(),
            ),
            const Gap(16),
            Center(
              child: ElevatedButton(
                onPressed: () async {追加処理}
                child: const Text('追加'),
              ),
            ),
```

flutter_hooks の [useFocusNode](https://pub.dev/documentation/flutter_hooks/latest/flutter_hooks/useFocusNode.html) を使用しています。dispose を自動でやってくれて便利です。

### 入力可能な値の絞り込み

最後に少し細かいところですが、入力可能な値を絞り込むと、より気持ちよくなれました。

- 郵便番号：数値 7 桁
- 都道府県：47 個の選択肢から一択

なので、前者はキーボードや formatter で絞り込み、後者は enum のドロップダウンで選択するようにしました。

```dart
            // 郵便番号
            TextFormField(
              autofocus: true,
              controller: zipcodeController,
              decoration: const InputDecoration(labelText: '郵便番号'),
              keyboardType: TextInputType.number, // 数値キーボードの指定
              inputFormatters: [
                LengthLimitingTextInputFormatter(7), // 桁数制限
                FilteringTextInputFormatter.digitsOnly, // ペースト対応バリデート
              ],
            ),
            const Gap(8),
            // 都道府県
            DropdownButtonFormField<Prefecture>(
              value: address1State.value,
              decoration: const InputDecoration(labelText: '都道府県'),
              items: Prefecture.values
                  .map((e) => DropdownMenuItem(value: e, child: Text(e.ja)))
                  .toList(),
              onChanged: (value) {
                address1State.value = value;
                address2FocusNode.requestFocus();
              },
            ),
```

:::details 都道府県 enum

```dart:
enum Prefecture {
  hokkaido(1, '北海道'),
  aomori(2, '青森県'),
  iwate(3, '岩手県'),
  miyagi(4, '宮城県'),
  akita(5, '秋田県'),
  yamagata(6, '山形県'),
  fukushima(7, '福島県'),
  ibaraki(8, '茨城県'),
  tochigi(9, '栃木県'),
  gunma(10, '群馬県'),
  saitama(11, '埼玉県'),
  chiba(12, '千葉県'),
  tokyo(13, '東京都'),
  kanagawa(14, '神奈川県'),
  niigata(15, '新潟県'),
  toyama(16, '富山県'),
  ishikawa(17, '石川県'),
  fukui(18, '福井県'),
  yamanashi(19, '山梨県'),
  nagano(20, '長野県'),
  gifu(21, '岐阜県'),
  shizuoka(22, '静岡県'),
  aichi(23, '愛知県'),
  mie(24, '三重県'),
  shiga(25, '滋賀県'),
  kyoto(26, '京都府'),
  osaka(27, '大阪府'),
  hyogo(28, '兵庫県'),
  nara(29, '奈良県'),
  wakayama(30, '和歌山県'),
  tottori(31, '鳥取県'),
  shimane(32, '島根県'),
  okayama(33, '岡山県'),
  hiroshima(34, '広島県'),
  yamaguchi(35, '山口県'),
  tokushima(36, '徳島県'),
  kagawa(37, '香川県'),
  ehime(38, '愛媛県'),
  kochi(39, '高知県'),
  fukuoka(40, '福岡県'),
  saga(41, '佐賀県'),
  nagasaki(42, '長崎県'),
  kumamoto(43, '熊本県'),
  oita(44, '大分県'),
  miyazaki(45, '宮崎県'),
  kagoshima(46, '鹿児島県'),
  okinawa(47, '沖縄県'),
  ;

  const Prefecture(this.code, this.ja);

  /// JIS X 0401 に定められた2桁の都道府県コード。
  /// ref: https://www.soumu.go.jp/denshijiti/code.html
  ///
  /// 01：北海道
  /// 02：青森県　03：岩手県　04：宮城県　05：秋田県　06：山形県　07：福島県
  /// 08：茨城県　09：栃木県　10：群馬県　11：埼玉県　12：千葉県　13：東京都　14：神奈川県
  /// 15：新潟県　16：富山県　17：石川県　18：福井県　19：山梨県　20：長野県
  /// 21：岐阜県　22：静岡県　23：愛知県　24：三重県
  /// 25：滋賀県　26：京都府　27：大阪府　28：兵庫県　29：奈良県　30：和歌山県
  /// 31：鳥取県　32：島根県　33：岡山県　34：広島県　35：山口県
  /// 36：徳島県　37：香川県　38：愛媛県　39：高知県
  /// 40：福岡県　41：佐賀県　42：長崎県　43：熊本県　44：大分県　45：宮崎県　46：鹿児島県　47：沖縄県
  ///
  final int code;

  /// 日本語の都道府県名
  /// 「都」「道」「府」「県」の文字列を含む
  final String ja;
}

extension PrefEx on List<Prefecture> {
  Prefecture byCode(String code) =>
      Prefecture.values.firstWhere((e) => e.code == int.parse(code));
}

```

:::

複雑な処理などは実装していないので、あまり説明なくコードベースになってしまいましたが、どなたかの参考になれば幸いです。

もし他にも工夫があれば、ぜひコメントいただきたいです。

## 2. FirebaseExtension で住所検証（日本非対応・番外編）

さて、前半では入力について紹介しましたが、後半は住所の住所の検証 Extension の紹介です。

本当はこちらを主題に持ってこようかと思っていたのですが、実装の途中で **日本では非対応であることに気づいた** （かなしい）ので、グローバルプロジェクトを開発している方へ、そしていつか日本に対応した時に少しでも参考になるようにと、念の為記録を残しておきます。

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

自分もこの記事を書く中で初めて知ったのですが、ぜひ興味のある方は、奥の深い住所の森を覗いてみてはいかがでしょうか。

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
