---
title: "Siv3D で FeliCa リーダを使う"
emoji: "🪪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["siv3d", "felica"]
published: false
---

この記事は [Siv3D Advent Calendar 2025](https://qiita.com/advent-calendar/2025/siv3d) 9 日目の記事です。

## はじめに

今年の学園祭で展示したゲームで、FeliCa リーダを使ったユーザ管理・ログイン機構を実装しました。
学生証や Suica カードをリーダにかざすだけでログインできる仕組みは、体験として想像以上にウケが良かったです。

本記事では、Siv3D でカードリーダを使って FeliCa の IDm を取得する方法についてまとめます。

## 環境

今回使用した環境は Windows + Siv3D v0.6.16 です。
また、リーダには PaSoRi RC-S300 を使いました。[^1]

https://www.sony.co.jp/Products/felica/consumer/products/RC-S300.html

そのため、本記事ではこの環境を想定しています。

[^1]: おそらく RC-S380 などでもいけると思います。

## 実装

読み取りは PC/SC (WinSCard) を叩くことで行いました。
WinSCard (`winscard.dll`) は Windows に入っているライブラリで、最低限カードリーダから情報を得るための機能が入っています。

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

	// リーダ列挙
	DWORD mszLen = 0;
	if (SCardListReaders(c.ctx, nullptr, nullptr, &mszLen) != SCARD_S_SUCCESS or mszLen <= 2)
	{
		self->m_ok = false;
		self->m_running = false;
	}
	std::vector<wchar_t> msz(mszLen);
	if (SCardListReaders(c.ctx, nullptr, msz.data(), &mszLen) != SCARD_S_SUCCESS) {
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
		if (SCardGetStatusChange(c.ctx, 200, &st, 1) != SCARD_S_SUCCESS)
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
		if (SCardConnect(c.ctx, readerW.c_str(), SCARD_SHARE_SHARED, SCARD_PROTOCOL_T0 | SCARD_PROTOCOL_T1, &card.handle, &proto) != SCARD_S_SUCCESS)
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
- `SCardListReaders` でリーダ名を列挙
- `SCardGetStatusChange` で短いタイムアウトでポーリングしつつカードがかざされるのを監視
- `SCardConnect` で PCI を選択
- `SCardTransmit` で APDU `FF CA 00 00 00` を送信
- `SCardDisconnect` で切断

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
