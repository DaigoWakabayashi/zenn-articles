---
title: "【<デバイス> is busy: Copying cache files from device】への対処法"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter","iOS"]
published: true
---

# エラー内容

```
Error: <デバイス名> is busy: Copying cache files from device. Xcode will continue when <デバイス名> is finished. (code -10)
```

# 対応

- XCode > Preferences > locations > Derived data の小さい矢印をタップ

![](https://storage.googleapis.com/zenn-user-upload/f728737a015fb5025d277940.png)

- Finderが開くので、Derived data以下のフォルダをすべて削除し、再起動する

![](https://storage.googleapis.com/zenn-user-upload/d5ef0e1f51eccb1fb77c13ba.png)

# 参考

- Flutter大学のtimesチャンネルでつぶやいたら、以下の記事を[こんぶさん](https://zenn.dev/pressedkonbu)に教えてもらいました。ありがとうございました。


https://www.reddit.com/r/iOSProgramming/comments/j1hsl3/copying_cache_files_from_device/?utm_source=share&utm_medium=ios_app&utm_name=iossmf

