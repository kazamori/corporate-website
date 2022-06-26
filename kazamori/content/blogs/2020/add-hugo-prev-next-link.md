---
title: "Hugo で前後の記事へのリンクを追加する"
slug: add-hugo-prev-next-link
date: 2020-08-21T19:56:28+09:00
with_date: true
tags: [hugo, golang, template]
---

Twitter のタイムラインでたまたま JJUG サイトを
[Hugo](https://gohugo.io/) を使ってリニューアルしたという投稿をみかけました。

{{< tweet user="cero_t" id="1296248908268494848" >}}

当社のサイトも Hugo で作っているのもあり、どんな雰囲気か覗いてみました。

次の画像は JJUG サイトのコンテンツの最下部を切り取ったものです。

{{< figure src="/blogs/2020/images/jjug-site-contents-bottom1.png"
           width=768
           alt="jjug-site-contents"
           title="JJUGサイトのコンテンツの最下部"
           class="common-style" >}}

次の違いに気付きました。

* シェア用途の各種ソーシャルボタン
* `PREVIOUS POST` と `NEXT POST` のボタン


ぱっとみて前後の記事へのボタンはシンプルでいいなと思えました。そこで当社のサイトでも追加するために調べてみました。

## テンプレートに前後の記事へのリンクを追加する

> Hugo’s templates are context aware and make a large number of values available to you as you’re creating views for your website.

Hugo のテンプレートには組み込み変数がたくさん用意されているとあります。

[Pages Methods](https://gohugo.io/variables/pages/) によると、
`Pages` オブジェクトがもつ `.Next`/`.Prev` メソッドを使うことで
意図したページへのリンクを取得できるようです。

公式ドキュメントでは次のように `.Site.RegularPages` から Permalink を取得する例が紹介されています。

```html
{{ with .Site.RegularPages.Next . }}{{ .RelPermalink }}{{ end }}
```

`.Site.RegularPages` からコンテンツを取得すると、すべてのコンテンツから日付順に取得します。

当社のサイトでは、それぞれメニュー別にコンテンツを管理しています。

* 事例紹介
* ニュース
* ブログ

例えば、ブログのコンテンツならば、ブログ内にあるコンテンツのみを扱いたいです。

そこで `.Parent.Pages` からコンテンツを取得することでメニュー内のコンテンツのみを扱えます。

```html
{{ with .Parent.Pages.Next . }}{{ .RelPermalink }}{{ end }}
```

実際にはコンテンツのタイトルを取得したり、タイトルの位置調整をしたり、
[Font Awesome](https://fontawesome.com/) のアイコンを設定したりして次のようになりました。

```html
{{ with .Parent.Pages.Prev . }}
<span class="prev-page far fa-arrow-alt-circle-left">
  <a href="{{ .RelPermalink }}">{{ substr .Title 0 35 }}</a>
</span>
{{ end }}
```

~~ちょっとした工夫として、タイトルが長くても1行におさまるように
[substr](https://gohugo.io/functions/substr/) を使ってタイトルを35文字以内にしています。~~

> #### 追記
> タイトルの文字数が多くて1行におさまらないときは複数行にわかれて表示されることがスマートフォンのブラウザで確認できました。とくに `substr` を使う必要はありませんでした。

これらの設定により、
次のように前後を示す矢印アイコンとコンテンツのタイトルリンクが表示されるようになりました。

{{< figure src="/blogs/2020/images/kazamori-contents-bottom1.png"
           width=768
           alt="kazamori-contents"
           title="当社サイトのコンテンツの最下部"
           class="common-style" >}}

## まとめ

実際のソースコードの差分は [a0bfc32f](https://github.com/kazamori/corporate-website/commit/a0bfc32f3d9775fe9c583063017ef1741dcbde62) になります。

テンプレートのカスタマイズと `Pages` にある情報を使ってコンテンツのタイトルやリンク情報を扱えることを学びました。

## リファレンス

* [Variables and Params](https://gohugo.io/variables/)

