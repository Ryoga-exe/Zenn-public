---
title: "Siv3D で知覚的色空間を使う - Oklab と Oklch 色空間"
emoji: "🎨"
type: "tech"
topics: ["siv3d", "opensiv3d", "design"]
published: true
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

![Oklab 空間における直交座標 (Oklab) と極座標 (Oklch)](https://evilmartians.com/static/2a08d3d2ca022b7d57d8ad75ac9459ba/c6a69/oklab-vs-oklch.webp)
_Oklab 空間における直交座標 (Oklab) と極座標 (Oklch) - https://evilmartians.com/chronicles/oklch-in-css-why-quit-rgb-hsl より引用_

Oklab と Oklch 色空間は、2020 年に [Björn Ottosson](https://x.com/bjornornorn) により作成されました。[^1]
割と最近できたものですが、CSS 仕様に追加されたり[^2]、Photoshop にグラデーション補完のために追加されたり[^3]など、すでに様々な場面で導入されています。

Oklch については[直感的に計算してくれるツール](https://oklch.com/)があり、パラメータを調節しながらその結果をリアルタイムに確認することができます。適当にグリグリと動かすと、どのような構造になっているか、それぞれのパラメータがどのように影響するかがなんとなくわかると思います。

![](/images/siv3d-perceptual-color/oklch-color-picker.gif)
_OKLCH Color Picker でグリグリと動かしている様子 - https://oklch.com より引用_

ここまで Oklab と Oklch がどういったものかを説明してきました。それでは実際の応用例について見ていきます。
前節でボタンの例を挙げて HSV ではコントラストを均等に調整することが難しいということを述べましたが、Oklch を使ってボタンの背景色のコントラストを揃えてみます。

次の画像は Oklch 色空間を使ってみた例です。2 つのボタンは明度 (L) と彩度 (C) が同じで色相 (H) のみ異なります。HSV で実装したときの例と異なり、異なる色相によってコントラストが異なるという問題が起きないのが確認できると思います。

![ボタンの例、HSV で実装したときと違い、コントラストを揃えることができている](/images/siv3d-perceptual-color/button-example-oklch.png)

[^1]: https://bottosson.github.io/posts/oklab/

[^2]: https://www.w3.org/TR/css-color-4/#ok-lab

[^3]: https://helpx.adobe.com/photoshop/using/gradient-interpolation.html

## Siv3D で Oklab と Oklch を実装する

`Color` 型や `ColorF` 型に変換する関数があれば十分 Siv3D で使えそうです。
変換公式については [Wikipedia](https://en.wikipedia.org/wiki/Oklab_color_space) や [Björn Ottosson 氏の記事](https://bottosson.github.io/posts/oklab/)に記載があります。どちらも英語ですが、頑張って読み解くと単に特定の行列を掛けてやることで変換できることがわかります。

以下に Siv3D での実装例を示します。

```cpp
# include <Siv3D.hpp> // Siv3D v0.6.15
# include <numbers>

struct Oklab
{
	double l;
	double a;
	double b;

	[[nodiscard]]
	Oklab() = default;

	[[nodiscard]]
	Oklab(const Oklab&) = default;

	[[nodiscard]]
	constexpr Oklab(double _l, double _a, double _b) noexcept
		: l{ _l }
		, a{ _a }
		, b{ _b } {}

	[[nodiscard]]
	ColorF toColorF() const noexcept
	{
		const auto ld = std::pow(l + 0.3963377774 * a + 0.2158037573 * b, 3);
		const auto md = std::pow(l - 0.1055613458 * a - 0.0638541728 * b, 3);
		const auto sd = std::pow(l - 0.0894841775 * a - 1.291485548 * b, 3);

		return ColorF{
			+4.0767416621 * ld - 3.3077115913 * md + 0.2309699292 * sd,
			-1.2684380046 * ld + 2.6097574011 * md - 0.3413193965 * sd,
			-0.0041960863 * ld - 0.7034186147 * md + 1.707614701 * sd,
		}.applySRGBCurve();
	}
};

struct Oklch
{
	double l;
	double c;
	double h;

	[[nodiscard]]
	Oklch() = default;

	[[nodiscard]]
	Oklch(const Oklch&) = default;

	[[nodiscard]]
	constexpr Oklch(double _l, double _c, double _h) noexcept
		: l{ _l }
		, c{ _c }
		, h{ _h } {}

	[[nodiscard]]
	Oklab toOklab() const noexcept
	{
		const auto theta = std::numbers::pi * h / 180.0;
		return Oklab
		{
			l,
			c * std::cos(theta),
			c * std::sin(theta)
		};
	};

	[[nodiscard]]
	ColorF toColorF() const noexcept
	{
		return toOklab().toColorF();
	}
};
```

使用例

![Oklch を Siv3D で使ってみた例](/images/siv3d-perceptual-color/screenshot.png)

:::details コード（Main 関数のみ）

```cpp
void Main()
{
	while (System::Update())
	{
		for (int32 i = 1; i <= 7; i++)
		{
			Circle{ 100 * i, 200, 40 }.draw(Oklch{ 0.7, 0.3, 360 * i / 7.0 }.toColorF());
		}

		for (int32 i = 1; i <= 7; i++)
		{
			Circle{ 100 * i, 400, 40 }.draw(Oklch{ 0.4, 0.3, 360 * i / 7.0 }.toColorF());
		}
	}
}
```

:::

## おわりに

知覚的色空間という言葉自体、普段あまり耳にする機会は少ないかもしれません。今回の記事で初めて知ったという方も多かったのではないでしょうか。
Oklab や Oklch といった色空間は視覚的に心地よく、調和の取れたデザインを作るための強力なツールです。特にゲームの UI 設計などにも取り入れることができると思います。
今回紹介した実装を組み込まなくても、カラーパレットを Oklab や Oklch 色空間から選ぶことを意識するだけで、より洗練されたビジュアルを目指すことができると思います。
ぜひ活用してみてください。

余談ですが、Siv3D の HSV のように色の相互変換等が充実したインタフェースを提供できるように、数日前から Oklab/Oklch 構造体の作成に取り組んでいます。[^4]やる気があるうちになんとか完成したいところです。

[^4]: https://github.com/Ryoga-exe/SivPerceptualColor
