---
title: "Typeorm の Lazy relations とメモリ使用量"
slug: typeorm-lazy-relations-memory
date: 2021-07-12T15:23:05+09:00
with_date: true
tags: [typeorm, orm, performance]
---

[TypeORM](https://github.com/typeorm/typeorm) では、データベースから Entity を取得するときに関連している Entity を2種類の方法で取得できます。
ドキュメントは [Eager and Lazy Relations](https://github.com/typeorm/typeorm/blob/master/docs/eager-and-lazy-relations.md#eager-and-lazy-relations) になります。

* [Eager relations](https://github.com/typeorm/typeorm/blob/master/docs/eager-and-lazy-relations.md#eager-relations): Entity と関連 Entity を同じタイミングで読み込む
* [Lazy relations](https://github.com/typeorm/typeorm/blob/master/docs/eager-and-lazy-relations.md#lazy-relations): Promise を使って任意のタイミングで遅延して読み込む

そして TypeORM でデータを取得するとき、大きくわけて2通りの API があります。

* [Repository APIs](https://github.com/typeorm/typeorm/blob/master/docs/repository-api.md)
* [Select using Query Builder](https://github.com/typeorm/typeorm/blob/master/docs/select-query-builder.md)

ここで Eager relations で関連している Entity を取得するには Repository API の find 系メソッドを使う必要があります。
前記事の [TypeORM の @RelationId デコレーターとパフォーマンス]({{< relref "typeorm-relation-id-performance" >}})
で REPL を使って振る舞いの違いを確認しているのでそちらも参考にしてください。

どちらかと言えば、多くのアプリケーションで Eager よりも Lazy relations を使う機会の方が多いのではないでしょうか。
本稿では Lazy relations を使ったときに多くのメモリを消費していることに気付いたので調査方法を含めて整理しておきます。
Node.js/V8 のヒープメモリについて、私の理解が浅いので説明が誤っている箇所もあるかもしれません。ご注意ください。

## TL/DR

TypeORM で Lazy relations を設定すると、設定したプロパティ数に比例して1つの Entity 単位でメモリを消費します。

あるとき、複数のテーブルを結合して数百件から数千件程度のデータを取得した際、
`--max-old-space-size` に 512 MiB が設定された Node.js サーバーで OOM (Out Of Memory) が発生したことで気付きました。

TypeORM の GitHub リポジトリの issue にも複雑な関連をもつとメモリを消費するといった issue が報告されています。

* [Excessive memory usage when loading models with relations #4499](https://github.com/typeorm/typeorm/issues/4499)
  * [Hydration performance issue on complex dataset #2381](https://github.com/typeorm/typeorm/issues/2381) 

少し前に #4499 の issue は #2381 の duplicate として閉じられました。しかし、#2381 の issue ではあまりメモリ使用量については言及されていないため、元の issue のリンクも紹介しておきます。
2-3年前からある issue なので、古いコメントの一部は改善しているかもしれません。issue の内容に目を通しても何が原因なのかよくわかりません。

私が開発に関わっているアプリケーションだと Lazy relations の設定を削除すると、削除前より20-30%程度、メモリ使用量が削減するようにみえました。

## Lazy relations のメモリ使用量を測る

実際にサンプルアプリケーションを作ってメモリの使用量を計測してみます。
本稿で検証するサンプルアプリケーションは次のリポジトリにあります。

* https://github.com/kazamori/typeorm-performance-issues-sample

調査した環境は次になります。

* TypeORM: 0.2.34
* Node.js: v14.17.1
* PostgreSQL 12.7

### 環境構築

次のコマンドで構築できます。

```bash
$ docker-compose up -d
$ yarn dbinit
```

詳細は前記事の [TypeORM の @RelationId デコレーターとパフォーマンス]({{< relref "typeorm-relation-id-performance" >}}) の環境構築を参照してください。

### 検証対象の Entity オブジェクト

`@ManyToMany` と `@ManyToOne` と `@OneToOne` の関連をもつ3つの Entity を用意します (後述) 。

* Post2
* Post3
* Post4

Post1 は前記事の検証用途だったので Post2 から始まっています。

それぞれの Entity の設定と振る舞いの違いがわかりやすいように REPL を使って確認していきます。

```bash
$ yarn repl
... (デバッグ用に SQL を出力しています)
... (Enter を入力するとプロンプトが表示されます)
>
```

#### Post2

categories と attach を Lazy relations で設定します。
オプションで `{lazy: true}` を設定しなくても型として
Promise を指定すると TypeORM は自動的に Lazy relations として扱います。
user はオプションで lazy も eager も指定していないので自動的には取得されません。

```typescript
@Entity()
export class Post2 extends Base<Post2> {
  @PrimaryGeneratedColumn()
  id: number;
  ...
  @ManyToMany((type) => Category, (category) => category.posts2, {
    nullable: true,
    lazy: true,
  })
  @JoinTable()
  categories: Promise<Category[]>;

  @ManyToOne((type) => User, (user) => user.posts2)
  user: User;

  @OneToOne((type) => Attach, (attach) => attach.post2, {
    nullable: true,
    lazy: true,
  })
  @JoinColumn()
  attach: Promise<Attach>
}
```

REPL を使って Entity を取得すると、次の Entity を取得します。

```typescript
> await getConnection().getRepository(Post2).findOne()
Post2 {
  id: 1,
  contents: 'contents-1',
  createdAt: 2021-07-12T00:31:58.718Z,
  updatedAt: 2021-07-12T00:31:58.718Z
}
```

#### Post3

Post2 と比べて、categories と attach を Eager relations として扱います。

```typescript
Entity()
export class Post3 extends Base<Post3> {
  @PrimaryGeneratedColumn()
  id: number;
  ...
  @ManyToMany((type) => Category, (category) => category.posts3, {
    nullable: true,
    eager: true,
  })
  @JoinTable()
  categories: Category[];
  ...
  @OneToOne((type) => Attach, (attach) => attach.post3, {
    nullable: true,
    eager: true,
  })
  @JoinColumn()
  attach: Attach;
}
```

ここで categories として定義している `Category` の定義は次のようにそれぞれの
Post オブジェクトに対する双方向の関連をもち、Lazy relations として設定されています。
Post4 との比較に使うので頭の片隅に入れておいてください。

```typescript
@Entity()
export class Category extends Base<Category> {
  @PrimaryGeneratedColumn()
  id: number;

  @Column("text", { default: "" })
  name: String;

  @ManyToMany((type) => Post1, (post) => post.categories, {
    nullable: true,
    lazy: true,
  })
  posts1: Promise<Post1[]>;

  @ManyToMany((type) => Post2, (post) => post.categories, {
    nullable: true,
    lazy: true,
  })
  posts2: Promise<Post2[]>;

  @ManyToMany((type) => Post3, (post) => post.categories, {
    nullable: true,
    lazy: true,
  })
  posts3: Promise<Post3[]>;
}
```

REPL を使って Entity を取得すると、次の Entity を取得します。

```typescript
Post3 {
  id: 1,
  contents: 'contents-1',
  createdAt: 2021-07-12T00:31:59.497Z,
  updatedAt: 2021-07-12T00:31:59.497Z,
  categories: [
    Category { id: 1, name: 'category-1' },
    Category { id: 2, name: 'category-2' },
    Category { id: 3, name: 'category-3' }
  ],
  attach: Attach { id: 1, attr: 'attr-1' }
}
```

#### Post4

Post3 と同様、categories と attach を Eager relations として扱います。

```typescript
@Entity()
export class Post4 extends Base<Post4> {
  @PrimaryGeneratedColumn()
  id: number;
  ...
  @ManyToMany((type) => Category4, {
    nullable: true,
    eager: true,
  })
  @JoinTable()
  categories: Category4[];
  ...
  @OneToOne((type) => Attach, (attach) => attach.post4, {
    nullable: true,
    eager: true,
  })
  @JoinColumn()
  attach: Attach;
}
```

Post3 との違いとして categories として `Category4` という別の Entity を定義しています。
Category4 は Post オブジェクトに対する関連 (Lazy relations) をもっていないことが違いです。

```typescript
@Entity()
export class Category4 extends Base<Category4> {
  @PrimaryGeneratedColumn()
  id: number;

  @Column("text", { default: "" })
  name: String;
}
```

REPL を使って Entity を取得すると、次の Entity を取得します。
テーブルの構造/データの内容は Post3 と全く同じです。

```typescript
Post4 {
  id: 1,
  contents: 'contents-1',
  createdAt: 2021-07-12T00:32:00.284Z,
  updatedAt: 2021-07-12T00:32:00.284Z,
  categories: [
    Category4 { id: 1, name: 'category-1' },
    Category4 { id: 2, name: 'category-2' },
    Category4 { id: 3, name: 'category-3' }
  ],
  attach: Attach { id: 1, attr: 'attr-1' }
}
```

#### ER 図

整理のために ER 図は次のようになります。

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/memory-profile-tables.png"
           alt="memory profile tables"
           title="メモリ使用量の検証のためのテーブル"
           class="common-style"
           width=768 >}}

Category が Lazy relations をもつときとそうでないときを検証するために Category4 を別に定義しています。
Entity の構造 (設定) は意図的に差異を加えていますが、データベースからみたときのテーブルの構造と関連は基本的に同じ構成となるよう Post2, Post3, Post4 のテーブルを作っています。

この ER 図は [eralchemy](https://github.com/Alexis-benoist/eralchemy) というツールで出力できます。

```bash
$ eralchemy -i 'postgresql+psycopg2://root:password@localhost:15432/test' \
    -o memory-profile-tables.png \
    --include-tables post2 post3 post4 user category category4 attach \
                     post2_categories_category \
                     post3_categories_category \
                     post4_categories_category4
```

### Node.js (V8) のヒープメモリの使用量を測る

Node.js (V8) のヒープのメモリ使用量を測る方法はいくつかあります。

Node.js の [process.memoryUsage()](https://nodejs.org/api/process.html#process_process_memoryusage) API を使う方法や [node-heapdump](https://github.com/bnoordhuis/node-heapdump) のライブラリでヒープのスナップショットを取得する方法も試してみました。

最終的にソースを書き換えながら意図したタイミングでヒープのスナップショットを取得する方法として、
Chrome DevTools を使う方法がもっとも簡単だったのでその方法を紹介します。

次の URI で Chrome のウィンドウを開きます。

```
chrome://inspect/
```

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/chrome-devtools1.png"
           alt="Chrome DevTools inspect 1"
           title="Chrome DevTools inspect の画面1"
           class="common-style"
           width=600 >}}

node コマンドに `--inspect` オプションを指定してアプリケーションを起動します。

```bash
$ node --inspect --require ts-node/register/transpile-only src/memoryProfile.ts
Debugger listening on ws://127.0.0.1:9229/9515df11-4992-481e-aa76-22387262dee9
For help, see: https://nodejs.org/en/docs/inspector
```

Remote Target の下に起動したアプリケーションの情報が表示されます。

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/chrome-devtools2.png"
           alt="Chrome DevTools inspect 2"
           title="Chrome DevTools inspect の画面2"
           class="common-style"
           width=600 >}}

`inspect` というリンクを選択して `Memory` タブを選択すると、次のようなメモリープロファイルの画面が開きます。

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/chrome-devtools3.png"
           alt="Chrome DevTools Memory Profile"
           title="Chrome DevTools の Memory Profile 画面"
           class="common-style"
           width=600 >}}

アプリケーションに接続できていれば `Take snapshot` ボタンを選択すると、
ヒープのスナップショットが取得されます。

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/chrome-devtools4.png"
           alt="Chrome DevTools Heap Snapshot"
           title="Chrome DevTools の Heap Snapshot 画面"
           class="common-style"
           width=600 >}}

