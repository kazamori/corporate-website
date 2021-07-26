---
title: "TypeORM の Bulk Insert と psql の \\copy を比較する"
slug: typeorm-bulk-insert
date: 2021-07-21T17:29:08+09:00
with_date: true
tags: [typeorm, postgres, performance]
---

テストデータなどをバッチ処理で作成するときに
[TypeORM](https://github.com/typeorm/typeorm) の機能で行うのと、
PostgreSQL 付属の psql コマンドを使って行うときでどのぐらいの実行時間の差異があるかを検証します。

## TL/DR

関連をもたない1つのテーブルで10万行を作成するときの実行時間は次のようなりました。

* TypeORM の [Repository API](https://github.com/typeorm/typeorm/blob/master/docs/repository-api.md) または [EntityManager API](https://github.com/typeorm/typeorm/blob/master/docs/entity-manager-api.md): 約12秒
* psql の \\copy コマンド: 約2秒

psql コマンドを使うときは CSV ファイルを生成しないといけないため、
TypeORM と比べて要件に対する柔軟性やプログラミングの自由度は減ります。

しかし、要件を満たせる場合は実行時間が約6倍ぐらいは速く完了する見通しになります。

## 検証に使うサンプルアプリケーション

実際にサンプルアプリケーションを作って振る舞いを確認します。
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
$ yarn install
$ yarn dbsync
```

dbsync を実行すると既存のテーブルを削除してから再作成します。
データベースの環境を初期化してテストを何度もやり直すときなどに便利です。

psql コマンドで接続してテーブルが作成されていることやデータがあることを確認してみましょう。

```bash
$ docker-compose exec postgres psql -h localhost -U root test
test=# \d
... (いくつかテーブル情報が出力される)
```

## TypeORM で Insert する

私が調べた範囲では、3通りのやり方があるようにみえます。
そこまで深く調べていないのでもっとあるかもしれません。

* [Repository APIs](https://github.com/typeorm/typeorm/blob/master/docs/repository-api.md)
  * Repository.save というメソッドを使う (関連データを一緒に保存できない)
* [EntityManager API](https://github.com/typeorm/typeorm/blob/master/docs/entity-manager-api.md)
  * EntityManager.save というメソッドを使う (関連データも一緒に保存できる)
* [Insert using Query Builder](https://github.com/typeorm/typeorm/blob/master/docs/insert-query-builder.md)
  * Query Builder を使う、最も低レベルの API になる (本稿では扱わない)

psql コマンドの違いを測る上で、シンプルな関連をもたないテストデータの方が検証しやすかったので
Repository.save と EntityManager.save も一緒に測りました。
同じような SQL の Insert 文が発行されるので実行時間に大きな違いはないようにみえました。
本稿では Repository.save のみを取り上げます。

[前記事]({{< relref "customize-node-repl" >}}) で紹介した REPL 環境を使って検証していきましょう。

いま `Article` という Entity が定義されています。
`Article` の定義は一旦置いておいて、最低限の入力のみを与えてデータを作成してみます。

### 単一の Entity を生成する

Repository.save() を使います。

```typescript
> await getConnection().getRepository(Article).save({wordCount: 10, readMinutes: 3})
query: START TRANSACTION
query: INSERT INTO "article"("wordCount", "readMinutes", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt", "userId", "attachId") VALUES ($1, $2, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT) RETURNING "id", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt" -- PARAMETERS: [10,3]
query: COMMIT
{
  wordCount: 10,
  readMinutes: 3,
  id: 1,
  text1: '',
  text2: '',
  text3: '',
  text4: '',
  text5: '',
  text6: '',
  text7: '',
  text8: '',
  createdAt: 2021-07-21T00:58:27.273Z,
  updatedAt: 2021-07-21T00:58:27.273Z
}
```

psql で実際にデータが作成されているかを確認するのもよいでしょう。

```sql
test=# select * from article;
 id | wordCount | readMinutes | text1 | text2 | text3 | text4 | text5 | text6 | text7 | text8 |         createdAt          |         updatedAt          | userId | attachId 
----+-----------+-------------+-------+-------+-------+-------+-------+-------+-------+-------+----------------------------+----------------------------+--------+----------
  1 |        10 |           3 |       |       |       |       |       |       |       |       | 2021-07-21 09:58:27.273846 | 2021-07-21 09:58:27.273846 |        |         
```

Repository.save() は array で複数データを渡すこともできます。

```typescript
> await getConnection().getRepository(Article).save([{wordCount: 11, readMinutes: 4}, {wordCount: 12, readMinutes: 5}])
query: START TRANSACTION
query: INSERT INTO "article"("wordCount", "readMinutes", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt", "userId", "attachId") VALUES ($1, $2, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT), ($3, $4, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT) RETURNING "id", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt" -- PARAMETERS: [11,4,12,5]
query: COMMIT
```

カラムが多くて見づらいので省略して書きます。
複数行をまとめて挿入する INSERT 文が発行されていることがわかります。

```sql
INSERT INTO "article"(...) VALUES ($1, $2, ...), ($3, $4, ...) RETURNING ...
```

### 関連をもつ Entity を生成する

まず `@ManyToMany` の `Category` の Entity を作ります。

```typescript
> categories = await getConnection().getRepository(Category).save([{name: "cate1"}, {name: "cate2"}, {name: "cate3"}])
query: START TRANSACTION
query: INSERT INTO "category"("name") VALUES ($1), ($2), ($3) RETURNING "id", "name" -- PARAMETERS: ["cate1","cate2","cate3"]
query: COMMIT
[
  { name: 'cate1', id: 1 },
  { name: 'cate2', id: 2 },
  { name: 'cate3', id: 3 }
]
```

次に作成した categories を使って `Article` の Entity を作ってみます。

```typescript
> await getConnection().getRepository(Article).save({wordCount: 13, readMinutes: 6, categories: categories})
query: START TRANSACTION
query: INSERT INTO "article"("wordCount", "readMinutes", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt", "userId", "attachId") VALUES ($1, $2, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT) RETURNING "id", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt" -- PARAMETERS: [13,6]
query: COMMIT
```

INSERT 文 をみた感じだと `@ManyToMany` の関連は生成されていないようにみえます。

実際にも生成されていません。

```sql
test=# select * from article_categories_category;
 articleId | categoryId
-----------+------------
(0 行)
```

[Saving many-to-many relations](https://github.com/typeorm/typeorm/blob/master/docs/many-to-many-relations.md#saving-many-to-many-relations) によると、[cascades](https://github.com/typeorm/typeorm/blob/master/docs/relations.md#cascades) というオプションが必要だそうです。

`Article` の categories の `@ManyToMany` アノテーションに `cascade` というオプションを追加します。

```typescript
  @ManyToMany((type) => Category, {
    nullable: true,
    lazy: true,
    cascade: true,
  })
  @JoinTable()
  categories: Promise<Category[]>;
```

REPL を再起動して、先ほどと同じように実行してみましたが、`@ManyToMany` の関連は生成されませんでした。

TypeORM のドキュメントのサンプルコードをよく見返すと、[EntityManager API](https://github.com/typeorm/typeorm/blob/master/docs/entity-manager-api.md) を使っています。

少しコードを書き換えて次のように実行してみます。

```typescript
> await getConnection().manager.save(new Article({wordCount: 13, readMinutes: 6, categories: categories}))
query: SELECT "Category"."id" AS "Category_id", "Category"."name" AS "Category_name" FROM "category" "Category" WHERE "Category"."id" IN ($1, $2, $3) -- PARAMETERS: [1,2,3]
query: START TRANSACTION
query: INSERT INTO "article"("wordCount", "readMinutes", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt", "userId", "attachId") VALUES ($1, $2, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT) RETURNING "id", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt" -- PARAMETERS: [13,6]
query: INSERT INTO "article_categories_category"("articleId", "categoryId") VALUES ($1, $2), ($3, $4), ($5, $6) -- PARAMETERS: [6,1,6,2,6,3]
query: COMMIT
```

意図したように `@ManyToMany` の関連の INSERT 文が生成されました。

```sql
INSERT INTO "article_categories_category"("articleId", "categoryId") VALUES ($1, $2), ($3, $4), ($5, $6)
```

```sql
test=# select * from article_categories_category;
 articleId | categoryId 
-----------+------------
         6 |          1
         6 |          2
         6 |          3
(3 行)
```

余談ですが、この例からもわかるように、TypeORM を使うときの難しさの1つとして、
パラメーター渡しができてしまって特にエラーは発生しないため、
期待した振る舞いになるのかどうかを実際に実行してみないとわからないことがちょくちょくあります。

REPL などで実際に実行して SQL やデータベースのデータを確認しながら開発を進めるのがよいでしょう。

他の関連についてもみてみましょう。

`@ManyToOne` の `User` の生成をやってみます。まずは `User` の Entity を作ります。

```typescript
> user1 = await getConnection().getRepository(User).save({name: "user1"})
query: START TRANSACTION
query: INSERT INTO "user"("name") VALUES ($1) RETURNING "id", "name" -- PARAMETERS: ["user1"]
query: COMMIT
{ name: 'user1', id: 1 }
```

同様に user1 を使って `Article` のインスタンスを作ってみます。

```typescript
> await getConnection().getRepository(Article).save({wordCount: 14, readMinutes: 7, user: user1})
query: START TRANSACTION
query: INSERT INTO "article"("wordCount", "readMinutes", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt", "userId", "attachId") VALUES ($1, $2, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT) RETURNING "id", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt" -- PARAMETERS: [14,7]
query: COMMIT
```

やはり `userId` に意図した user1 の id がセットされないようです。

EntityManager を使ったやり方も試してみます。

エラーが発生しますが、`userId` にパラメーターを渡そうしていることはわかります。

```typescript
> await getConnection().manager.save(new Article({wordCount: 14, readMinutes: 7, user: user1}))
query failed: INSERT INTO "article"("wordCount", "readMinutes", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt", "userId", "attachId") VALUES ($1, $2, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, $3, DEFAULT) RETURNING "id", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt" -- PARAMETERS: [14,7,{"name":"user1","id":1}]
```

次のように `user: user1.id` というパラメーターの渡し方に変更します。

```typescript
> await getConnection().manager.save(new Article({wordCount: 14, readMinutes: 7, user: user1.id}))
query: START TRANSACTION
query: INSERT INTO "article"("wordCount", "readMinutes", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt", "userId", "attachId") VALUES ($1, $2, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, $3, DEFAULT) RETURNING "id", "text1", "text2", "text3", "text4", "text5", "text6", "text7", "text8", "createdAt", "updatedAt" -- PARAMETERS: [14,7,1]
query: COMMIT
```

この方法だと、意図したように INSERT 文が発行されてデータも生成されます。

```sql
test=# select * from article;
 id | wordCount | readMinutes | text1 | text2 | text3 | text4 | text5 | text6 | text7 | text8 |         createdAt          |         updatedAt          | userId | attachId
----+-----------+-------------+-------+-------+-------+-------+-------+-------+-------+-------+----------------------------+----------------------------+--------+----------
...
 12 |        14 |           7 |       |       |       |       |       |       |       |       | 2021-07-21 11:19:15.712291 | 2021-07-21 11:19:15.712291 |      1 |         
```

TypeORM の Repository API と EntityManager API の違いがなんとなくわかってきたでしょうか。

条件を揃えるのがややこしくなるので本稿では関連をもたない単一の
Article のみを生成するときの実行時間を測ることにしました。

##  Repository API で Bulk Insert する

次のようにして 1000件ずつ100回にわけて10万件の Article のデータを生成します。

```typescript
test("Bulk create 1000 articles 100 times", async () => {
  const start = new Date();
  let i = 0;
  while (i < 100) {
    await articleRepo.save(createArticles(1000));
    i++;
  }
  const end = new Date();
  console.log(getElapsedTime(start, end));
});
```

次のように実行するとミリ秒で実行時間が出力されます。
私の環境では11秒台の数字になります。

```bash
$ yarn test --testNamePattern "Bulk create 1000 articles 100 times"
...
    console.log
      11451
...
```

ちなみに1万件ずつやろうとすると、次のようなエラーが発生しました。

```bash
$ yarn test --testNamePattern "Bulk create 10000 articles"

    QueryFailedError: bind message has 34464 parameter formats but 0 parameters

      at new QueryFailedError (src/error/QueryFailedError.ts:11:9)
      at PostgresQueryRunner.<anonymous> (src/driver/postgres/PostgresQueryRunner.ts:228:19)
      at step (node_modules/tslib/tslib.js:143:27)
      at Object.throw (node_modules/tslib/tslib.js:124:57)
      at rejected (node_modules/tslib/tslib.js:115:69)
```

あまりバインド変数が多過ぎるとどこかでしきい値を超えてしまうようです。

## PostgreSQL の COPY コマンドと psql の `\copy` の違い

PostgreSQL がサポートする SQL コマンドに **COPY** があります。これは [psql](https://www.postgresql.jp/document/12/html/app-psql.html) の `\copy` とは似て非なるものです。ドキュメントから引用します。

> COPYをpsqlの\copyと混同しないでください。
> 
> \copyはCOPY FROM STDINやCOPY TO STDOUTを呼び出し、
> psqlクライアントからアクセスできるファイルにデータの書き込み/読み込みを行います。
> したがって、\copyコマンドでは、ファイルへのアクセスが可能かどうかと、
> ファイルに対するアクセス権限の有無は、サーバではなくクライアント側に依存します。
> 
> https://www.postgresql.jp/document/12/html/sql-copy.html

つまり、取り込みたいデータファイルへのアクセスをサーバー側から行うのか、クライアント側から行うのかの違いになります。
使い分けは PostgreSQL サーバーの運用次第になります。
例えば、Amazon RDS のようなマネージドサービスを使うときは物理的なサーバー環境にアクセスできないため、
必然的にクライアント側で行う `\copy` しか選択肢がありません。

Amazon RDS の PostgreSQL のユーザーガイドにおいても psql の `\copy` の使い方が説明されています。

* [Using the \copy command to import data to a table on a PostgreSQL DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Procedural.Importing.html#PostgreSQL.Procedural.Importing.Copy)

## psql の \\copy コマンドを使う

psql コマンドを実行するシェルスクリプトから実行します。
date コマンドで測っているので厳密ではありませんが、
私の環境では10万件で2秒ほどになります。

```bash
$ bash tests/bulk/psql.sh
2021年  7月 26日 月曜日 12:17:40 JST
COPY 100000
2021年  7月 26日 月曜日 12:17:42 JST
```

カレントディレクトリに次のような CSV が作成され、それを psql コマンドで取り込みます。

```bash
$ head -n 3 article.csv
100,10,text1-0,text2-0,text3-0,text4-0,text5-0,text6-0,text7-0,text8-0
100,10,text1-1,text2-1,text3-1,text4-1,text5-1,text6-1,text7-1,text8-1
100,10,text1-2,text2-2,text3-2,text4-2,text5-2,text6-2,text7-2,text8-2
```

## まとめ

SQL の INSERT 文で取り込むより、
RDBMS が提供する専用のデータ取り込み機能を使った方が速くなることは容易に推測できます。
それが実際にどのぐらいかを比較しました。

少ない件数であれば複雑な要件やプログラミングの自由度が高い TypeORM で行うメリットもあると思います。
単純に大量のデータがあればいいといったときであれば psql コマンドを使った方が環境構築を迅速に行えるのでよいでしょう。
