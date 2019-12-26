---
title: "Hugoで企業ホームページを作る"
date: 2019-12-27T17:07:18+09:00
with_date: true
tags: ["hugo", "golang"]
---

この企業ホームページは [Hugo](https://gohugo.io/) という Go 言語で開発された
Static Site Generator (静的サイトジェネレーター) を使って作成しています。

Slant というプロダクトのお奨めをする Q&A サイトの Static Site Generator のランキングでは3位になっています。
あるコミュニティにおけるランキングではありますが、人気のある Static Site Generator の1つには違いないでしょう。

* [What are the best static site generators?](https://www.slant.co/topics/330/~best-static-site-generators)

以前、勤めていた企業で私が開発したプロダクトの紹介ページがなかったので当時 Hugo で作りました。
実際にはそれは採用されず、そのとき作ったものはお蔵入りになったものの、
Hugo の使い勝手がよかったので記憶に残っていました。
それから数年が経ち、自社のホームページを作る機会ができたのでまた使ってみることにします。
[Hugo のリポジトリ](https://github.com/gohugoio/hugo) をみる限りは活発に開発が継続されているようです。

なにかしら当社の技術情報をブログで発信する過程で
Hugo を使ってみてのノウハウや所感などを書いてみようと思います。
ソースコードはすべて次のリポジトリで公開しています。

* [github.com/kazamori/corporate-website](https://github.com/kazamori/corporate-website)

これから作ってみようと思う方は動くサンプルとして参考にしていただければと思います。
