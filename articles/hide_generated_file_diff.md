---
title: "【Flutter開発Tips】自動生成ファイルの diff をプルリクに表示しないようにする"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['GitHub','Flutter',]
published: true
---

## やりたいこと

[freezed](https://pub.dev/packages/freezed) や [json_serializable](https://pub.dev/packages/json_serializable) 、[flutter_gen](https://pub.dev/packages/flutter_gen) など、Flutter には便利な code generator がたくさんあり、私自身も愛用させていただいています。

そんな便利なコード生成ツールですが、生成されたファイルの記述量が多く、プルリクの diff を見る時にちょっぴり邪魔だったりします。



![](https://storage.googleapis.com/zenn-user-upload/f40bd6fe9b53-20220509.png)



ignore するべきファイルでも無いと思うので、縮小した形で表示出来れば嬉しいですね。

## 解決策

`.gitattributes` ファイルをプロジェクトのルートに配置し、非表示にしたいファイルを記述する。

```yaml:.gitattributes
# *<自動生成ファイルの拡張子> linguist-generated=true
*.freezed.dart linguist-generated=true
*.g.dart linguist-generated=true
*.gen.dart linguist-generated=true
*.gr.dart linguist-generated=true
```

こうすることで、自動生成コードの diff が表示されなくなりました 🎉


![](https://storage.googleapis.com/zenn-user-upload/23a961e96cad-20220509.png)

生成コードの内部をチェックすることはあまり無いとは思いますが、Load diff を押せば一応内部の変更も見れるので便利ですね。

どなたかの助けになれば幸いです。

## 参考

[やまたつさんのツイート](https://twitter.com/yamatatsu109_ja/status/1489801580329066498?s=20&t=decWewkD7ENUL37Otf2OPA)でこの存在を知りました！
便利な知見をありがとうございます！🙌

https://docs.github.com/en/repositories/working-with-files/managing-files/customizing-how-changed-files-appear-on-github