---
title: "Hugo の URL をカスタマイズする"
slug: customize-hugo-url
date: 2019-12-29T15:09:46+09:00
with_date: true
tags: ["hugo", "golang", "naming"]
---

Hugo でコンテンツを作成するときに URL のカスタマイズについて紹介します。

## コンテンツのセクション

Hugo では `content` ディレクトリ配下にコンテンツを作成して管理します。
ここで `content` の直下にあるディレクトリを `section` (セクション) と呼び、
コンテンツを分類することに使えます。

本サイトでは次のように5つのセクションがあります。

```bash
$ tree -L 1 content
content
├── about
├── blogs
├── cases
├── news
└── service
```

本サイトではセクションを2つの用途に使っています。

* 1つのページしか存在しないとセクション: about, service
* 複数ページを含むセクション: blogs, cases, news

これらの用途によって作成するコンテンツのファイル名やディレクトリ構成も変わってきます。
最も基本的な名前付けとして、複数ページを含むセクションは意図的にセクション名を複数系にしています。

### 1つのページしかないセクション

`content` 配下のディレクトリに `index.md` という名前でファイルを作成します。
そうすると、そのディレクトリにあるコンテンツは1つのページを構成するものとして扱われます。
シンプルなページであれば `index.md` を作成して、そこにコンテンツを記述するだけでよいはずです。

### 複数ページを含むセクション

ブログのように時系列にコンテンツを追加していくようなセクションになります。
`index.md` を作らなければそういった扱いになります。
hugo new コマンドでコンテンツを作成するときは次のように `content` 配下のどこに作成するかを指定します。

```bash
$ hugo new blogs/2019/customize-hugo-url.md
path/to/content/blogs/2019/customize-hugo-url.md created
```

この他に `blogs` 直下にファイルをどんどん追加していくこともできます。

私は1つのディレクトリに無限にファイルが増えていくという構造そのものに懸念があるため、
年単位のディレクトリを作成して1年という分類で管理することにしました。
この構成のとき [/blogs/](https://kazamori.jp/blogs/) にアクセスすると、
その配下にあるコンテンツの一覧ページが表示されるようになります。

### リファレンス

* [Content Sections](https://gohugo.io/content-management/sections/)
* [Page Bundles](https://gohugo.io/content-management/page-bundles/)


## permalink (パーマリンク) のカスタマイズ

パーマリンクとは、1つ1つのページに設定される URL で、
そのコンテンツの内容を更新しても永久に変わらないことを意図するリンクです。
デフォルトでは `content` 配下のファイルへのパスが設定されます。

例えば次のようなディレクトリ構成の場合、

```bash
├── content
│   ├── blogs
│   │   └── 2019
│   │       ├── customize-hugo-url.md
```

次の URL がパーマリンクとして設定されます。

```
https://kazamori.jp/blogs/2019/customize-hugo-url/
```

このままでも悪くはないですが、あとでファイル名を変更したくなったときに
URL が変わってしまい、パーマリンクではなくなってしまいます。
私は名前を付けるのが下手なのであとからより適切な名前に変えることがよくあります。

そこで URL をカスタマイズしてみましょう。

`config.toml` に `[permalinks]` の設定をすることで任意の URL にカスタマイズできます。

```toml
[permalinks]
    cases = "/:sections/:year/:month/:slug/"
    blogs = "/:sections/:year/:month/:day/:slug/"
    news = "/:sections/:year/:month/:day/:slug/"
```

blogs はセクション名と年月日を入れて最後に `slug` という属性を設定しています。
`slug` を設定しない場合はデフォルトでタイトルが使われます。

```
https://kazamori.jp/blogs/2019/12/29/Hugo の URL をカスタマイズする/
```

URL に日本語を使うと URL エンコーディングされると長く読めない文字列になるため、
私は URL を英数字のみで設定する方が好みです。なので任意の文字列を設定したいです。
そこで `customize-hugo-url.md` の属性に `slug` を定義します。

```markdown
---
title: "Hugo の URL をカスタマイズする"
slug: customize-hugo-url
---
```

これでタイトルと `slug` を別々に設定しました。
あとでタイトルを変更してもパーマリンクには影響がありません。

最終的にいまご覧になっているように次の URL になります。

```
https://kazamori.jp/blogs/2019/12/29/customize-hugo-url/
```

### リファレンス

* [URL Management](https://gohugo.io/content-management/urls/)

## コンテンツの雛形にデフォルトの属性を設定

hugo new でコンテンツを作成するときに `slug` 属性を設定し忘れないように
`archetypes/default.md` にデフォルト値を設定しておきましょう。

```markdown
---
title: "{{ replace .Name "-" " " | title }}"
slug: {{ .Name }}
date: {{ .Date }}
with_date: true
tags: []
---
```

この設定により、デフォルトではファイル名が使われるようになります。

私はファイル名を日本語を使わないのでデフォルトはこれでよいです。
さらに後になってなんらかの理由でファイル名を変えたくなっても `slug` を変えない限りはパーマリンクには影響がないので安心です。

### リファレンス

* [Archetypes](https://gohugo.io/content-management/archetypes/)
