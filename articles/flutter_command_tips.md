---
title: "【Flutter】個人的「なんかあったらコレ」リスト"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "iOS", "XCode"]
published: true
---

Flutter で開発中、よくわからないエラー（特にビルド周り）がでたときにやるコマンドたちです。
普段は mac のメモ帳に入れているのですが、どなたかの助けに慣れば幸いです。

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

もし他にも「なんかあったらコレ」的なモノがあれば教えていただけると嬉しいです。
