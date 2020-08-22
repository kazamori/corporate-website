---
title: "Hugo で SNS 向けのシェアボタンを追加する"
slug: add-hugo-social-buttons
date: 2020-08-22T17:22:43+09:00
with_date: true
tags: [hugo, golang, template]
---

前回の [Hugo で前後の記事へのリンクを追加する]({{< relref "add-hugo-prev-next-link" >}}) の続きです。

今回はシェア用途の各種ソーシャルボタンを追加してみます。

Hugo に標準でそういった機能が用意されているのかな？と調べてみましたが、どうもそういうわけではないようです。
ググってみると、過去に同種の質問は多く、いくつか方法はみつかります。

調べ始めたときの私の要望はこれらでした。

* 外部サイトに依存した仕組み (スクリプト) は使いたくない
* 保守しやすいように複雑な仕組みは導入したくない

いくつか試してみて私は次のブログで紹介されている方法を採用しました。

* [Hugoで作ったサイトにシェアボタンを足した](https://aakira.app/blog/2018/08/share/)

## シェアボタンを追加する

基本的には上述した記事で紹介されている方法のままです。

但し、当社のサイトではいくつか意図したように動作しなかったので少しだけ手を入れて変更しました。
おそらくサイトに適用しているテーマによって変わると思うので私が変更したところだけ簡単に紹介しておきます。

当社のサイトでは [min_night](https://themes.gohugo.io/min_night/) というテーマを採用しています。
このテーマが扱っているコンテンツの html を生成するテンプレートは
`layouts/_default/single.html` になります。
先日の前後の記事へのリンクを追加するときに
オリジナルのテンプレートをコピーしてカスタマイズすることにしました。

### マウスオーバー時のボタン毎の色指定が反映されない

例えば、twitter のボタンであればマウスオーバー時に青い色が反映されるよう、
次のように `custom.css` に設定しています。

```css
.twitter:hover a {
  color: #1B95E0;
}
```

しかし、テーマの `color` に `!important` が設定されているため、
意図したボタンごとの色が反映されませんでした。

```css
#bigbody main a {
    color: var(--accent) !important;
}
```

そこでテンプレート内の `<main>` タグの外部に出すことで
マウスオーバー時にシェアボタンごとの色が反映されるようになりました。
この変更により、個々の記事の html の構成も変わることになりましたが、
ほとんど誰もみていないようなサイトなので影響はないでしょう。

### シェアボタンを中央に揃える

テーマの css によるものだと推測しますが、
紹介されている方法をそのままもってきてもシェアボタンが左寄りになってしまい、
中央揃えになりませんでした。

私は css に疎いのでなにがどうという説明はできません。
適当にググって調べながらうまくいった方法を書いておきます。
紹介されているサイトでも中央揃えの方法はいくつかあると言及されていました。

記事の幅の制御がうまくいかなかったので `<div>` タグのクラスから `container` を取り除きました。

```html
<section class="section sns_parent">
  <div class="container sns_section">
```

から

```html
<section class="section sns_parent">
  <div class="sns_section">
```

に変更する。

中央揃えのための `custom.css` を次のように設定する。

テーマの css 設定で `text-align: center` が適用されていたのでそれも省略しています。
場合によっては必要かもしれません。`.sns_parent` の設定も不要でした。

```css
.sns_parent {
  text-align: center;
}

.sns_section {
  display: inline-block;
  text-align: left;
}
```

から


```css
.sns_section {
    display: inline-block;
}
```

に変更する。

### まとめ

実際のソースコードの差分は [5dbda356](https://github.com/kazamori/corporate-website/commit/5dbda356c9dab1b8cce4a3a8856ed12e5f46a8dd) になります。

シェアボタンとは関係ありませんが、
テンプレートが読みやすくなるよう、
前後の記事へのリンクを追加したテンプレートも
partial として管理するように変更しています。
