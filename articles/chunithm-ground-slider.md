---
title: "CHUNITHM のスライダーを PC で使う"
emoji: "🕹️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["game", "serial", "controller"]
published: false
---

> この記事は、[mast Advent Calendar 2025](https://adventar.org/calendars/11736) 15 日目の記事です。
> 14 日目の記事は [🍏](https://x.com/ao_ringo_uni) さんの「[Cloudflare Email Routing で予約完了メールから予定を自動作成する](https://zenn.dev/ao_ringo_uni/articles/046e54001c7562)」でした。

## はじめに

[CHUNITHM（チュウニズム）](https://chunithm.sega.jp/)はセガにより開発されているアーケード音楽ゲームです。
アーケード筐体の手前に、GROUND SLIDER（グラウンドスライダー）と呼ばれるコントローラが設置されており、画面奥から手前に流れてくる「ノーツ」に合わせてタッチすることで遊びます。[^1]

そんなスライダーを入手することができたので、入手した経路や、それを PC で使えるようにしてみたりしたのでそれについてまとめます。

[^1]: 他にも手を振り上げる動作を検知するセンサーが付いており、腕を振り上げたり[ヘドバン](https://youtu.be/gVKEajMLGnY)したりします。

## スライダーを入手する

ヤフオクの海を適当に彷徨っていたらたまたま既視感のあるスライダーが漂流していました。
15,000 円と微妙に手が届いてしまう値段だったので、勢いで購入してしまいました。

![家に届いた箱、本当にでかすぎる](/images/chunithm-ground-slider/slider.jpg)
_家に届いた箱、本当にでかすぎる_

かなりデカい箱で届きました。1 日で私の部屋が埋まってしまいました。
また、大部分が金属でできており、かなり重量があります。持ち上げると腰がやられます。

https://x.com/Ryoga_exe/status/1806307725133136263

箱から取り出した様子。
おそらく古い筐体から取り出したものだと思われます。多少汚れが目立ちます。

## PC に接続してみる

スライダーからは謎のコネクタが 2 本生えています。

![スライダーから生えている謎の 2 本のコネクタ](/images/chunithm-ground-slider/connector.jpg)
_スライダーから生えている 2 本の謎のコネクタ_

これが何なのか、謎です。謎なのでインターネットを漁ってみることにしました。
すると、[Chunithm Ground Slider | Rhythm Cons Wiki](https://rhythm-cons.wiki/controllers/chunithm/chunithm-ground-slider/) という謎の海外の Wiki を見つけました。
ここに、コネクタについて詳細な説明があります。

どうやら片方は RS-232C（D-Sub9） 相当、もう片方は 12V 給電なようでした。[^2]
RS-232C（±電圧）なので USB-TTL では動かないようです。

RS-232C 側は、USB–RS-232 変換アダプタなどを用いることで PC と接続することができます。
DB9 ブレークアウト基板とかいうものを買い、コネクタのピンから線を伸ばすのが確実ですが、時間がなかったので古いヤマハのルーターを分解し、ハンダ吸い取り機などを活用することで頑張って D-Sub9 コネクタ部分を取得しました。（バカすぎる）

![古いヤマハのルーターを分解してコネクタ部分を取り出している様子](/images/chunithm-ground-slider/disassemble-connector.jpg)
_古いヤマハのルーターを分解してコネクタ部分を取り出している様子_

DB9 のオスのピンが見えている側を正面から見た図は以下のようになっています。

```
1  2  3  4  5
 6  7  8  9
```

図はオス側を正面から見たときの番号です。裏面 / メス側だと鏡写しになるので注意してください。

このうち、2=RXD, 3=TXD, 5=GND を先程の Wiki の情報を参考に配線します。
スライダー側から生えているソケットは謎ですが、いろいろと試してみると、PC 用のケースファン 3 ピンのものがピッタリだったのでそれに繋ぐことにしました。

![配線したもの](/images/chunithm-ground-slider/connector-make.jpg)
_配線したもの_

あとは USB 変換アダプタを用いて PC に接続しました。

12V の給電は、2A の電源を使いました。
もちろん謎のコネクタなんて持っていないので、適当に銅線をコネクタに差し込み、ビニルテープでぐるぐる巻きにすることで接続しています。[^3]

[^2]: 先程の図において、オスのコネクタが 12V 給電、メスのコネクタが RS-232C です。

[^3]: どうやら調べてみると、給電用のコネクタは YLP-03V というコネクタ、もう片方は SMR-04V というコネクタだそうです。

## スライダーと通信してみる

Linux だと `/dev/ttyUSB*` に、Windows だとデバイスマネージャからシリアルポートにいます。
これといい感じにおしゃべりすることでスライダーを使うことができそうです。

当初は、オシロスコープやロジアナを使って頑張って通信を解析していましたが、普通に既に解析されているそうです。

自分で解析するのは骨が折れるので、大人しく先人の知恵を借りましょう。
プロトコルは、[初音ミク Project DIVA Arcade](https://miku.sega.jp/arcade/) のタッチスライダーと同じだそうで、[謎の Gist](https://gist.github.com/dogtopus/b61992cfc383434deac5fab11a458597) や、[こちらのサイト](https://ryun.halfmoon.jp/touchslider/slider_protocol.html)にまとめられています。

詳細なプロトコルはこれらのサイトに委ねるとして、いくつか主要なものを紹介します。

### プロトコル

パケットは、`SYNC(0xFF), CMD, LEN, PAYLOAD..., CHK` の順で構成されます。
`CMD` はペイロードの種別、`LEN` はペイロードのバイト数（ヘッダとチェックサムは含まない）で、`CHK` は二の補数のチェックサムです。
チェックサムは、パケットのバイトの総和が `0x00 (mod 256)` となるような数です。

主要なコマンドは以下の通りです。

- RESET (`0x10`)
  - `0xFF 0x10 0x00 0xF1` が正常な応答です。
- GetHWInfo (`0xF0`)
  - 応答は `LEN = 0x12` 固定です。
  - `model(8), device_class, chip_pn(5), ..., fw_ver` のようなパケットが返ってきます。
- StartInput (`0x03`) / StopInput (`0x04`)
  - StartInput コマンドにより、スライダーから約 12ms 間隔でタッチセンサの値が飛んでくるようになります。[^4]
- InputReport (`0x01`)
  - `LEN = 0x20` （32バイト）のタッチセンサの情報が返ってきます。
  - 各センサの値は強さに応じて `0x00` から `0xFE` を取り得ます。何もタッチしていないと値は `0x00` になります。
- LedReport (`0x02`)
  - `LEN = 0x5E` の`PAYLOAD` をスライダーに送ります。`PAYLOAD` は明るさ `1 + [B,R,G] * 31` のような形式です。

さらに調べてみると、これを Python で実装したリポジトリが見つかりました。

https://github.com/CrazyRedMachine/ChunithmIO

`slidertest/` ディレクトリにあるファイルを実行すると、スライダーを光らせたり、タッチセンサの値を取ったりすることができます。

[^4]: フレームレート換算で 83.3 FPS です。最新の CHUNITHM では 120FPS に対応した機種もあるため、最新機種のスライダーでは間隔が短くなっているのでしょうか。

## ライブラリを書く

このプロトコルを C++ で実装しました。フレームワークとして Siv3D を使っています。

https://github.com/Ryoga-exe/SivGroundSlider

`SivGroundSlider/` ディレクトリに Siv3D で使う前提のヘッダオンリーなライブラリとして実装しています。
実行すると、以下の動画のようになります。

https://x.com/Ryoga_exe/status/1983200762026176776

:::details 動画で使ったコード

```cpp
# include <Siv3D.hpp> // Siv3D v0.6.16
# include "SivGroundSlider.hpp"

static Array<String> GetSerialPortOptions(const Array<SerialPortInfo>& infos)
{
	Array<String> options = infos.map([](const SerialPortInfo& info)
	{
		return U"[{}] {}"_fmt(info.port, info.description);
	});

	options << U"None";
	return options;
}

static void DrawBars(SivGroundSlider::TouchFrame& frame)
{
	const double w = Scene::Width() / 16.0;
	const double h = Scene::Height() * 0.2;
	for (size_t i = 0; i < 32; ++i)
	{
		const double v = (frame.zones[i] / static_cast<double>(0xFE));
		const auto y = Scene::Center().y + (i % 2 == 0 ? 0 : h);
		const auto index = (31 - i) / 2;
		RectF{ w * index, y, w, h }.stretched(-2).draw(ColorF{v, 0, 0}).drawFrame(1, Palette::Gray);
	}
}

static Array<Color> LedReactive(SivGroundSlider::TouchFrame& frame)
{
	Array<Color> color(31);
	for (size_t i = 0; i < 31; ++i)
	{
		if (i & 1)
		{
			// separators
			color[i] = Color{ 0x23, 0x00, 0x7F };
		}
		else if (frame.zones[i])
		{
			// top zone
			color[i] = Color{ 0x00, 0x7F, 0x23 };
		}
		else if (frame.zones[i + 1])
		{
			// bottom zone
			color[i] = Color{ 0x7F, 0x23, 0x00 };
		}
	}
	return color;
}

void Main()
{
	using SivGroundSlider::GroundSlider;
	using SivGroundSlider::TouchFrame;

	Window::Resize(1000, 600);
	const Array<SerialPortInfo> infos = System::EnumerateSerialPorts();
	const Array<String> options = GetSerialPortOptions(infos);
	size_t index = (options.size() - 1);
	GroundSlider slider;

	const Array<String> testOptions{
		U"Color Test (Red)",
		U"Color Test (Green)",
		U"Color Test (Blue)",
		U"I/O Test",
		U"None",
	};
	size_t testIndex = (testOptions.size() - 1);
	bool prevTestIO = false;
	TouchFrame frame;

	while (System::Update())
	{
		if (SimpleGUI::RadioButtons(index, options, Vec2{ 200, 60 }))
		{
			ClearPrint();

			if (index == (options.size() - 1))
			{
				slider.close();
			}
			else
			{
				Print << U"Open {}"_fmt(infos[index].port);

				if (slider.open(infos[index].port))
				{
					Print << U"Open succeeded";
				}
				else {
					Print << U"Open failed";
				}

				Print << U"Initialize slider";

				if (slider.initialize())
				{
					Print << U"Initialize succeeded";
				}
				else
				{
					Print << U"Initialize failed";
				}

			}
		}

		bool enable = slider.initialized();
		if (SimpleGUI::RadioButtons(testIndex, testOptions, Vec2{ 750, 60 }, unspecified, enable))
		{
			if (prevTestIO)
			{
				slider.stopInput();
			}

			switch (testIndex)
			{
				case 0: slider.setLED(Color{ 255, 0, 0 }); break;
				case 1: slider.setLED(Color{ 0, 255, 0 }); break;
				case 2: slider.setLED(Color{ 0, 0, 255 }); break;
				case 3: {
					slider.setLED(Color{ 0, 0, 0 });
					slider.startInput();
					prevTestIO = true;
				} break;
				default : slider.setLED(Color{ 0, 0, 0 }); break;
			}
		}

		if (enable and testIndex == 3)
		{
			slider.update();
			while (true)
			{
				if (auto fr = slider.pop())
				{
					frame = *fr;
					slider.setLED(LedReactive(*fr));
				}
				else
				{
					break;
				}
			}
			DrawBars(frame);
		}
	}
}
```

:::

## まとめ

ヤフオクで勢いで購入してしまった CHUNITHM のスライダーでしたが、どうにか自分の PC で自由に使えるようにできました。
やはり、奇妙なハードウェアを触るのは面白いです。