Node.js (V8) ではスナップショットを取得する前に GC が実行されます。
スナップショットを取得する瞬間の GC 待ちのメモリも含めたヒープの状態を取得することはできないようにみえます。
またメモリを大量に使って負荷のかかっている状況でスナップショットを取得しようとすると、
私の環境では Segmentation fault が発生して取得できませんでした。

```bash
Segmentation fault (core dumped)
```

いくつか Node.js のバージョンを変えてみたり、
core の中身からエラーが発生している関数を検索したりしてみたのですが、
よくわかりませんでした。

* v14.15.5
* v14.12.0
* v14.5.0

そこで負荷をかけながらヒープのスナップショットを取得することは諦めて、
確認したい TypeORM の Entity をグローバルに保持するようにして調査を進めました。

本稿では紹介しませんが、
Node.js のメモリに関する調査をするときに便利なオプションをいくつか紹介しておきます。

* [`--max-old-space-size`](https://nodejs.org/api/cli.html#cli_max_old_space_size_size_in_megabytes):
  * ヒープメモリの old section のサイズを MiB で指定する
  * V8 が管理するヒープメモリはいくつか種類がある。そのうちの長時間メモリが保持され、メジャー GC によって管理される領域になる
  * 一般的にヒープメモリのサイズを指定するときはこのサイズを調整する
* [`--abort-on-uncaught-exception`](https://nodejs.org/api/cli.html#cli_abort_on_uncaught_exception):
  * Node.js が異常終了したときに core ファイルを出力する
* `--expose-gc`:
  * プログラム内から `global.gc();` を呼び出すことで任意のタイミングでメジャー GC を発生させられる
* `--trace-gc`:
  * GC ログを出力する
  * アプリケーションを実際に動かしながらヒープメモリの使用量の遷移や GC の実行状況を観察できる

### サンプルアプリケーションの Entity のメモリ使用量を測る

Chrome DevTools を使って実際にサンプルアプリケーションの Entity サイズをみていきます。次のようにしてサーバーアプリケーションを起動します。

```bash
$ yarn memoryProfile
```

意図したタイミングでヒープのスナップショットを取得できれば、サーバーアプリケーションである必要はありません。
私がヒープのスナップショットを取得するプログラムをうまく実装できなかったため、
サーバーアプリケーションにして Chrome DevTools からリモートデバッグできるようにしています。

```typescript
var post2: Post2[];
var post3: Post3[];
var post4: Post4[];

async function loadPosts() {
  const connection = getConnection();
  const post2Repo = connection.getRepository(Post2);
  const post3Repo = connection.getRepository(Post3);
  const post4Repo = connection.getRepository(Post4);

  post2 = await post2Repo.find({ take: 100 });
  console.log("post2", post2[0]);
  post3 = await post3Repo.find({ take: 100 });
  console.log("post3", post3[0]);
  post4 = await post4Repo.find({ take: 100 });
  console.log("post4", post4[0]);

  console.log("success loading posts");
}

export async function main() {
  await connect(false);
  await loadPosts();
  const server = http.createServer(async (req, res) => {
    res.writeHead(200);
    res.end("OK");
  });
  server.listen({ host: "localhost", port: 18080 });
}
```

起動後に Post2, Post3, Post4 をグローバルの領域に保持してスナップショットに現れるようにしています。
もっとよいやり方があると思いますが、私がわからなかったのでこんなやり方になっています。

前節で紹介したように Chrome DevTools で接続してスナップショットを取得します。

Summary の横にある Class filter で "Post" と入力すると、
グローバルに保持しておいた Post2, Post3, Post4 の Entity がフィルターされます。

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/chrome-devtools5.png"
           alt="Chrome DevTools Post Entity size"
           title="Chrome DevTools Post Entity のサイズ"
           class="common-style"
           width=600 >}}

