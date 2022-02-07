---
title: "【iOS】低速回線をシュミレートする"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Xcode','iOS',]
published: true
---

## 開発ツールのインストール

- Xcode > Developer Tools > More Developer Tools 

![](https://storage.googleapis.com/zenn-user-upload/c9efdf43bdc2-20220207.png)

- ブラウザが立ち上がるので、Additional Tools for Xcode と検索し、自分のXcodeのバージョンにあったものをダウンロードする

![](https://storage.googleapis.com/zenn-user-upload/93ec2dc8311b-20220207.png)


- ダウンロードした dmg ファイルを開き、Hardware > Network Link Conditioner.prefPane をインストールする

![](https://storage.googleapis.com/zenn-user-upload/905828efb8a8-20220207.png)

## 低速回線のテスト

- Profiles から Edge を選択しONにすると、iOS シュミレータ上で低速状態をテストできる

![](https://storage.googleapis.com/zenn-user-upload/e17dd3e9bc29-20220207.png)

