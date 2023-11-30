# commands

## 新しい記事を作成する

```
npx zenn new:article
```

## プレビューを見る

```
npx zenn preview
```

# Markdown guides

## インラインスタイル

```
*イタリック*
**太字**
~~打ち消し線~~
インラインで`code`を挿入する
```

## コードブロック

```js:ファイル名

```

## 画像の横幅指定

```
![altテキスト](https://画像のURL =250x)
```

## 画像にリンクを貼る

```
[![altテキスト](画像のURL)](リンクのURL)
```

## テーブル

```
| Head | Head | Head |
| ---- | ---- | ---- |
| Text | Text | Text |
| Text | Text | Text |
```

## シンタックスハイライト

https://prismjs.com/#supported-languages

## 黄色アラート

```
:::message
メッセージをここに
:::
```

## 赤アラート

```
:::message alert
警告メッセージをここに
:::
```

## アコーディオン

```
:::details タイトル
表示したい内容
:::
```

## CodePen

```
@[codepen](ページのURL)
```
