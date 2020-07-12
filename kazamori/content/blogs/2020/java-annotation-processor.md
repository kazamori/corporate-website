---
title: "Java のアノテーションプロセッサを試す"
slug: java-annotation-processor
date: 2020-07-12T17:05:57+09:00
with_date: true
tags: [java, code-generate, gradle]
---

Java のアノテーションプロセッサの仕組みは
Java 5 で *Annotation Processor Tool (以下APT)* という名称で登場し、
その後 Java 6 で *Pluggable Annotation Processing*
という仕組みが追加され、Java 8 で APT は削除されたそうです。

いま私たちが使うアノテーションプロセッサの仕組みは
*Pluggable Annotation Processing* になるわけですが、
過去の名残で APT という名前がそのまま使われていたりもしているようです。
アノテーションプロセッサについて調べると検索結果に APT というキーワードも出てきて
意図するものがどちらを指しているのか、ややこしいかもしれません。

ある Kafka streams を扱うプロジェクトで
ボイラープレートコードを減らす意図で
アノテーションプロセッサを採用する機会があったので調べてみました。

サンプルコードは次になります。
Kafka streams アプリケーションの検証の過程で作成したので
Avro スキーマの定義とコード生成もプロジェクトに含まれています。
そこは本質ではないのであまり気にしなくてよいです。

* https://github.com/t2y/annotation-processors-sample


## アノテーションプロセッサの用途

