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
  名前やキーワードにより団体を検索することができる
- お気に入り機能
  気になった団体をブックマークできる

を実装することとなりました。

これら掲載事項や団体情報の収集については [toririm](https://x.com/toriyoriMidori) さんが担当してくれました。私は連絡が苦手なのでかなり助かりました……

## デザイン

ロゴについては[ぱうろ](https://x.com/210on)さんが担当してくれました。
こちらについては [Advent Calendar 2 日目の記事](https://note.com/210on/n/n9cc7c8b2a54e)で触れられているので是非そちらも併せてご覧ください！

サイトのメインのデザインは [🍏](https://x.com/ao_ringo_uni) さんが担当してくれました。
ロゴデザインの雰囲気に合わせ、温かみがあり、柔らかさを感じるデザインになっています。

![新歓 Web の OG 画像。中央部に新歓祭のロゴと「新歓祭情報・団体紹介サイト 新歓 Web」と書かれている。](/images/tsukuba-shinkan-web/og-image.png)
_今年度の新歓 Web の OG 画像 - https://shinkan-web.zdk.tsukuba.ac.jp より引用_

## 実装

フロントエンドにはフレームワークとして [Astro](https://astro.build/) を採用し、[Cloudflare](https://www.cloudflare.com/) にホスティングしました。

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
_団体個別ページのスクリーンショット - https://shinkan-web.zdk.tsukuba.ac.jp/orgs/102 [^2]より引用_

また、この団体個別ページについては団体ごとに OG 画像を生成するようにしました。
[Astro の動的ルーティング](https://docs.astro.build/ja/guides/routing/#%E5%8B%95%E7%9A%84%E3%83%AB%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0)機能の `getStaticPaths()` 関数を用いてビルド時に `og/{団体ID}.png` という形式で [Satori](https://github.com/vercel/satori) を用いて生成しています。
今回初めて Satori を使いましたが、HTML で画像を生成できるのはかなり新鮮で使いやすかったです。
ただ、`background-image` に画像を指定する際に少々苦戦しました。`ArrayBuffer` として画像をインポートし、Base64 形式で埋め込むことで解決しましたがもっと良い方法はあるのでしょうか……

![団体個別で生成される OG 画像が並んでいる](/images/tsukuba-shinkan-web/org-og-images.png)
_生成される OG 画像たち_

今回は団体名のみ入れる簡単なものでしたが、サムネイル画像を入れるなどしても良かったのかなと思っています。
また、長い団体名だと中途半端なところで改行されてしまったため、[BudouX](https://developers-jp.googleblog.com/2023/09/budoux-adobe.html) などを活用しても良かったのかもしれません。

[^2]: ちなみに筆者は[学園祭実行委員会](https://sohosai.com)にも所属しています

### 検索機能・絞り込み

前述した通り、検索機能については Astro の SSR 機能を用いて実装しています。
クエリパラメータをもとにフィルタするようにしており、`q=` で指定された文字列をもとに Meilisearch で検索し、`c[]=` で指定された種別でフィルタリングをしています。

お気に入りのみを表示する機能として、`f=` が指定された場合には該当する団体のみを表示させるようにしています。

![クエリパラメータの構造の説明](/images/tsukuba-shinkan-web/query.png)
_クエリパラメータの構造_

### お気に入り機能

お気に入りした団体はブラウザの[ローカルストレージ](https://developer.mozilla.org/ja/docs/Web/API/Web_Storage_API/Using_the_Web_Storage_API)に保存されます。
保存される値ですが、クエリパラメータに乗せてバックエンドに送りフィルタするために使用されるため少しでも短くなるように設計しました。

基本的な実装の方針として、ビットが立っているかどうかを状態として持つようにしました。例えば `10001` の場合は ID が 0 と4 の団体がお気に入り登録されているといった感じです。
これを全ての団体に対して行えば `1000100100...` のようなビット列が得られるので、これを 5 桁ごとに区切り、32 進数のような形で保存しています。
具体的なコードは以下の通りです。

```ts
// お気に入りされている団体を localStorage から取得し、ID の配列として得る
let favoriteGroups = ((): string[] => {
  if (typeof localStorage !== "undefined") {
    const val = localStorage.getItem("favoriteGroups");
    if (val) {
      let ret = [];
      for (let i = 0; i < val.length; i++) {
        ret.push(
          ...[
            ...("0" <= val[i] && val[i] <= "9"
              ? parseInt(val[i])
              : val.charCodeAt(i) - 97 + 10
            )
              .toString(2)
              .padStart(5, "0"),
          ].reverse(),
        );
      }
      return ret;
    }
  }
  return [];
})();

// localStorage に保存する形式に変換する
function encodeFavoriteGroups(val: string[]): string {
  let ret = "";
  for (let i = 0; i < val.length; i += 5) {
    const num = parseInt(
      val
        .slice(i, i + 5)
        .reverse()
        .join(""),
      2,
    );
    if (num < 10) {
      ret += String(num);
    } else {
      ret += String.fromCharCode(97 + num - 10);
    }
  }
  return ret;
}
```

また、ハートを押した際のアニメーションも工夫したポイントです。
X (旧 Twitter) のいいねボタンの挙動を参考に、押して気持ちいいアニメーションを目指しました。

![](/images/tsukuba-shinkan-web/heart.gif)
_ハートのアニメーション_

動きは CSS アニメーションで実現しています。

```html
<button
  class="heart"
  aria-label="お気に入りに登録する"
  aria-pressed="false"
  data-group-id="{id}"
>
  <Icon class="like-line" name="ri:heart-3-line" aria-hidden />
  <Icon class="like-fill" name="ri:heart-3-fill" aria-hidden />
  <div class="effects" aria-hidden>
    <!-- ホバー時の背景の円 -->
    <span class="bg-circle"></span>
    <!-- クリックした際の円が広がるような演出 -->
    <span class="explosion"></span>
  </div>
</button>

<style>
  /* 一部省略 */
  .heart[aria-pressed="true"] .like-fill {
    display: block;
    animation: likeAnimation 1000ms forwards;
  }
  .heart[aria-pressed="true"] .explosion {
    animation: explosionAnimation 1000ms forwards;
  }

  @keyframes explosionAnimation {
    0% {
      width: 0.0001px;
      border: solid 0 #dd4688;
      opacity: 0;
    }
    20% {
      border: solid 0.75rem #cc8ef5;
      opacity: 1;
    }
    40% {
      width: 1.4rem;
      border: solid 0px #cc8ef5;
      opacity: 0;
    }
    100% {
      opacity: 0;
    }
  }
  @keyframes likeAnimation {
    0% {
      transform: scale(0);
    }
    30% {
      transform: scale(0);
    }
    40% {
      transform: scale(1.2, 1.2);
    }
    60% {
      transform: scale(1, 1) translate(0%, -10%);
    }
    65% {
      transform: scale(1.1, 0.9) translate(0%, 5%);
    }
    70% {
      transform: scale(0.95, 1.05) translate(0%, -3%);
    }
    80% {
      transform: scale(1, 1) translate(0%, 0%);
    }
  }
</style>
```

上記のコードはハートのコンポーネントの抜粋です。

また、アクセシビリティに配慮するため `aria-label` や `aria-pressed` を指定しました。
特に CSS アニメーションについては `aria-pressed` の値を参照することで動きを切り替えています。

### サイドバー

サイドに固定される部分はレスポンシブ対応を頑張りました。PC 表示では右側に、SP または、縦幅が小さい時は上部に表示されるようにしました。
ブレイクポイントを横幅だけでなく、縦幅にも設けることでどんな端末で見ても柔軟に表示されることを目指しました。[^3]

![](/images/tsukuba-shinkan-web/responsive.gif)
_様々な大きさで表示しているデモ、柔軟にレイアウトが切り替わっている - https://shinkan-web.zdk.tsukuba.ac.jp より引用_

[^3]: ちなみに、こちらも X (旧 Twitter) の挙動を参考にしています

### その他

パフォーマンスの他にアクセシビリティにも力を入れています。タブキーで適切に順番通りフォーカスがされるようになど、細かいところをこだわったつもりです。

また、404 ページはデザインとして上がっていなかったので私が一から考えました。404 の 0 を新歓祭のロゴにして遊び心を取り入れました。
404 を踏む人はあまりいなさそうな感じでしたが、気づいた人はいたのでしょうか。

![](/images/tsukuba-shinkan-web/404.png)
_404 ページのスクリーンショット - https://shinkan-web.zdk.tsukuba.ac.jp/404 より引用_

## むすびにかえて

こうして今年度の新歓 Web は作られ、[4/3 に公開されました](https://x.com/tsukuba_shinkan/status/1775345383339942340)。公開後、SNSなどを通じて多くの反響をいただき、多くの新入生や関係者の皆さまにご利用いただけたことを大変嬉しく思います。ご覧いただき誠にありがとうございました。

現在は一部改修などの作業をしています [^4]。コードも公開する方針でいますので続報をお待ちください！
来年度以降の新歓 Web サイトがさらに進化し、新しい形での情報提供が実現できるよう改善を続けていきたいと思っています。

末筆ではございますが、今回の開発にご協力いただいた全ての方々に、この場をお借りして心より感謝申し上げます。皆さまのご支援があったからこそ、新歓 Web を短期間[^5]で完成させることができました。
また、このような活動に興味が湧いた筑波大生・ITF.25 の皆さん！[全代会 情報処理推進特別委員会 (IPC)](https://www.stb.tsukuba.ac.jp/~zdk/ipc) でお待ちしています！新歓 Web のみならず、大学生活に密接に関わるさまざまな開発に取り組むことができます！

それでは来年度、2025 年度の新歓祭で！！

[^4]: 具体的にはバックエンド依存をやめる、アニメーションの改善などをしています

[^5]: というのも私自身、開発期間中ちょうど丸々免許合宿が被っており、公開されたくらいと同じくらいの日に仮免試験を受けていました。連絡の迅速な返信など無かったら普通に間に合わなかったかもしれません。
