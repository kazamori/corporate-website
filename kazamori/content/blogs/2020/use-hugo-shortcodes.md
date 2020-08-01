---
title: "Hugo の Shortcodes テンプレートを使う"
slug: use-hugo-shortcodes
date: 2020-08-01T15:26:08+09:00
with_date: true
tags: [hugo, golang, template]
---

Hugo ではコンテンツを Markdown で記述します。

Markdown はプログラマーにとって身近な記法なので文章を書くことそのものへのコストが減ることで、書きやすいフォーマットと言うことはできますが、HTML に変換するときに十分な表現力がありません。例えば、画像ファイルのサイズを指定することさえできません。

コンテンツの中でちょっと HTML を加工したい、特定のパラメーターを指定して CSS でスタイルを適用したいといったときに Hugo の [Shortcodes](https://gohugo.io/content-management/shortcodes/) というテンプレートを使うと簡単に拡張できます。

## 組み込みの Shortcodes テンプレート

ドキュメントをみると、組み込みでいくつかテンプレートが提供されています。画像ファイルのサイズを変更するために [figure](https://gohugo.io/content-management/shortcodes/#figure) テンプレートが使えます。

figure テンプレートは次のように実装されています。ちなみに↓のソースコードは組み込みの [gist](https://gohugo.io/content-management/shortcodes/#gist) テンプレートを使って gist から取得しています。コンテンツの中にそのまま次のフォーマットで記述します。

```html
{{</* gist t2y 608f260609a545fde1cb3f4776f1c99a "figure.html" */>}}
```

{{< gist t2y 608f260609a545fde1cb3f4776f1c99a "figure.html" >}}

ソースコードをみると HTML の `<figure>` タグに対してパラメーターを設定していることが伺えます。

## 画像ファイルのサイズを指定

閑話休題。figure を使って画像ファイルのサイズも変更してみましょう。ここではロゴの幅を50に設定しています。

```html
{{</* figure src="/img/blue-logo-with-spaces.png" alt="logo" width=50 */>}}
```

{{< figure src="/img/blue-logo-with-spaces.png" alt="logo" width=50 >}}

Markdown では指定できない、こういったちょっとした変更に使いやすいものが Shortcodes テンプレートのよいところです。

## Shortcodes テンプレートを実装

ユーザー定義の Shortcodes テンプレートを作成するときは `layouts/shortcodes/` 配下に保存します。

試しに css で任意のクラスを指定できるように次のような p タグのテンプレートを作成してみました。

```html
<p{{ with .Get "class" }} class="{{ . }}"{{ end }}>
{{ .Get "text" }}
</p>
```

このファイルを `layouts/shortcodes/p.html` として保存します。

p タグに対する `sample-style` というカスタム css を次のように定義します。

```css
p.sample-style {
    color: blue;
    font-style: italic;
    margin-left: 25px;
}
```

準備は整いました。コンテンツにユーザー定義のテンプレートを使って css クラスを指定してみましょう。

```html
{{</* p class="sample-style" text="特別なスタイルを適用するテキスト" */>}}
```

{{< p class="sample-style" text="特別なスタイルを適用するテキスト" >}}

ちょっとテンプレートを作成して簡単にカスタマイズできるのが Shortcodes のよいところです。

## リファレンス

* [Shortcodes](https://gohugo.io/content-management/shortcodes/)
* [HUGO で独自 Shortcode を作る](https://blog.chick-p.work/blog/2019/20191015_hugo-shortcode/)
