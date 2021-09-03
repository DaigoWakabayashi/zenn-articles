---
title: "【Flutter】個人的「なんかあったらコレ」リスト"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "iOS", "XCode"]
published: true
---

Flutter で開発中、ビルド周りで謎のエラーがでたときにやるコマンドたちです。

```
flutter clean
flutter pub cache repair
pod repo update
rm ios/Podfile.lock
rm ios/Podfile
rm -Rf ios/Pods
rm -Rf ios/.symlinks
rm -Rf ios/Flutter/Flutter.framework
rm -Rf ios/Flutter/Flutter.podspec

```

需要と時間があれば、それぞれのコマンドの意味を追記しようと思います。

もし他にも「なんかあったらコレ」的なモノがあれば教えていただけると嬉しいです。
