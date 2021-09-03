---
title: "【iOS】プロビジョニング周りの沼で得た知識をまとめておく"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["iOS", "XCode"]
published: false
---

# はじめに

iOS のプロビジョニング周沼を 2 日ほど泳いでいたので、忘れないうちに自分なりに噛み砕いて形に残しておこうと思います。
もし間違っている点などがあったら、ご指摘いただけると幸いです。

# 全体像

![](https://storage.googleapis.com/zenn-user-upload/68ef7940b85b4870775f9e2a.png)

https://qiita.com/fujisan3/items/d037e3c40a0acc46f618
この図がわかりやすかったです。

## CertSigningRequest

![](https://storage.googleapis.com/zenn-user-upload/d3a8ecb8247c12b8bc200764.png =100x)

署名証明書（SigningCertificate）の作成のリクエスト用ファイル。
「キーチェーン」>「認証アシスタント」>「認証局に証明書を要求」で取得する。

## Certificates

![](https://storage.googleapis.com/zenn-user-upload/c69a4548b7ae7c2c06fb0931.png =100x)

署名証明書（SigningCertificate）。
先ほど作成した CertSigningRequest↑ を使って AppleDeveloper 上で作成する。
開発用（Development）と製品用（Distribution）がある。

https://developer.apple.com/account/resources/certificates/list

#### 開発用証明書(Development)

開発者として登録して取得した証明書をキーチェーンアクセスに登録した PC でないとビルド出来ない。.cer ファイル。

#### 製品用証明書(Distribution)

製品用（AppStore、AdHoc 配布含む）
こっちは正直よくわかっていない。

## Devices

端末情報。
開発機として UDID を登録したデバイスでないとアプリをインストールできない。

## Identifiers

アプリの ID。
Bundle ID を設定したり、プッシュ通知が利用可能かなどをアプリ情報と紐付ける。

## Provisioning Profile

- Certificates
- Devices
- Identifiers

を合わせて作られたもの。.mobileprovision。
Provisioning Profile には Development と Distribution という種類が別れていて、Development は単純に開発用のもの。Distribution は配布用のもの。 Distribution = 配布用と言っても、App Store に配布するのか、Ad Hoc でテスト用として配布するのかが別れてきたりするので、二つ必要。

## p12 ファイル

Certificates から出力できるファイル。
Certificates は証明書のみ（証明書と、必要な秘密鍵が書かれているのみ。実際の鍵は入っていない。）で、p12 ファイルは秘密鍵を含めた証明書なので管理は厳重に。Admin 権限をもつことと等価。

# 証明書の自動生成

XCode から自動生成してくれる。

https://xyk.hatenablog.com/entry/2020/10/29/171732
