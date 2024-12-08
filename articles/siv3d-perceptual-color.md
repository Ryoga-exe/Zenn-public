---
title: "Siv3D で知覚的色空間を使う - Oklab と Oklch 色空間"
emoji: "🌊"
type: "tech"
topics: ["siv3d", "opensiv3d", "design"]
published: false
---

この記事は [Siv3D Advent Calendar 2024](https://qiita.com/advent-calendar/2024/siv3d) 9 日目の記事です。

ゲームの UI を作る際、色の扱いは欠かせません。
特にカラーパレットのコントラストを均等に調整することは、美しさの他に見やすさにも影響します。

今回はそういった際に便利な色空間と Siv3D での実装を紹介します。

## HSV ではコントラストを均等に調整するのが難しい

HSV は色相 (Hue)、彩度 (Saturation)、明度 (Value) の 3 つの要素で色を指定する色空間です。RGB に比べて直感的に色を扱えるため、便利で多くの場面で使われていますが、とある問題を抱えています。

次の画像をご覧ください。
簡単なボタンを描画してみました。背景色は、左側は `HSV(270, 1.0, 0.5)`、右側は `HSV(90, 1.0, 0.5)` で指定されています。
どちらも同じ明度 `0.5` を指定していますが、右側のボタンでは背景色とのコントラストが低く、文字が読みづらく感じられるのではないでしょうか。

![ボタンの例、同じ明度であるがコントラストが異なる](/images/siv3d-perceptual-color/button-example.png)

この現象は、HSV の明度が同じでも、色相によって人間が感じる明るさが異なるために起こります。
HSV は便利ですが、コントラストを均等に保つデザインを目指す際には大きな問題があります。

こういった課題を解決するのが、Oklab や Oklch といった知覚的色空間です。
これらの色空間は、人間の視覚に基づいて色を均等に扱えるため、コントラストのバランスを均等に調整できます。

## Oklab と Oklch

異なる色相でもコントラスト比が一定になっていると非常に便利です。そんな風に設計された色空間が Oklab や Oklch です。

Oklab は、色を明度 (L)、赤緑成分 (a)、青黄成分 (b) の 3 つの要素で色を表します。この空間では**同じ明度 L の色は視覚的にコントラストが均一に感じられる**よう設計されています。
Oklch は、Oklab の極座標版です。同じ L 軸を持ち、極座標系の C (彩度) と H (色相) を使用します。

![Oklab 空間における直交座標 (Oklab) と極座標 (OKLCH)](https://evilmartians.com/static/2a08d3d2ca022b7d57d8ad75ac9459ba/c6a69/oklab-vs-oklch.webp)
_Oklab 空間における直交座標 (Oklab) と極座標 (OKLCH) - https://evilmartians.com/chronicles/oklch-in-css-why-quit-rgb-hsl より引用_

Oklab と Oklch 色空間は、2020 年に [Björn Ottosson](https://x.com/bjornornorn) により作成されました。[^1]
割と最近できたものですが、CSS 仕様に追加されたり[^2]、Photoshop にグラデーション補完のために追加されたり[^3]など、すでに様々な場面で導入されています。

Oklch については[直感的に計算してくれるツール](https://oklch.com/)があり、パラメータを調節しながらその結果をリアルタイムに確認することができます。適当にグリグリと動かすと、どのような構造になっているか、それぞれのパラメータがどのように影響するかがなんとなくわかると思います。

![](/images/siv3d-perceptual-color/oklch-color-picker.gif)
_OKLCH Color Picker でグリグリと動かしている様子 - https://oklch.com より引用_

ここまで Oklab と Oklch がどういったものかを説明してきました。それでは実際の応用例について見ていきます。
前節でボタンの例を挙げて HSV ではコントラストを均等に調整することが難しいということを述べましたが、Oklch を使ってボタンの背景色のコントラストを揃えてみます。

次の画像は Oklch 色空間を使ってみた例です。2 つのボタンは明度 (L) と彩度 (C) が同じで色相 (H) のみ異なります。HSV で実装したときの例と異なり、異なる色相によってコントラストが異なるとった問題が起きないのが確認できると思います。

![ボタンの例、HSV で実装したときと違い、コントラストを揃えることができている](/images/siv3d-perceptual-color/button-example-oklch.png)

[^1]: https://bottosson.github.io/posts/oklab/

[^2]: https://www.w3.org/TR/css-color-4/#ok-lab

[^3]: https://helpx.adobe.com/photoshop/using/gradient-interpolation.html

## Siv3D で Oklab と Oklch を実装する

Siv3D で利用するためには `Color` 型や `ColorF` 型に変換する関数があれば十分そうです。

## おわりに
