---
title: "【Flutter】SVGが黒く表示されてしまう時の対処法"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter"]
published: true
---

# 症状

https://pub.dev/packages/flutter_svg

こちらのパッケージで SVG を表示しようとしたところ、色があるはずの場所が黒くなってしまった。

# 原因

README をよく読むと書いてあったのですが、
このパッケージは<style>タグに対応していないようで、色を指定している部分が読み込めずに黒くなってしまっていた模様。

> Note: The library currently only detects unsupported elements (like the <style>-tag), but not unsupported attributes.

https://pub.dev/packages/flutter_svg#check-svg-compatibility

```svg
...
<svg version="1.1" id="レイヤー_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px"
	 y="0px" viewBox="0 0 375 632" style="enable-background:new 0 0 375 632;" xml:space="preserve">
<!-- ここが読み込めない  -->
<style type="text/css">
	.st0{fill:#FF63A0;}
	.st1{fill:#009B01;}
	.st2{fill:#003CBE;}
</style>
<path class="st0" d="M284.4,131l142.3-57.5c18.4-7.5,39.4,1.5,46.9,19.9l0,0c7.5,18.4-1.5,39.4-19.9,46.9l-142.3,57.5
	c-18.4,7.5-39.4-1.5-46.9-19.9l0,0C257,159.4,265.9,138.4,284.4,131z"/>
...
</svg>
```

# 解決策

「SVGCleaner」というツールを使って、<style>タグを除いたファイルへ変換する。

https://github.com/RazrFalcon/svgcleaner-gui

```svg
<svg height="222.96mm" viewBox="0 0 375 632" width="132.29mm" xmlns="http://www.w3.org/2000/svg"><rect fill="#ff63a0" height="72.03" rx="36.01" transform="matrix(.92718385 -.37460659 .37460659 .92718385 -23.94 148.11)" width="225.48" x="256.28" y="99.61"/><rect fill="#009b01" height="72.03" rx="36.01" transform="matrix(.92718385 -.37460659 .37460659 .92718385 -3.09 109.84)" width="365.64" x="98.18" y="26.86"/><rect fill="#ff63a0" height="72.03" rx="36.01" transform="matrix(.92718385 -.37460659 .37460659 .92718385 -205.78 1.12)" width="365.64" x="-282.82" y="493.86"/><rect fill="#003cbe" height="72.03" rx="36.01" transform="matrix(.92718385 -.37460659 .37460659 .92718385 -36.49 -11.96)" width="365.64" x="-231.82" y="51.86"/><rect fill="#009b01" height="72.03" rx="36.01" transform="matrix(.92718385 -.37460659 .37460659 .92718385 -122.67 12.19)" width="225.48" x="-142.72" y="285.61"/><rect fill="#003cbe" height="72.03" rx="36.01" transform="matrix(.92718385 -.37460659 .37460659 .92718385 -96.62 178.18)" width="225.48" x="297.28" y="301.61"/><circle cx="75.9" cy="196.05" fill="#003cbe" r="25.05"/><circle cx="149.05"
...
</svg>
```

# 参考

https://stackoverflow.com/questions/58567864/im-trying-use-flutter-svg-but-load-with-black-svg
