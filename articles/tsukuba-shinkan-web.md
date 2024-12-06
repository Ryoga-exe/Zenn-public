---
title: "筑波大学新歓 Web 2024 の舞台裏"
emoji: "🌸"
type: "tech"
topics: ["web", "astro", "css", "筑波大学", "adventcalendar"]
published: false
---

この記事は、[mast Advent Calendar 2024](https://adventar.org/calendars/10425) 8 日目の記事です。
私は情報科学類生で情報メディア創成学類生でもなんでもないのですが、知人に書こうと勧められた[^1]ので情報メディア創成の気持ちになって書いていこうと思います。
7 日目の記事は [🍏](https://x.com/ao_ringo_uni) さんの「2024 ベストバイ」でした。

2024 年 4 月に筑波大学新歓祭が開催されました。
私は、[新歓祭 Web サイト](https://shinkan-web.zdk.tsukuba.ac.jp/)の制作を担当したので、工夫した点などの裏話を少しお話できればと思います。

[^1]: というか「書け！！！」と脅された

## 新歓祭とは

筑波大学では毎年 4 月に新入生歓迎委員会主催の下、新歓祭が開催されます。
これは新入生を歓迎する大学の一大イベントであり、サークルや部活動、学生団体の大規模な新歓が行われます。

この新歓祭に関する情報を多くの新入生に届け、また各参加団体の詳細情報を提供するための Web サイトとして「新歓 Web」を作ることとなりました。

![新歓 Web のスクリーンショット。上部には検索バーが表示され中央部に団体が一覧表示されている。サイドバーに新歓祭のロゴが大きく表示されている。](/images/tsukuba-shinkan-web/screenshot.png)
_今年度の新歓 Web のスクリーンショット - https://shinkan-web.zdk.tsukuba.ac.jp より引用_

## 掲載情報

サイトには以下の情報を掲載することとなりました。

- 団体情報
  - 一覧ページ
  - 団体個別ページ
- 新歓祭についての情報
  - 新歓祭についての案内文
  - 新歓祭パンフレット
  - 開催日時
  - 開催場所
  - 入場について
  - 諸注意

また、機能として

- 団体の検索機能
  名前やキーワードにより検索することができる
- お気に入り機能
  気になった団体をブックマークできる

を実装することとなりました。

## デザイン

ロゴと OG 画像については[ぱうろ](https://x.com/210on)さんが担当してくれました。
特にロゴについては [Advent Calendar 2 日目の記事](https://note.com/210on/n/n9cc7c8b2a54e)で触れられているので是非そちらも併せてご覧ください！

サイトのメインのデザインは [🍏](https://x.com/ao_ringo_uni) さんが担当してくれました。
ロゴデザインの雰囲気に合わせ、温かみがあり、柔らかさを感じるデザインになっています。

![新歓 Web の OG 画像。中央部に新歓祭のロゴと「新歓祭情報・団体紹介サイト 新歓 Web」と書かれている。](/images/tsukuba-shinkan-web/og-image.png)
_今年度の新歓 Web の OG 画像 - https://shinkan-web.zdk.tsukuba.ac.jp より引用_

## 実装

フロントエンドのフレームワークとして [Astro](https://astro.build/) を採用し、[Cloudflare](https://www.cloudflare.com/) にホスティングしました。

Astro を採用した理由ですが、シンプルで使いやすく、パフォーマンスの高いサイトを構築しやすいからというのが大きかったです。
また、今回はそこまで複雑な Web アプリケーションではないため、Astro の特性が最適だと判断しました。

そして、CMS として [Strapi](https://strapi.io/) が、検索のためのバックエンドとして [Meilisearch](https://www.meilisearch.com/) が建っています。
Strapi と Meilisearch については [raspi0124](https://x.com/raspi0124) さんがメインで担当してくれました。

検索機能については [Astro の SSR 機能](https://docs.astro.build/en/guides/on-demand-rendering/)を用いて実装しています。団体個別のページについては Strapi のデータからページを SSG しています。

パフォーマンスの高い Astro と高速な検索が可能な Meilisearch、そして適切に SSR と SSG を使い分けることにより、快適なユーザ体験を実現できたと思います。
新歓 Web は団体検索機能を有しているため、新歓祭の最中に屋外で使用されることが予想されます。そういった点からパフォーマンスを高めることは個人的に特に意識したポイントでした。

以下、こだわった点などについて要素ごとに述べていきます。

### 団体一覧

検索ワードが空文字である場合、全ての団体が表示されます。
この並び順ですが、公平性の観点から半日に1回順序をランダムに並び替えています。

以下のような実装をしており、半日周期に変わる値を乱数のシードとして、それをもとに並び替えています。

```ts
if (query === "") {
  const today = new Date(
    Date.now() + (new Date().getTimezoneOffset() + 540) * 60 * 1000,
  );
  const seed =
    today.getFullYear() * today.getMonth() +
    today.getDate() *
      (today.getHours() * 100 + today.getMinutes() > 1200 ? 7 : 13);
  const random = new RandXor(seed);

  orgs.sort(() => random.next() - random.next());
}
```

### 団体個別ページ

団体個別ページは大きなスライダーが目立ちます。このスライダーは [Splide](https://ja.splidejs.com/) を使用して実装しています。
[Splide のサムネイルスライダーのチュートリアル](https://ja.splidejs.com/tutorials/thumbnail-carousel/)を参考に、プレビュー用の小さな画像からなるスライダーをクリックすると、メインのスライダーも切り替わる、2 つのスライダーが同期するもの実装しました。

スライダーを実装するにあたり、様々なライブラリを比較しました。Splide は他のライブラリと比べ、小さく軽いことや、柔軟性があること、アクセシビリティに優れていることから採用しました。

![団体個別ページのスクリーンショット](/images/tsukuba-shinkan-web/org-screenshot.png)
_団体個別ページのスクリーンショット - https://shinkan2024.pages.dev/orgs/102 [^2]より引用_

[^2]: ちなみに筆者は[学園祭実行委員会](https://sohosai.com)に所属しています

### 検索機能・絞り込み

前述した通り、検索機能については Astro の SSR 機能を用いて実装しています。
クエリパラメータをもとにフィルタするようにしており、`q=` で指定された文字列をもとに Meilisearch で検索し、`c[]=` で指定された種別でフィルタリングをしています。

お気に入りのみを表示する機能として、`f=` が指定された場合には該当する団体のみを表示させるようにしています。

![クエリパラメータの構造の説明](/images/tsukuba-shinkan-web/query.png)
_クエリパラメータの構造_

### お気に入り機能

![ハートのアニメーション](/images/tsukuba-shinkan-web/heart.gif)

### サイドバー

レスポンシブ対応

![](/images/tsukuba-shinkan-web/responsive.gif)

## むすびにかえて
