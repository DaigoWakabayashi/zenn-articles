---
title: "【Flutter】Android版リリース作業で参考にした記事"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Android"]
published: false
---

初の個人開発アプリ（Flutter 製）を Android 版リリースするにあたって参考にさせていただいた記事を挙げています。

## Flutter 公式 Doc

- Build and release an Android app

一旦サクッと見てみるのはアリ（全体の流れだけ把握できる、詳細は日本の記事を調べた方が良さそう）

https://flutter.dev/docs/deployment/android

## ビルドまでにすること

- 証明書の作成

https://qiita.com/kasa_le/items/d23075d817f42e869778#1%E8%A8%BC%E6%98%8E%E6%9B%B8%E3%82%92%E4%BD%9C%E6%88%90

- ビルドコマンド

詰まったけど、flavor 指定したら行けた

```
flutter build appbundle --flavor production --dart-define=FLAVOR=production

```
