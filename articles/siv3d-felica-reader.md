---
title: "Siv3D で FeliCa リーダーを使う"
emoji: "🪪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["siv3d", "felica"]
published: false
---

この記事は [Siv3D Advent Calendar 2025](https://qiita.com/advent-calendar/2025/siv3d) 9 日目の記事です。

今年の学園祭で展示したゲームで FeliCa リーダーを使ったユーザ管理・ログイン機構を作りました。
学生証や Suica カードをリーダーにかざすだけでログインできるようにしたところ、意外とウケが良かったので今回の記事でまとめようと思います。

## 環境

今回使用した環境は Windows + Siv3D v0.6.16 です。
また、リーダーには PaSoRi RC-S300 を使いました。

https://www.sony.co.jp/Products/felica/consumer/products/RC-S300.html

そのため、本記事ではこの環境を想定しています。

## 実装

読み取りには winscard.dll を使用しました。
winscard.dll は Windows に入っているライブラリで、最低限カードリーダーから情報を得るための機能が入っています。
