---
title: "【Flutter】BottomNavigationBar 永続化の最小サンプル作ってみた"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter"]
publication_name: "flutteruniv_dev"
published: true
---

## はじめに

はじめまして、[ダイゴ](https://twitter.com/Mamushi_journey)と申します。

Flutter の BottomNavigationBar は、各画面上に新しい画面をスタックさせると、自身が消えてしまう設計になっています。（特に工夫せず[公式 Doc のサンプルコード](https://api.flutter.dev/flutter/material/BottomNavigationBar-class.html)のように実装した場合）

[![FlutterでBottomNavigatorBar を残したまま各画面を遷移させる](https://storage.googleapis.com/zenn-user-upload/69b9b8792897-20220715.gif)](https://qiita.com/0maru/items/8e81ed5f0ddd5bae9658)

[まるさん](https://twitter.com/0maru_dev) の記事「[**Flutter で BottomNavigatorBar を残したまま各画面を遷移させる**](https://qiita.com/0maru/items/8e81ed5f0ddd5bae9658)」を参考に実装を進めていた（とても参考になりました 🙌）のですが、2019 年の記事だったので「今の Flutter ならもう少し短く書けるんじゃないか」と思い、2022 年版の個人的な最小サンプルを作ってみました。

ひとつの参考にして頂けると幸いです。

## サンプル

サンプルプロジェクトはこちらです。

- 環境
  - Flutter 3.0.0
  - Dart 2.17
- 依存パッケージ
  - [flutter_hooks](https://pub.dev/packages/flutter_hooks)
  - [pedantic_mono](https://pub.dev/packages/pedantic_mono)

https://github.com/DaigoWakabayashi/nested_bottom_navigation_bar

こんな感じで動きます。

![](https://storage.googleapis.com/zenn-user-upload/002d9658bae4-20220715.gif =250x)

### 1. 各タブの要素を Enum で定義

まずは各タブの要素を Enum で定義していきます。

今回は

- ホーム
- タイムライン
- 設定

という 3 つのタブを用意しています。

```cpp
import 'package:flutter/material.dart';
import 'package:nested_bottom_navigation_bar/pages/home_page.dart';
import 'package:nested_bottom_navigation_bar/pages/settings_page.dart';
import 'package:nested_bottom_navigation_bar/pages/timeline_page.dart';

enum TabItem {
  home(
    title: 'ホーム',
    icon: Icons.home,
    page: HomePage(),
  ),

  timeline(
    title: 'タイムライン',
    icon: Icons.timeline,
    page: TimelinePage(),
  ),

  settings(
    title: '設定',
    icon: Icons.settings,
    page: SettingsPage(),
  );

  const TabItem({
    required this.title,
    required this.icon,
    required this.page,
  });

  /// タイトル
  final String title;

  /// アイコン
  final IconData icon;

  /// 画面
  final Widget page;
}
```

Dart 2.17 から Enum の各値に定数を持たせることが出来るようになったので（[Enhanced Enum](https://medium.com/dartlang/dart-2-17-b216bfc80c5d)）、今回はそれを使い、各タブに

- title（String）
- icon（IconData）
- page（Widget）

を持たせています。

## 2.各ページの作成

Enum に対応する各ページ部分です。

- ページ名
- 詳細ページへの遷移ボタン

が表示されているシンプルなページです。

```cpp
import 'package:flutter/material.dart';
import 'package:nested_bottom_navigation_bar/enums/tab_item.dart';
import 'package:nested_bottom_navigation_bar/pages/detail_page.dart';

class SettingsPage extends StatelessWidget {
  const SettingsPage({super.key});

  @override
  Widget build(BuildContext context) {
    final pageTitle = TabItem.settings.title;
    return Scaffold(
      appBar: AppBar(title: Text(pageTitle)),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(pageTitle),
            ElevatedButton(
              onPressed: () {
                Navigator.push<void>(
                  context,
                  MaterialPageRoute<void>(
                    builder: (BuildContext context) => const DetailPage(),
                  ),
                );
              },
              child: const Text('詳細ページへ'),
            ),
          ],
        ),
      ),
    );
  }
}
```

ちなみに SettingsPage の super コンストラクタの記法も Dart 2.17 から導入された新しい記法です。

```diff cpp
class MyWidget extends StatelessWidget {
- const MyWidget({Key? key}) : super(key: key); // これまで
+ const MyWidget({super.key}); // 2.17 以後
```

## 3. BottomNav を持つ土台ページ

土台となるこのページで BottomNavigationBar を定義しています。

```cpp
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:nested_bottom_navigation_bar/enums/tab_item.dart';

final _navigatorKeys = <TabItem, GlobalKey<NavigatorState>>{
  TabItem.home: GlobalKey<NavigatorState>(),
  TabItem.timeline: GlobalKey<NavigatorState>(),
  TabItem.settings: GlobalKey<NavigatorState>(),
};

class BasePage extends HookWidget {
  const BasePage({super.key});

  @override
  Widget build(BuildContext context) {
    // ① useState で選択状態の管理
    final currentTab = useState(TabItem.home);
    return Scaffold(
      body: Stack(
        children: TabItem.values
            .map(
              (tabItem) => Offstage(
                offstage: currentTab.value != tabItem,
                // ② 各ページの Navigator に NavigatorState を持った Key を渡す
                child: Navigator(
                  key: _navigatorKeys[tabItem],
                  onGenerateRoute: (settings) {
                    return MaterialPageRoute<Widget>(
                      builder: (context) => tabItem.page,
                    );
                  },
                ),
              ),
            )
            .toList(),
      ),
      bottomNavigationBar: BottomNavigationBar(
        type: BottomNavigationBarType.fixed,
        currentIndex: TabItem.values.indexOf(currentTab.value),
        items: TabItem.values
            .map(
              (tabItem) => BottomNavigationBarItem(
                icon: Icon(tabItem.icon),
                label: tabItem.title,
              ),
            )
            .toList(),
        onTap: (index) {
          // ③ 選択済なら第一階層まで pop / 未選択なら currentTab に指定
          final selectedTab = TabItem.values[index];
          if (currentTab.value == selectedTab) {
            _navigatorKeys[selectedTab]
                ?.currentState
                ?.popUntil((route) => route.isFirst);
          } else {
            currentTab.value = selectedTab;
          }
        },
      ),
    );
  }
}
```

コメントをつけている 3 つのポイントについて解説します。

**① useState で選択状態の管理**

タブの選択状態を flutter_hooks の useState で管理してみました。

StatefulWidget でも動作的には問題ありませんが、何かと冗長になりがちなので、より簡潔に書くためにこちらを採用しました。

BasePage 全体で 60 行程度に収まっているので、無駄な情報が少なく可読性が高いコードに出来たかなと思います。

**② 各ページの Navigator に NavigatorState を持った Key を渡す**

一意な遷移状態（GlobalKey<NavigatorState>）を Navigator に渡して、ページ毎の遷移を実現しています。

BasePage の body で定義している Navigator 上で画面がスタックされていくため、BottomNav の永続化が実現できます。

![](https://storage.googleapis.com/zenn-user-upload/63c1c8c04b76-20220715.png)

**③ 選択済なら第一階層まで pop / 未選択なら currentTab に指定**

タップされたタブが選択済みか否かによって分岐する処理を行っています。

NavigatorState クラスを使うと遷移に関する様々な処理を行うことができるので、「もし選択済みのタブだった場合は第一階層（↑ の図でいう FirstPage）まで pop する」といった処理を行っています。

## まとめ

新しい記法を使って色々するのは楽しいですね。

ちなみにこのサンプルでは、Android で戻るボタンが押された場合のハンドリングは行っていません。

そちらの実装も[まるさんの記事](https://qiita.com/0maru/items/8e81ed5f0ddd5bae9658#android%E3%81%AE%E6%88%BB%E3%82%8B%E3%83%9C%E3%82%BF%E3%83%B3%E3%82%92%E5%88%B6%E5%BE%A1%E3%81%99%E3%82%8B)が参考になるので、興味がある方は参考にして実装してみてください。

最後までご覧いただき、ありがとうございました。

https://github.com/DaigoWakabayashi/nested_bottom_navigation_bar

## 参考

https://api.flutter.dev/flutter/material/BottomNavigationBar-class.html

https://qiita.com/0maru/items/8e81ed5f0ddd5bae9658