次の2つのサイズがあります。

* **Shallow Size**: そのオブジェクトが直接保持しているメモリのサイズ
* **Retained Size**: GC によって解放されるメモリのサイズ (依存しているオブジェクトのサイズなども含まれる)

スナップショットをみると Shallow Size より Retained Size が小さくなることはないため、
メモリの使用量を考慮するときは Retained Size をみておくとよい気がします。
Retained Size をどうやって算出しているのか、
自分で計算しようと試みたのですが、よくわかりませんでした。

### TypeORM の Entity のメモリ使用量についての考察

いまそれぞれの Post の Entity を100件ずつ保持しています。

Retained Size を比較すると、
Post3 が 631KiB と最も大きく、次に Post2 が 182KiB、Post4 が 79KiB となっています。

まず Post2 と Post4 を比較してみましょう。

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/heap-snapshot-post2.png"
           alt="Heap Snapshot Post2 Detail"
           title="ヒープスナップショット Post2 の詳細"
           class="common-style"
           width=800 >}}

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/heap-snapshot-post4.png"
           alt="Heap Snapshot Post4 Detail"
           title="ヒープスナップショット Post4 の詳細"
           class="common-style"
           width=800 >}}

Post2 と Post4 の違いは Lazy と Eager による関連する Entity の取得タイミングの違いです。
Shallow Size は Post4 の方が大きくなっているのは
Eager loading によって関連する Entity を取得しているからだと推測します。

