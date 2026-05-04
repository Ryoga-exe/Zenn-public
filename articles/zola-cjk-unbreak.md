---
title: "Zola で改行時に不自然なスペースが入る問題をどうにかしたい"
emoji: "📃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zola", "tera", "markdown"]
published: false
---

Zola は高速にビルドができることが特徴の SSG です。
私は個人的なブログを Zola で構築しており、それをしばらく使い続けています。

Zola 自体にはかなり満足しているのですが、日本語の Markdown を書いているときに少し気になることがありました。

## 改行時に不自然なスペースが入ってしまう

私は、Markdown を書く際に句点の後に改行をしがちなのですが、これのせいで表示上は不自然な半角スペースが入ったように見えてしまいます。

例えば

```md
これは日本語の文章です。
こんにちは。
```

と書くと、表示上は次のように見えてしまいます。

```
これは日本語の文章です。 こんにちは。
```

改行が不自然な半角スペースとして現れてしまうのです。
エディタ上では適当な位置で改行したいのですが、表示上は日本語の文章として自然につながってほしい、という気持ちがあります。

英語であれば単語間にスペースが入るのは自然ですが、日本語では文中に突然半角スペースが入ると少し目立ちます。

似たような問題に対して、Typst には [cjk-unbreak](https://typst.app/universe/package/cjk-unbreak/) というパッケージが存在し、これを解決できます。今回は、これと似たようなことを Zola でも行いたいわけです。

## 解決策

Zola はテンプレートエンジンとして [Tera](https://keats.github.io/tera/) を使用しています。
今回は、そのテンプレート側で `regex_replace` を使って CJK 文字間の改行を消す方針で対処しました。
具体的には以下のようなマクロを作成しました。

```jinja2:templates/macros/cjk.html
{% macro unbreak(html) %}
{{ html
  | regex_replace(
      pattern=`([\p{Han}\p{Hiragana}\p{Katakana}々〆〤、。，．・；：！？‘’“”（）「」『』【】〈〉《》…—ー])\r?\n([\p{Han}\p{Hiragana}\p{Katakana}々〆〤、。，．・；：！？‘’“”（）「」『』【】〈〉《》…—ー])`,
      rep=`$1$2`
    )
  | safe
}}
{% endmacro %}
```

以下のように使います。

```jinja2
{% import "macros/cjk.html" as cjk %}

<article>
  {{ cjk::unbreak(html=page.content) }}
</article>
```

マクロで使用している正規表現は、前述した Typst の `cjk-unbreak` のコードを参考にしました。

## 注意点

この方法は Markdown の AST ではなく、生成済み HTML 文字列に対する後処理にすぎません。
そのため、タグをまたぐケースに効かなかったり、逆にコードブロック内に日本語がある場合に効いてしまったりすることがあります。

例えば

```markdown
これは[リンク](https://example.com)
です。
```

のようなケースでは、HTML 化後にタグが挟まるため、単純な正規表現では期待通りに改行を消せません。

厳密には Markdown の AST レベルで処理するのが理想ですが、今回の私のブログではこのようなケースは起こらなそうだったため許容としています。

## 余談

### Zola の制約

少なくとも現在の Zola では、設定だけで Markdown の Soft line break の扱いを差し替えることは難しそうです。

Zola 本体の issue には Markdown render hooks の Tracking issue があり、その中で `pulldown_cmark::html::push_html` の HTML 出力をカスタマイズできるか、という話題も挙がっています。（2026 年 5 月 4 日現在）

https://github.com/getzola/zola/issues/2307

また、Zola 0.23 に向けて Markdown rendering を大きく書き直したい、という issue も開かれています。

https://github.com/getzola/zola/issues/3070

そのため現状では、Zola 本体の設定で解決するというより、今回のようにテンプレート側で `page.content` を後処理するのが手軽そうです。

### これは Zola 特有の問題ではない

この問題は Zola 特有のものではありません。

Markdown / CommonMark では、段落内の普通の改行は Soft line break として扱われます。
HTML 上では改行も空白文字の一種なので、ブラウザ上では改行位置にスペースが入ったように見えます。

CommonMark の議論でも、Soft line break をスペースとして扱うと、中国語など CJK の文章で不自然な空白が出るという問題が指摘されています。

https://talk.commonmark.org/t/soft-line-breaks-should-not-introduce-spaces/285

実際、他の Markdown 処理系や SSG では、この問題に対する機能やプラグインが存在します。

Pandoc には `east_asian_line_breaks` という拡張があります。

https://pandoc.org/MANUAL.html#extension-east_asian_line_breaks

また、markdown-it には `markdown-it-cjk-breaks` というプラグインがあります。

https://github.com/markdown-it/markdown-it-cjk-breaks

Hexo にも `hexo-filter-fix-cjk-spacing` というプラグインがあります。

https://github.com/lotabout/hexo-filter-fix-cjk-spacing

## まとめ

今回は Tera の `regex_replace` を使って、CJK 文字間の改行を削除するマクロを作り対処しました。

生成される HTML に対する後処理なので完璧ではありませんが、私のブログでは十分実用的でした。
同じように Zola で日本語ブログを書いていて、改行位置に半角スペースが生えるのが気になる場合は、このようなマクロを挟むと改善できるかもしれません。

また、今後の Zola のアップデートにより、Markdown のレンダリング段階でもう少し本質的に処理できるようになるかもしれません。
