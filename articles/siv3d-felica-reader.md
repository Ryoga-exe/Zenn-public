---
title: "Siv3D で FeliCa リーダーを使う"
emoji: "🪪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["siv3d", "felica"]
published: false
---

この記事は [Siv3D Advent Calendar 2025](https://qiita.com/advent-calendar/2025/siv3d) 9 日目の記事です。

## はじめに

今年の学園祭で展示したゲームで、カードリーダーを使ったユーザ管理・ログイン機構を実装しました。
学生証や Suica カードをリーダーにかざすだけでログインできる仕組みは、体験として想像以上にウケが良かったです。

そこで本記事では、Siv3D でカードリーダーを使って FeliCa の IDm を取得する方法についてまとめます。

## 環境

今回使用した環境は Windows + Siv3D v0.6.16 です。
また、リーダーには PaSoRi RC-S300 を使いました。[^1]

https://www.sony.co.jp/Products/felica/consumer/products/RC-S300.html

そのため、本記事ではこの環境を想定しています。

[^1]: おそらく RC-S380 などでもいけると思います。

## 実装

読み取りは PC/SC (WinSCard) を叩くことで行いました。
WinSCard (`winscard.dll`) は Windows に入っているライブラリで、カードリーダーから情報を取得するための最低限の API が提供されています。

今回はこれを用いて `CardReaderWin` というクラスを作成し、Siv3D で使いやすい形式で実装しました。
コードは非常に長いため、折りたたみの中に載せます。

:::details コード

CardReaderWin.hpp

```cpp
# pragma once
# include <Siv3D.hpp>
# include <array>
# include <atomic>
# include <mutex>

class CardReaderWin
{
public:
	using IDm = std::array<uint8, 8>;

	CardReaderWin() = default;
	~CardReaderWin() { stopScan(); }

	CardReaderWin(const CardReaderWin&) = delete;
	CardReaderWin& operator=(const CardReaderWin&) = delete;

	void startScan();

	void stopScan();

	[[nodiscard]]
	bool isReady() const;

	[[nodiscard]]
	bool isOK() const;

	[[nodiscard]]
	IDm getIDm() const;

private:
	static void Read(CardReaderWin* self);

	IDm m_idm = {};

	AsyncTask<void> m_task;
	mutable std::mutex m_mutex;

	std::atomic<bool> m_running{ false };
	std::atomic<bool> m_ready{ false };
	std::atomic<bool> m_ok{ true };
};
```

CardReaderWin.cpp

```cpp
# include "CardReaderWin.hpp"

# define NOMINMAX
# include <Windows.h>
# include <winscard.h>
# include <vector>
# include <span>
# pragma comment(lib, "winscard.lib")

void CardReaderWin::startScan()
{
	stopScan();

	m_ready = false;
	m_ok = true;
	m_running = true;

	m_task = AsyncTask<void>(Read, this);
}

void CardReaderWin::stopScan()
{
	if (m_running.exchange(false))
	{
		if (m_task.isValid())
		{
			m_task.wait();
		}
	}
}

bool CardReaderWin::isReady() const
{
	return m_ready.load();
}

bool CardReaderWin::isOK() const
{
	return m_ok.load();
}

CardReaderWin::IDm CardReaderWin::getIDm() const
{
	std::scoped_lock lock{ m_mutex };
	return m_idm;
}

// helpers
namespace {
	struct PcscContext
	{
		SCARDCONTEXT ctx{};

		~PcscContext()
		{
			if (ctx)
			{
				SCardReleaseContext(ctx);
			}
		}

		bool establish()
		{
			return (SCardEstablishContext(SCARD_SCOPE_USER, nullptr, nullptr, &ctx) == SCARD_S_SUCCESS);
		}
	};

	struct PcscCard
	{
		SCARDHANDLE handle{};

		~PcscCard()
		{
			if (handle)
			{
				SCardDisconnect(handle, SCARD_LEAVE_CARD);
			}
		}
	};

	Array<String> SplitMultiString(const wchar_t* msz)
	{
		Array<String> out;
		if (not msz)
		{
			return out;
		}
		const wchar_t* p = msz;
		while (*p)
		{
			out << Unicode::FromWstring(p); // UTF-16 to UTF-32
			p += (wcslen(p) + 1);
		}
		return out;
	}

	// PaSoRi / RC-S3xx を優先選択（見つからなければ先頭）
	size_t ChooseReaderIndex(const Array<String>& readers)
	{
		for (size_t i = 0; i < readers.size(); ++i)
		{
			const auto& r = readers[i];
			if (r.includes(U"Sony") || r.includes(U"PaSoRi") || r.includes(U"RC-S3"))
			{
				return i;
			}
		}
		return 0;
	}

	bool IsOKSW(const uint8* buf, DWORD len)
	{
		return (len >= 2 and buf[len - 2] == 0x90 and buf[len - 1] == 0x00);
	}
}


void CardReaderWin::Read(CardReaderWin* self)
{
	// PC/SC 初期化
	PcscContext c;
	if (not c.establish())
	{
		self->m_ok = false;
		self->m_running = false;
		return;
	}

	// リーダー列挙
	DWORD mszLen = 0;
	if (SCardListReadersW(c.ctx, nullptr, nullptr, &mszLen) != SCARD_S_SUCCESS or mszLen <= 2)
	{
		self->m_ok = false;
		self->m_running = false;
		return;
	}
	std::vector<wchar_t> msz(mszLen);
	if (SCardListReadersW(c.ctx, nullptr, msz.data(), &mszLen) != SCARD_S_SUCCESS) {
		self->m_ok = false;
		self->m_running = false;
		return;
	}

	const auto readers = SplitMultiString(msz.data());
	if (readers.isEmpty())
	{
		self->m_ok = false;
		self->m_running = false;
		return;
	}
	const std::wstring readerW = readers[ChooseReaderIndex(readers)].toWstr();

	// IDm 取れたら終了
	SCARD_READERSTATEW st{};
	st.szReader = readerW.c_str();
	st.dwCurrentState = SCARD_STATE_UNAWARE;

	while (self->m_running)
	{
		// 200ms タイムアウトで監視
		if (SCardGetStatusChangeW(c.ctx, 200, &st, 1) != SCARD_S_SUCCESS)
		{
			continue;
		}
		const bool present = (st.dwEventState & SCARD_STATE_PRESENT) != 0;
		st.dwCurrentState = st.dwEventState;
		if (!present)
		{
			continue;
		}

		// 接続
		PcscCard card;
		DWORD proto{};
		if (SCardConnectW(c.ctx, readerW.c_str(), SCARD_SHARE_SHARED, SCARD_PROTOCOL_T0 | SCARD_PROTOCOL_T1, &card.handle, &proto) != SCARD_S_SUCCESS)
		{
			continue;
		}
		const SCARD_IO_REQUEST* const pci = (proto == SCARD_PROTOCOL_T0) ? SCARD_PCI_T0 : SCARD_PCI_T1;

		// IDm 取得
		std::array<uint8, 258> rIDm{};
		DWORD nIDm = static_cast<DWORD>(rIDm.size());
		static constexpr std::array<uint8, 5> CMD_IDM{ 0xFF, 0xCA, 0x00, 0x00, 0x00 };

		const bool ok =
			(SCardTransmit(card.handle, pci, CMD_IDM.data(), CMD_IDM.size(), nullptr, rIDm.data(), &nIDm) == SCARD_S_SUCCESS)
			and IsOKSW(rIDm.data(), nIDm)
			and nIDm >= 10;

		if (ok) {
			IDm got{};
			for (size_t i = 0; i < 8; ++i)
			{
				got[i] = rIDm[i];
			}

			{
				std::scoped_lock lock{ self->m_mutex };
				self->m_idm = got;
			}
			self->m_ready = true;
			self->m_ok = true;
			break;
		}
	}
	self->m_running = false;
}
```

:::

また、コードは GitHub 上でも公開しています。

https://github.com/Ryoga-exe/SivCardReader

流れは以下のようになっています。

- `SCardEstablishContext` で PC/SC を開始
- `SCardListReadersW` でリーダー名を列挙
- `SCardGetStatusChangeW` で短いタイムアウトでポーリングしつつ、在席を監視
- `SCardConnectW` で T=0/T=1 を交渉
- `SCardTransmit` に `proto` に応じて `SCARD_PCI_T0`/`SCARD_PCI_T1` を渡し、APDU `FF CA 00 00 00` を送信
- `SW1 SW2 == 90 00` なら先頭 8 バイトが IDm
- `SCardDisconnect` で切断

なお、APDU 受信バッファは最大 258 バイト（データ 256 + SW1/SW2 の 2 バイト）を常に確保するようにしています。

これを使った最小限のサンプルコードは以下のようになります。
「カードがかざされたら IDm を取得し、表示する」という簡単な動作です。

```cpp
# include <Siv3D.hpp> // Siv3D v0.6.16
# include "CardReaderWin.hpp"

String IDmToString(const CardReaderWin::IDm& idm)
{
	return U"{:02X} {:02X} {:02X} {:02X} {:02X} {:02X} {:02X} {:02X}"_fmt(idm[0], idm[1], idm[2], idm[3], idm[4], idm[5], idm[6], idm[7]);
}

void Main()
{
	CardReaderWin reader;
	reader.startScan();

	Optional<String> idm;

	while (System::Update())
	{
		ClearPrint();

		if (reader.isReady() and not idm)
		{
			idm = IDmToString(reader.getIDm());
		}

		Print << U"Status: " << (reader.isOK() ? U"OK" : U"ERROR");
		Print << U"IDm: " << idm.value_or(U"waiting...");
	}
}
```

学園祭で展示したゲームでは、取得した IDm を API サーバーに送ってユーザーを引き当て、スコアや履歴をデータベースに記録する構成にしました。
ゲームのログインや、ランキング集計がスムーズでした。

体験面の小さな課題として、Windows ではカード着脱のたびにデバイス接続音が鳴ることがあります。
展示環境では気になるので、「デバイス接続/切断」サウンドをオフにしておくと快適です。

## 注意点

今回の方法は、FeliCa (NFC-F) の IDm を識別子として扱うことを前提としています。
ところが、マイナンバーカードのような ISO/IEC 14443 系（多くは Type B）のカードは、非選択状態で返す識別子が毎回変わる設計になっています。
そのため、今回の実装の PC/SC の簡易コマンド (`FF CA 00 00 00`) で取れる値は固定の ID ではないため、使えません。[^2]
展示や運用で、きちんと識別したい場合は、FeliCa のみを対象とする・簡易的なフィルタをかけるなど対策が必要そうです。

また、 今回は学園祭というライトな場である・そもそもめちゃくちゃヤバイデータを記録しているわけではなかったため、このような運用で問題はありませんでしたが、IDm は認証の秘密ではない点に注意が必要です。
一般に IDm はカード固有の識別子として扱われますが、読み取りそのものに秘密性はなく、エミュレーション等による偽装の余地があります。
つまり、IDm をそのまま「本人確認」の根拠に使うのは危険なようです。

[^2]: 実際に、東工大の学生証ではうまく識別できませんでした。詳しくはよくわからなかったのですが、どうやら Visa カードがベースになっているとかなんとかで少々複雑なようです。

## まとめ

FeliCa の IDm を使ったライトなタッチでログイン体験は、Siv3D + PC/SC だけで簡単に実装することができました。

いくつか注意点に気をつければ、文化祭や学園祭でのゲーム展示などで活用できそうに思えます。