Shallow Size と Retained Size の数字は単純にビューをドリルダウンしていった数値の合計とは一致しないので特別な計算方法があるようにみえます。
但し、どの要素の Retained Size が大きいかをみていくと、なんとなくメモリを消費しているところを推測できるかもしれません。

TypeORM では Lazy relation の機能を提供するために [RelationLoader](https://github.com/typeorm/typeorm/blob/0.2.34/src/query-builder/RelationLoader.ts#L186) を使います。
`Object.defineProperty()` で Entity ごとに setter/getter を設定しています。
クロージャで定義しているので環境情報を含めてメモリが消費されているのかなと推測されます。

次に最もメモリ使用量の大きかった Post3 の詳細をみていきます。

{{< figure src="/blogs/2021/images/typeorm-lazy-relations/heap-snapshot-post3.png"
           alt="Heap Snapshot Post3 Detail"
           title="ヒープスナップショット Post3 の詳細"
           class="common-style"
           width=800 >}}

Post 3 は Post4 と同様、categroies を Eager loading で取得します。
しかし、Post3 の categroies のそれぞれの要素の Retained Size は Post4 のそれらとは大きくサイズが異なります。
意図的に Lazy relations をもつ `Category` (**しかも3つ！**) と、それをもたない `Category4` の違いです。

ヒープのスナップショットの Retained Size のみをみる限り、
取得した Entity の Lazy relations が多いほど、
Retained Size のサイズが大きくなる傾向があることがわかりました。

本稿の冒頭で紹介した issue では複雑な関連をもつ Entity を取得すると、
メモリを大きく消費するといった内容がいくつも報告されていました。
この調査結果からもわかるように関連する Entity を一緒に取得すると、
それらの Entity に設定されている Lazy relations に対して `RelationLoader` が設定され、
その数に比例してメモリの消費量が増えていきます。

`RelationLoader` がすべてではないかもしれませんが、
いくらかメモリ消費に関連しているのではないかと本稿では推測しています。

## まとめ

TypeORM の Lazy relations も必須の機能ではありません。

少しでもメモリの消費量を抑えたいなら使わないようにすればよいです。
しかし、そうすると同時に ORM を使っている利便性も失われていくので悩ましい問題になるかもしれません。

## リファレンス

Node.js (V8) のメモリの使用量を調査するときに参考になった記事をまとめておきます。

* [🚀 Visualizing memory management in V8 Engine (JavaScript, NodeJS, Deno, WebAssembly)](https://deepu.tech/memory-management-in-v8/)
* [Documentation / Chrome DevTools / Memory / Record heap snapshots](https://developer.chrome.com/docs/devtools/memory-problems/heap-snapshots/)
* [Documentation / Chrome DevTools / Memory / Memory terminology](https://developer.chrome.com/docs/devtools/memory-problems/memory-101/#object-sizes)
* [Retained Size in Chrome memory snapshot - what exactly is being retained?](https://stackoverflow.com/questions/62049063/retained-size-in-chrome-memory-snapshot-what-exactly-is-being-retained)
* [Fast properties in V8](https://v8.dev/blog/fast-properties)
* [V8のHidden Classの話](https://engineering.linecorp.com/ja/blog/v8-hidden-class/)

TypeORM の Lazy relations は Hidden Class が新たに生成されているわけではありませんが、ヒープのスナップショットを調査しているときに参考になったので一緒に紹介しておきます。
