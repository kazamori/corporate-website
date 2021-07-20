---
title: "Node.js の REPL 環境をカスタマイズする"
slug: customize-node-repl
date: 2021-07-20T14:18:39+09:00
with_date: true
tags: [node.js, repl, typeorm]
---

前記事の [Typeorm の Distinct を伴うクエリとパフォーマンス]({{< relref "typeorm-distinct-performance" >}}) で REPL 環境を使って検証しました。


本稿では REPL 環境の作成方法について整理しておきます。

[Node.js](https://nodejs.org/en/) 向けの TypeScript の実行環境として [ts-node](https://github.com/TypeStrong/ts-node) があります。他にもいくつかツールはあるそうですが、ここでは ts-node を使って TypeScript のソースコードを JavaScript に変換して Node.js で実行する REPL 環境を構築します。

## 環境構築

実際に動作する REPL 環境は次のリポジトリにあります。

* https://github.com/kazamori/typeorm-performance-issues-sample

リポジトリをクローンしてデータベース環境と初期データを生成します。
REPL の動作確認するためだけなので中身はとくに重要ではありません。

```bash
$ git clone https://github.com/kazamori/typeorm-performance-issues-sample.git
$ cd typeorm-performance-issues-sample
$ docker-compose up -d
$ yarn install
$ yarn dbinit
```

## REPL  の起動

```bash
$ yarn repl
... (デバッグ用に SQL を出力しています)
... (Enter を入力するとプロンプトが表示されます)
>
```

SQL がずらずらと出力され、Enter キーを入力するとプロンプトが表示されます。

ここで package.json には repl コマンドを次のように指定しています。

```bash
node --require ts-node/register/transpile-only --experimental-repl-await src/repl
```

* [`--require`](https://nodejs.org/api/cli.html#cli_r_require_module): node の起動時にモジュールを読み込む
  * `ts-node/register/transpile-only`: ts-node の機能を使って TypeScript の変換のみを行う、型チェックは行わない
* [`--experimental-repl-await`](https://nodejs.org/api/cli.html#cli_experimental_repl_await): REPL 環境でトップレベルの `await` を有効にする

## Node.js の repl モジュール

Node.js は repl モジュールを提供していて、`repl.start()` すると [REPLServer](https://nodejs.org/api/repl.html#repl_class_replserver) のインスタンスを返します。基本的にはこれだけで REPL 環境が起動します。

ちょっとしたカスタマイズとしてプロンプトを変更したいときは次のようにオプションを渡します。

```typescript
import repl from "repl";
    
const replServer = repl.start({prompt: "myapp > "});
```

```bash
$ yarn repl
myapp >
```

## プロジェクトごとの設定をする

例えば、typeorm モジュールが提供する API をグローバルに利用したいとします。
開発用途に限定して使う REPL 環境であれば、
個別に指定するのは面倒なのでまとめてインポートして、
すべてグローバルの名前空間にセットするといった荒技でもよいと思います。

```typescript
import * as typeorm from "typeorm";

Object.entries(typeorm).map(([key, value]) => {
    // @ts-ignore
    globalThis[key] = value;
});
```

[VS Code](https://code.visualstudio.com/) で lintチェックのワーニングが出たら `@ts-ignore` を指定すると無視できます。

完全なカスタマイズのソースは [src/repl.ts](https://github.com/kazamori/typeorm-performance-issues-sample/blob/main/src/repl.ts) を確認してください。

## REPL の振る舞い

例えば、`get` と入力してタブキーを入力すると補完された get で始まる関数が列挙されます。

```bash
myapp > get
getConnection           getConnectionManager    getConnectionOptions
getCustomRepository     getFromContainer        getManager
getMetadataArgsStorage  getMongoManager         getMongoRepository
getRepository           getSqljsManager         getTreeRepository
```

TypeORM が提供する `getConnection()` や `getManager()` のカッコまで入力するとレスポンスの型情報なども表示されるようです。

```bash
myapp > getConnection()
<ref *1> Connection { migrations: [], subscribers: [], entityMetadatas: [ [EntityMetadata], [EntityMetadata], [EntityMetadata], [EntityMetadata], [EntityMetadata], [EntityMetadata], [EntityMetadata], [Entit...
```

```bash
myapp > getManager()
<ref *1> EntityManager { repositories: [], plainObjectToEntityTransformer: PlainObjectToNewEntityTransformer {}, connection: Connection { migrations: [], subscribers: [], entityMetadatas: [Array], name: 'de...
```

実際の用途として、次のコードを実行したときにログとして SQL を確認しつつ、意図した返り値として10000が返っているのを確認できます。

```bash
myapp > await getConnection().getRepository(Article).count()
query: SELECT COUNT(1) AS "cnt" FROM "article" "Article"
10000
```

## まとめ

Node.js が提供する repl モジュールを使ってプロジェクト独自の REPL 環境を構築しました。

例として [TypeORM](https://github.com/typeorm/typeorm) のような O/R マッパーの振る舞いを確認するのを紹介しました。
プロジェクトごとの設定やモジュール読み込みなどを自動化しておけば、
ちょっとしたコードを実行したりデバッグしたりするときに役に立つでしょう。

## リファレンス

* [Rails like console for Node JS (and Node TS)](https://sapandiwakar.in/rails-like-console-for-node-js-and-node-ts-2/)
* [Node.jsエコシステムを体験しよう](https://future-architect.github.io/typescript-guide/ecosystem.html)
