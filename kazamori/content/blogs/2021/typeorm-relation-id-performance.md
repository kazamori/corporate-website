---
title: "TypeORM の @RelationId デコレーターとパフォーマンス"
slug: typeorm-relation-id-performance
date: 2021-07-12T09:26:18+09:00
with_date: true
tags: [typeorm, orm, performance]
---

[TypeORM](https://github.com/typeorm/typeorm) に [@RelationId](https://github.com/typeorm/typeorm/blob/master/docs/decorator-reference.md#relationid) というデコレーターがあります。

これは関連テーブルの主キー (外部キー) へのアクセサーを提供し、Entity オブジェクトを取得するときに自動的にプロパティとしてセットしてくれます。
アプリケーションの開発において、あれば便利といった機能の位置付けだと思います。

## TL/DR

複数件の Entity オブジェクトを取得するリスト系の用途に使うなら `@RelationId` の機能は使わない方がよいです。

* [O(n^2) operation performed when RelationId annotation is used #3507](https://github.com/typeorm/typeorm/issues/3507)

TypeORM の GitHub リポジトリの issue にもパフォーマンスの問題が登録されています。

issue 登録されていたものの、私はこの機能がボトルネックになっていると当初わからなかったため、
自分でデバッグしていてボトルネックを見つけてからこの issue に気付きました。
知っていないと、開発時にこの問題には気付きにくいと思います。

issue のタイトルにもある通り、
`@RelationId` を使うと取得件数に対して **O(n^2)** の計算量を要求します。
経験則では、ライブラリで O(n^2) の計算量を要求する機能をあまりみたことがありません。
ドキュメントにもパフォーマンスについての注意などもないため、(便利なので)安易に使ってしまいやすいかもしれません。

特に Node.js サーバーをシングルスレッドで運用する場合、
O(n^2) のような計算量を要求するような処理は、
たった1つのリクエストが他のリクエストを阻害してしまうので致命的な不具合となりえます。

TypeORM の作者も次のようにコメントしています。

> @RelationId functionality is complex.. It's subject of rework in next versions of typeorm. And its supposed to be used in simple cases (since it can fetch a lot otherwise)
> https://github.com/typeorm/typeorm/issues/3507#issuecomment-457468879
> 
> (意訳) `@RelationId` の機能は複雑です。TypeORM の次バージョンで作り直します。(大量にデータを取得するので)シンプルな用途のみを想定しています。

1つの `@RelationId` で O(n^2) になるので、例えば、
あるテーブルが5つの `@RelationId` をもっていると 5 * O(n^2) になります。

私が開発に関わっているアプリケーションでパフォーマンス検証した限りでは、
数百件程度 (n^2 で数万件のオーダー) までは大きな影響はありませんでした。
但し、千件を超えると秒単位でのレイテンシの差が出てきて、
それより大きくなると指数関数的にレイテンシの差が広がっていきました。

## @RelationID のパフォーマンス検証

実際にサンプルアプリケーションを作ってパフォーマンスを検証してみます。
本稿で検証するサンプルアプリケーションは次のリポジトリにあります。

* https://github.com/kazamori/typeorm-performance-issues-sample

調査した環境は次になります。

* TypeORM: 0.2.34
* Node.js: v14.17.1
* PostgreSQL 12.7

### 環境構築

docker-compose を使って PostgreSQL のデータベースを作ります。

```bash
$ docker-compose up -d
```

次にテーブル作成とテストのための初期データを作成します。

```bash
$ yarn dbinit
initialize tables and create data
...
```

psql コマンドで接続してテーブルが作成されていることやデータがあることを確認してみましょう。

```bash
$ docker-compose exec postgres psql -h localhost -U root test

test=# \d post1
                                        Table "public.post1"
  Column   |            Type             | Collation | Nullable |              Default
-----------+-----------------------------+-----------+----------+-----------------------------------
 id        | integer                     |           | not null | nextval('post1_id_seq'::regclass)
 contents  | text                        |           | not null | ''::text
 createdAt | timestamp without time zone |           | not null | now()
 updatedAt | timestamp without time zone |           | not null | now()
 userId    | integer                     |           |          |
 attachId  | integer                     |           |          |
Indexes:
    "PK_5d96af62243b5566110655681a7" PRIMARY KEY, btree (id)
    "REL_391744cd3e041310c3d92fc246" UNIQUE CONSTRAINT, btree ("attachId")
Foreign-key constraints:
    "FK_1d098299fa1236d2895e76559f3" FOREIGN KEY ("userId") REFERENCES "user"(id)
    "FK_391744cd3e041310c3d92fc2464" FOREIGN KEY ("attachId") REFERENCES attach(id)
Referenced by:
    TABLE "post1_categories_category" CONSTRAINT "FK_659193a73a9bae5acd033a83562" FOREIGN KEY ("post1Id") REFERENCES post1(id) ON UPDATE CASCADE ON DELETE CASCADE

test=# select * from post1 limit 3;
 id |  contents  |         createdAt          |         updatedAt          | userId | attachId 
----+------------+----------------------------+----------------------------+--------+----------
  1 | contents-1 | 2021-07-12 01:42:12.628818 | 2021-07-12 01:42:12.628818 |      1 |        1
  2 | contents-2 | 2021-07-12 01:42:12.628818 | 2021-07-12 01:42:12.628818 |      2 |        2
  3 | contents-3 | 2021-07-12 01:42:12.628818 | 2021-07-12 01:42:12.628818 |      3 |        3
(3 rows)

test=# select count(*) from post1;
 count 
-------
 10000
(1 row)
```

### 検証対象の Entity オブジェクト

`@ManyToMany` と `@ManyToOne` の関連をもつ次のような3つの Entity を用意します。

#### Post1

categoryIds と userId に `@RelationId` によるプロパティをもちます。
また categories と attach は [Lazy relations](https://github.com/typeorm/typeorm/blob/master/docs/eager-and-lazy-relations.md#lazy-relations) で取得するようにしています。

```typescript
@Entity()
export class Post1 extends Base<Post1> {
  @PrimaryGeneratedColumn()
  id: number;
  ...
  @ManyToMany((type) => Category, (category) => category.posts1, {
    nullable: true,
    lazy: true,
  })
  @JoinTable()
  categories: Promise<Category[]>;

  @RelationId((post: Post1) => post.categories)
  categoryIds: number[];

  @ManyToOne((type) => User, (user) => user.posts1)
  user: User;

  @RelationId((post: Post1) => post.user)
  userId: number;

  @OneToOne((type) => Attach, (attach) => attach.post1, {
    nullable: true,
    lazy: true,
  })
  @JoinColumn()
  attach: Promise<Attach>
}
```

それぞれの Entity の違いがわかりやすいように REPL を使って振る舞いを確認してみます。

```bash
$ yarn repl
... (デバッグ用に SQL を出力しています)
... (Enter を入力するとプロンプトが表示されます)
>
```

post1 の Entity を取得したタイミングで `@RelationId` の機能により categoryIds と userId をプロパティに保持していることがわかります。

```typescript
> const post1 = await getConnection().getRepository(Post1).findOne()
query: SELECT "Post1"."id" AS "Post1_id", "Post1"."contents" AS "Post1_contents", "Post1"."createdAt" AS "Post1_createdAt", "Post1"."updatedAt" AS "Post1_updatedAt", "Post1"."userId" AS "Post1_userId", "Post1"."attachId" AS "Post1_attachId" FROM "post1" "Post1" LIMIT 1
query: SELECT "Post1_categories_rid"."post1Id" AS "post1Id", "Post1_categories_rid"."categoryId" AS "categoryId" FROM "category" "category" INNER JOIN "post1_categories_category" "Post1_categories_rid" ON ("Post1_categories_rid"."post1Id" = $1 AND "Post1_categories_rid"."categoryId" = "category"."id") ORDER BY "Post1_categories_rid"."categoryId" ASC, "Post1_categories_rid"."post1Id" ASC -- PARAMETERS: [1]

> post1
Post1 {
  id: 1,
  contents: 'contents-1',
  createdAt: 2021-07-11T16:42:12.628Z,
  updatedAt: 2021-07-11T16:42:12.628Z,
  categoryIds: [ 1, 2, 3 ],
  userId: 1
}
```

Lazy relations として設定した categories と attach は await を使って参照したときに SQL が発行されてそれぞれの Entity が生成されていることがわかります。categories は1つの post に対してすべて3件ずつ紐付いています。

```typescript
> await post1.categories
query: SELECT "categories"."id" AS "categories_id", "categories"."name" AS "categories_name" FROM "category" "categories" INNER JOIN "post1_categories_category" "post1_categories_category" ON "post1_categories_category"."post1Id" IN ($1) AND "post1_categories_category"."categoryId"="categories"."id" -- PARAMETERS: [1]

[
  Category { id: 1, name: 'category-1' },
  Category { id: 2, name: 'category-2' },
  Category { id: 3, name: 'category-3' }
]

> await post1.attach
query: SELECT "attach"."id" AS "attach_id", "attach"."attr" AS "attach_attr" FROM "attach" "attach" INNER JOIN "post1" "Post1" ON "Post1"."attachId" = "attach"."id" WHERE "Post1"."id" IN ($1) -- PARAMETERS: [1]
query: SELECT "Post1_categories_rid"."post1Id" AS "post1Id", "Post1_categories_rid"."categoryId" AS "categoryId" FROM "category" "category" INNER JOIN "post1_categories_category" "Post1_categories_rid" ON ("Post1_categories_rid"."post1Id" = $1 AND "Post1_categories_rid"."categoryId" = "category"."id") ORDER BY "Post1_categories_rid"."categoryId" ASC, "Post1_categories_rid"."post1Id" ASC -- PARAMETERS: [null]

Attach { id: 1, attr: 'attr-1' }
```

attach を取得するときに categries に関する SQL が発行されているのは意図したものではありませんが、ここでは無視しておきます。

#### Post2

Post1 と比べて `@RelationId` をもたない Entity を作成します。

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

同様に REPL を使って Entity を取得します。
`@RelationId` をもっていないので categoryIds と userId はありません。

```typescript
> const post2 = await getConnection().getRepository(Post2).findOne()
query: SELECT "Post2"."id" AS "Post2_id", "Post2"."contents" AS "Post2_contents", "Post2"."createdAt" AS "Post2_createdAt", "Post2"."updatedAt" AS "Post2_updatedAt", "Post2"."userId" AS "Post2_userId", "Post2"."attachId" AS "Post2_attachId" FROM "post2" "Post2" LIMIT 1

> post2
Post2 {
  id: 1,
  contents: 'contents-1',
  createdAt: 2021-07-11T16:42:13.392Z,
  updatedAt: 2021-07-11T16:42:13.392Z
}
```

Lazy relations は同じ設定なので await を使って categories と attach を参照できます。

```typescript
> await post2.categories
[
  Category { id: 1, name: 'category-1' },
  Category { id: 2, name: 'category-2' },
  Category { id: 3, name: 'category-3' }
]
> await post2.attach
Attach { id: 1, attr: 'attr-1' }
```

Lazy relations を一度参照すると、Entity は次のような構造になります。

```typescript
> post2
Post2 {
  id: 1,
  contents: 'contents-1',
  createdAt: 2021-07-11T16:42:13.392Z,
  updatedAt: 2021-07-11T16:42:13.392Z,
  __categories__: [
    Category { id: 1, name: 'category-1' },
    Category { id: 2, name: 'category-2' },
    Category { id: 3, name: 'category-3' }
  ],
  __has_categories__: true,
  __attach__: Attach { id: 1, attr: 'attr-1' },
  __has_attach__: true
}
```

#### Post3

Post2 と比べて、categories と attach を [Eager relations](https://github.com/typeorm/typeorm/blob/master/docs/eager-and-lazy-relations.md#eager-relations) として設定した Entity を作成します。

```typescript
  ...
  @ManyToMany((type) => Category, (category) => category.posts3, {
    nullable: true,
    eager: true,
  })
  @JoinTable()
  categories: Category[]
  ...
  @OneToOne((type) => Attach, (attach) => attach.post3, {
    nullable: true,
    eager: true,
  })
  @JoinColumn()
  attach: Attach;
  ...
```

REPL を使って Entity を取得します。
post3 を取得したタイミングで categories と attach も取得されていることがわかります。

```typescript
> const post3 = await getConnection().getRepository(Post3).findOne()
query: SELECT DISTINCT "distinctAlias"."Post3_id" as "ids_Post3_id" FROM (SELECT "Post3"."id" AS "Post3_id", "Post3"."contents" AS "Post3_contents", "Post3"."createdAt" AS "Post3_createdAt", "Post3"."updatedAt" AS "Post3_updatedAt", "Post3"."userId" AS "Post3_userId", "Post3"."attachId" AS "Post3_attachId", "Post3_categories"."id" AS "Post3_categories_id", "Post3_categories"."name" AS "Post3_categories_name", "Post3_attach"."id" AS "Post3_attach_id", "Post3_attach"."attr" AS "Post3_attach_attr" FROM "post3" "Post3" LEFT JOIN "post3_categories_category" "Post3_Post3_categories" ON "Post3_Post3_categories"."post3Id"="Post3"."id" LEFT JOIN "category" "Post3_categories" ON "Post3_categories"."id"="Post3_Post3_categories"."categoryId"  LEFT JOIN "attach" "Post3_attach" ON "Post3_attach"."id"="Post3"."attachId") "distinctAlias" ORDER BY "Post3_id" ASC LIMIT 1
query: SELECT "Post3"."id" AS "Post3_id", "Post3"."contents" AS "Post3_contents", "Post3"."createdAt" AS "Post3_createdAt", "Post3"."updatedAt" AS "Post3_updatedAt", "Post3"."userId" AS "Post3_userId", "Post3"."attachId" AS "Post3_attachId", "Post3_categories"."id" AS "Post3_categories_id", "Post3_categories"."name" AS "Post3_categories_name", "Post3_attach"."id" AS "Post3_attach_id", "Post3_attach"."attr" AS "Post3_attach_attr" FROM "post3" "Post3" LEFT JOIN "post3_categories_category" "Post3_Post3_categories" ON "Post3_Post3_categories"."post3Id"="Post3"."id" LEFT JOIN "category" "Post3_categories" ON "Post3_categories"."id"="Post3_Post3_categories"."categoryId"  LEFT JOIN "attach" "Post3_attach" ON "Post3_attach"."id"="Post3"."attachId" WHERE "Post3"."id" IN (1)

> post3
Post3 {
  id: 1,
  contents: 'contents-1',
  createdAt: 2021-07-11T16:42:14.180Z,
  updatedAt: 2021-07-11T16:42:14.180Z,
  categories: [
    Category { id: 1, name: 'category-1' },
    Category { id: 2, name: 'category-2' },
    Category { id: 3, name: 'category-3' }
  ],
  attach: Attach { id: 1, attr: 'attr-1' }
}
```

#### ER 図

整理のために ER 図は次のようになります。

Entity の構造 (設定) は意図的に差異を加えていますが、データベースからみたときのテーブルの構造と関連は全く同じとなるよう Post1, Post2, Post3 のテーブルを作っています。

{{< figure src="/blogs/2021/images/typeorm-relation-id/relation-id-tables.png"
           alt="relation id tables"
           title="@RelationID 検証のためのテーブル"
           class="common-style"
           width=768 >}}

この ER 図は [eralchemy](https://github.com/Alexis-benoist/eralchemy) というツールで出力できます。

```bash
$ eralchemy -i 'postgresql+psycopg2://root:password@localhost:15432/test' \
    -o relation-id-tables.png \
    --include-tables post1 post2 post3 user category attach \
                     post1_categories_category \
                     post2_categories_category \
                     post3_categories_category
```

### 検証対象の Entity とデータ数

それぞれ1万件程度のデータを生成しています。

```typescript
> await getConnection().getRepository(Post1).count()
10000
> await getConnection().getRepository(Post2).count()
10000
> await getConnection().getRepository(Post3).count()
10000
> await getConnection().getRepository(User).count()
10000
> await getConnection().getRepository(Attach).count()
10000
> await getConnection().getRepository(Category).count()
100
```

`@ManyToMany` の関連をもつ categories の関連テーブルは1つの Entity に3件ずつ、合計3万件になります。

```sql
test=# select count(1) from post1_categories_category ;
 count 
-------
 30000
test=# select count(1) from post2_categories_category ;
 count 
-------
 30000
test=# select count(1) from post3_categories_category ;
 count 
-------
 30000
```

### Entity の設定違いによるベンチマーク

次のようなベンチマークのためのテストを書いてみました。

```typescript
async function getMany<T>(repo: Repository<T>, n: number): Promise<number> {
  const start = new Date();
  await repo.createQueryBuilder().limit(n).getMany();
  const end = new Date();
  return getElapsedTime(start, end);
}
```

作成した実際のテストでは次の3つの API のベンチマークを取得しました。

* Repository.find()
* QueryBuilder.getMany()
* QueryBuilder.getRawMany()

TypeORM でデータを取得するとき、大きくわけて2通りの API があります。

* [Repository APIs](https://github.com/typeorm/typeorm/blob/master/docs/repository-api.md)
* [Select using Query Builder](https://github.com/typeorm/typeorm/blob/master/docs/select-query-builder.md)

Repository API は高レベル API となっており、内部的には Query Builder を使います。Query Builder を使った方が生成したい SQL をカスタマイズできます。

Repository.find() と QueryBuilder.getMany() は同じ結果になることを確認したので、本稿では QueryBuilder で呼び出す API の結果のみに限定して比較します。

```bash
$ yarn test --testNamePattern RelationId
  ● Console
      RelationId Repository.find()
      RelationId QueryBuilder().getMany()
      RelationId QueryBuilder().getRawMany()
```

私のマシンで実行した結果は次のようになりました。

{{< gist t2y 6c9283cf620740a3c1fee5bb9ddfc800 "results.md" >}}

`@RelationId` を用いた post1 で取得件数が増えるごとに経過時間が大きく増えていくことが確認できます。
これが取得件数に対して O(n^2) の計算量を要求するということです。

`getMany()` は Entity オブジェクトを生成し、
`getRawMany()` は SQL を実行して返ってきたデータを Entity 型ではなく JavaScript の Object 型で返します。
メソッド名の通り、生データを取得するための API になります。

TypeORM では `getMany()` と `getRawMany()` は意識して使い分けることが重要です。
この結果からもわかるように `getMany()` を使って Entity を生成するときに様々な処理が行われており、
それがパフォーマンスに大きく影響を与える可能性があるからです。

この結果をみると、(大きな差ではないですが) Lazy relations の post2 よりも Eager relations の post3 の方が速い結果になっていることもわかります。
Lazy relations に関するパフォーマンスの問題もまた別の記事で書いてみたいと思います。

## まとめ

現実のアプリケーションの例として、
私が開発に関わっているアプリケーションでは [type-graphql](https://github.com/MichalLytek/type-graphql) と
[type-graphql-dataloader](https://github.com/slaypni/type-graphql-dataloader) を使っています。

v0.3.7 までの type-graphql-dataloader では [README](https://github.com/slaypni/type-graphql-dataloader/blob/v0.3.7/README.md#with-typeorm) から引用すると、
次のように TypeORM の `@RelationId` を使って `@TypeormLoader` という DataLoader の機能を提供していました。

```typescript
@ObjectType()
@Entity()
export class Photo {
  @Field((type) => ID)
  @PrimaryGeneratedColumn()
  id: number;

  @Field((type) => User)
  @ManyToOne((type) => User, (user) => user.photos)
  @TypeormLoader((type) => User, (photo: Photo) => photo.userId)
  user: User;

  @RelationId((photo: Photo) => photo.user)
  userId: number;
}
```

DataLoader のパラメーターのためにすべての Entity が `@RelationId` を保持していて、
Entity を取得する件数によって致命的なパフォーマンスの問題を引き起こしていました。

[Add SimpleTypeormLoader without using typeorm @RelationId](https://github.com/slaypni/type-graphql-dataloader/pull/18)
の PR がマージされ、v0.4.0 からは `@RelationId` を必要としない実装に改善されています。

このように安易に TypeORM の `@RelationId` を使うと意図しないパフォーマンス問題を引き起こす可能性があるのでご注意ください。