wikipedia の [Metaprogramming](https://en.wikipedia.org/wiki/Metaprogramming)
にアノテーションプロセッサは含まれていないので、コード生成の手法の1つではありますが、
アノテーションプロセッサによるコード生成をメタプログラミングとは呼ばないようです。
プリプロセッサと呼ぶ方が適切なのかもしれません。

アノテーションプロセッサを使った有名なライブラリに [Lombok](https://projectlombok.org/) があります。
Lombok はアノテーションが付いたコードの AST 変換により、コード生成を実現しています。
これは AST を直接書き換えるというもう1段階難しいコード生成のやり方ですが、
今回のサンプルコードではそこまでやらず、単純にアノテーションが付いたコードの情報を取得して、
それらの情報を使って別のソースコードを生成するものを試しています。

ソースコードを生成するためにテンプレートエンジンとして Mustache 互換の
[Handlebars.java](https://jknack.github.io/handlebars.java/) を使っています。
テンプレートエンジンも必須ではありませんが、
その方がコード生成の保守がしやすくなるので使うとよいでしょう。

別のやり方として [JavaPoet](https://github.com/square/javapoet) という、
DSL で Java のコード生成を行うライブラリもあるようです。
リファレンスの記事を読んでいて知りました。

試したサンプルコードの簡単な概要説明をこの後に書いていきます。
アノテーションプロセッサについて調べている方は、
私の記事よりも次のリファレンスを読んだ方がわかりやすいかもしれません。

### リファレンス

* [アノテーションプロセッサで AST 変換 - Lombok を参考にして変数の型をコンパイル時に変更](https://fits.hatenablog.com/entry/2015/01/17/172651)
* [「Java SE 6完全攻略」第94回 アノテーションを処理する その1](https://xtech.nikkei.com/it/article/COLUMN/20081219/321816/)
* [java モダンな Annocation Processor の開発手順まとめ](http://blog.64p.org/entry/2015/03/26/055347)


## Gradle でアノテーションプロセッサを使う

数年前に書かれたアノテーションプロセッサの記事をみると、Maven での設定方法が紹介されています。
Gradle でどういった設定をすればよいか、調べるのが難しかったので整理しておきます。

以前に書いた [Gradle のマルチプロジェクト]({{< relref "gradle-multi-project.md" >}}) な構成でサンプルコードを書いています。

* `annotation`: アノテーションを定義するサブプロジェクト
* `processor`: アノテーションプロセッサを定義するサブプロジェクト
* `app`: アノテーションプロセッサを使うアプリケーションのサブプロジェクト

`annotation` と `processor` のサブプロジェクトを分ける必要性があるかどうか、
私自身には明確なポリシーはありません。
一般論としてアノテーションとアノテーションプロセッサが
使われる場面や開発サイクルが異なるので分離しておくといった程度でしょうか。

パッケージングして配布することを考慮すれば、
アノテーションとアノテーションプロセッサは
別途インストールできる方が利用者の用途にあうと言えるでしょう。

開発時の `app` サブプロジェクトからみたとき、
`annotation` と `processor` のそれぞれのサブプロジェクトに依存関係を記述します。

#### app/build.gracle

```groovy
dependencies {
    compile project(':annotation')
    annotationProcessor project(':processor')
}
```

`processor` サブプロジェクトからは `annotation` サブプロジェクトに依存します。

#### processor/build.gradle

```groovy
dependencies {
    annotationProcessor 'com.google.auto.service:auto-service:1.0-rc7'
    compileOnly 'com.google.auto.service:auto-service:1.0-rc7'

    compile project(':annotation')
}
```

後述しますが、
[AutoService](https://github.com/google/auto/tree/master/service) も使うことをお勧めします。

これで `app` のアプリケーションのソースコードをコンパイルするときに
`processor` に定義されたアノテーションプロセッサがコンパイルされて実行されるようになります。


## アノテーションプロセッサの実装

*AbstractProcessor* を継承して `process()` メソッドを実装します。

```java
@AutoService(Processor.class)
@SupportedSourceVersion(SourceVersion.RELEASE_11)
@SupportedAnnotationTypes({"sample.annotation.MyType", "sample.annotation.MyField"})
public class AvroSchemaGenerator extends AbstractProcessor {

  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    if (annotations.isEmpty()) {
      return false;
    }
    ...
  }
}
```

一緒にサポートする Java のソースバージョンと、
このアノテーションプロセッサがサポートするアノテーションを宣言しています。

* `@SupportedSourceVersion(SourceVersion.RELEASE_11)`
* `@SupportedAnnotationTypes({"sample.annotation.MyType", "sample.annotation.MyField"})`

*@SupportedAnnotationTypes* に設定したアノテーションを含むソースコードのとき、
`process()` メソッドに渡される `annotations` に値がセットされるようです。
そのことを利用してこのサンプルコードではアノテーションプロセッサの処理を行うかどうかの制御をしています。

## アノテーションプロセッサのメタデータ

Java のコンパイラからアノテーションプロセッサをみつけるには、
次のメタデータに自分が開発したアノテーションプロセッサのクラス名を記述しなければなりません。

* `src/main/resources/META-INF/services/javax.annotation.processing.Processor`

自分でファイルを作成して定義しても構いませんが、
開発中にアノテーションプロセッサを複数定義したり名前変更したりすることを考慮すると、
クラス名から自動的にメタデータを生成してくれると便利です。

それをしてくれるのが先ほど紹介した [AutoService](https://github.com/google/auto/tree/master/service) です。

開発したアノテーションプロセッサのクラスに次のアノテーションを付けます。

```java
@AutoService(Processor.class)
```

そうすると、AutoService が `processor` サブプロジェクトのアノテーションプロセッサとして実行され、
次の場所に自動的に `javax.annotation.processing.Processor` ファイルを作成してクラス名を設定してくれます。

```bash
$ cat processor/build/classes/java/main/META-INF/services/javax.annotation.processing.Processor 
sample.processor.AvroSchemaGenerator
```

このメタデータの設定を忘れると、
アノテーションプロセッサがみつからないというエラーになります。
私はこのメタデータの設定を失念していてしばらくはまりました。


## Handlebars テンプレートを使ったコード生成

[Handlebars.java](https://jknack.github.io/handlebars.java/) も少しだけ紹介します。
私自身あまり詳しくありませんが、[Mustache](https://mustache.github.io/)
という昔からあるテンプレートエンジンのスーパーセットになるそうです。
[handlebars.js](https://github.com/handlebars-lang/handlebars.js) により
JavaScript でも使えるテンプレートなので Java と JavaScript で
テンプレートを共有したいときなどにも便利なのかもしれません。

例えば、次のように Java のコードを生成するテンプレートを作成します。

#### processor/src/main/resources/serde_template.hbs

```handlebars
package {{ packageName }};

import io.confluent.kafka.streams.serdes.avro.SpecificAvroSerde;

public class {{ className }} extends SpecificAvroSerde<{{ typeParameter }}> {}
```

アノテーションプロセッサでは、
*JavaFileObject* という Java のソースファイルを抽象化したオブジェクトを作成し、
Handlebars のテンプレートにその writer を渡すことで実際のソースコードを生成してくれます。

```java
    HashMap<String, String> map = new HashMap<>();
    map.put("packageName", packageName);
    map.put("className", serdeSimpleClassName);
    map.put("typeParameter", simpleClassName);

    JavaFileObject obj = processingEnv.getFiler().createSourceFile(serdeClassName);
    try (PrintWriter out = new PrintWriter(obj.openWriter())) {
      SERDE_TEMPLATE.apply(map, out);
```

生成したコードは `build/generated/sources/annotationProcessor/` 配下に置かれます。

#### app/build/generated/sources/annotationProcessor/java/main/sample/data/ContainerSerde.java

```java
package sample.data;

import io.confluent.kafka.streams.serdes.avro.SpecificAvroSerde;

public class ContainerSerde extends SpecificAvroSerde<Container> {}
```

## デバッグ

アノテーションプロセッサのデメリットの1つとして、
通常の Java アプリケーションの開発と比べて相対的にデバッグが難しいことです。

いくつかやり方はあると思いますが、
私が試した方法として gradle コマンドそのものに
デバッグオプションを渡してしまうことでリモートデバッグができました。
開発環境 (IDE) の違いによってやり方は変わると思うのでご参考まで。

```bash
$ ./gradlew :app:compileJava -Dorg.gradle.jvmargs='-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=5005'
```


## まとめ

私自身、アノテーションプロセッサの存在はずっと前から知っていましたが、
今回初めて自分で実装してみる機会がありました。

仮に同じことをリフレクションで実現できるとしても、
リフレクションを使うよりもアノテーションプロセッサを使う方が
次のようなメリットがあります。

* 型安全なアプリケーションを開発できる
* (リフレクションより) パフォーマンスがよい
* 誤っていたときにコンパイルエラーになる

依存関係を整理して開発環境さえ構築できれば、
さほど普通の Java アプリケーションの開発と違いがあるように私は感じません。
アノテーションプロセッサは開発スキルの
引き出しの1つとして知っておくとよい技術と言えるでしょう。
