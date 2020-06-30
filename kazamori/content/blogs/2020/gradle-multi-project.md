---
title: "Gradle のマルチプロジェクト機能を試す"
slug: gradle-multi-project
date: 2020-06-30T09:47:37+09:00
with_date: true
tags: [gradle, java, build]
---

モダンな Java のプロジェクトでは [Gradle](https://gradle.org/) というビルドツールがよく使われているようにみえます。
maven のように xml を記述しないだけでも好みかなと思って私も2-3年前ぐらいから新規に開発するプロジェクトでは Gradle を採用しています。

あるプロジェクトで Gradle の multi-project を採用する機会があったので調べてみました。サンプルコードは次になります。

* https://github.com/t2y/gradle-multi-project-sample

## multi-project とは

1つのリポジトリ内に複数のプロジェクト (ビルド設定) をもつようなプロジェクト構成を指します。
通常の1つのリポジトリに1つのプロジェクトではなく、multi-project を採用する背景としては次のようなものでしょうか。

* コア機能とプラグインのようなものを明確に区別して開発したい
* モジュール間のバージョン管理をリポジトリ内で管理したい
* 開発チームを分離してそれぞれのプロジェクトごとに役割分担して開発したい

他にもあるでしょうけど、私がパッと思いつくものをあげました。

好みで言えば、私はあまり multi-project な構成が好きではありません。
小さいプロジェクトであれば、1つのプロジェクト内でうまく設計すればよいし、
プラグインのようなものを明確に分けたいならリポジトリごと分けたほうがわかりやすくてよい気がします。
プロジェクトの数があまり増えていかないのがわかっているのであれば、こういった構成も機能するとは思います。
なんとなく1つのリポジトリが肥大化する機構そのものが好ましくないと思ってしまうため、
私は Multi-Project な構成を好ましく思わないのかもしれません。

## root project と sub-projects

リポジトリのトップディレクトリの階層のことを root project と言います。
`settings.gradle` に root project と sub-projects の設定をします。
次の設定では `include` というキーワードに続くものが sub-project を指します。

### settings.gradle

```groovy
rootProject.name = 'gradle-multi-project-sample'

include 'common'
include 'base'
include 'app'
```

root project の `build.gradle` はいくつか異なる設定をすることになります。

### build.gradle

```groovy
plugins {
    id 'com.github.sherter.google-java-format' version '0.9' apply false
}

allprojects {
    version = '1.0'
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'com.github.sherter.google-java-format'

    dependencies {
        // for type inference
        annotationProcessor 'org.projectlombok:lombok:1.18.12'
        compileOnly 'org.projectlombok:lombok:1.18.12'
        testAnnotationProcessor 'org.projectlombok:lombok:1.18.12'
        testCompileOnly 'org.projectlombok:lombok:1.18.12'

        // for logging
        implementation 'ch.qos.logback:logback-classic:1.2.3'
        implementation 'org.slf4j:slf4j-api:1.7.26'

        implementation 'com.google.guava:guava:29.0-jre'
        testImplementation 'org.junit.jupiter:junit-jupiter-api:5.6.2'
        testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.6.2'
    }

    repositories {
        jcenter()
    }

    test {
        useJUnitPlatform()
    }

    compileJava.dependsOn tasks.googleJavaFormat
}
```

`allprojects` というブロックは root project とすべての sub-projects に適用される設定を記述します。
そして `subprojects` というブロックは sub-projects のみに適用される設定を記述するところです。
この例では適用するプラグインや依存パッケージ、テストライブラリなどの共通設定を記述しています。

ここで google-java-format のプラグイン設定は一工夫が必要になります。

```groovy
plugins {
    id 'com.github.sherter.google-java-format' version '0.9' apply false
}

subprojects {
    apply plugin: 'com.github.sherter.google-java-format'

    compileJava.dependsOn tasks.googleJavaFormat
}
```

`plugins` ブロックでプラグイン設定を記述する方法は新しい構文になり、
現時点では `subprojects` ブロックではその構文に対応していないようです。
そのため、`plugins` ブロックでプラグインのバージョン指定を行い、
`apply false` を指定することでこのプラグインを root project には適用しないことを宣言しています。
このようにして `subprojects` ブロックでは `apply plugin:` という古い構文で設定を適用できます。


## サブプロジェクトの設定

サブプロジェクトの `build.gradle` の設定は普通の Gradle プロジェクトのように設定できます。
`plugins` にある `id 'java'` は root project の `build.gradle`
にも記述されているのでこちらに記述しなくても問題はないようです。
ただサブプロジェクトの設定だけをみたときに適用されているプラグインを
すべて記述した方がわかりやすいという考え方もあるので二重に設定しても問題にはならないようです。

`app` というサブプロジェクトはアプリケーションを意図しているので
メインクラスや実行可能な jar を生成する設定などを追加しています。

### app/build.gradle

```groovy
plugins {
    id 'java'
    id 'application'
}

mainClassName = 'app.Main'

dependencies {
    compile project(':base')
}

task customFatJar(type: Jar) {
    manifest {
        attributes 'Main-Class': 'app.Main'
    }
    baseName = 'executable'
    from { configurations.compile.collect { it.isDirectory() ? it : zipTree(it) } }
    with jar
}
```

`dependencies` のブロックに一つ見慣れない構文が出てきます。

```groovy
dependencies {
    compile project(':base')
}
```

これは `app` というサブプロジェクトが
`base` という別のサブプロジェクトに依存していることを表しています。

## gradle コマンドの操作

gradle コマンドの操作もほとんど違和感がないので簡単です。

### プロジェクト名を表示する

```bash
$ gradle -q projects

------------------------------------------------------------
Root project
------------------------------------------------------------

Root project 'gradle-multi-project-sample'
+--- Project ':app'
+--- Project ':base'
\--- Project ':common'
```

### コンパイルする

トップディレクトリで gradle コマンドを操作すると基本的にはすべてのサブプロジェクトに適用されます。

```bash
$ gradle compileJava
```

特定のサブプロジェクトを指定して操作したいときは
`:base:` のようにサブプロジェクト名に続けてタスクを指定します。

```bash
$ gradle :base:compileJava
```

もちろん特定のサブプロジェクトのディレクトリに移動したときは、
そこで行う gradle コマンドの操作はそのサブプロジェクトのみに適用されます。

```bash
$ cd base/
$ gradle compileJava  # base サブプロジェクトをコンパイル
```

接頭辞としてサブプロジェクト名を指定する以外、
違和感なく gradle の操作が行えるのでとてもよくできているように思いました。

## 実際のプロジェクトの実用例

実際のプロジェクトの実用例として embulk で使われているようです。
こういった実用例からどのような設定をしているかを参考にするのもよいと思います。

* https://github.com/embulk/embulk

### リファレンス

* [Creating Multi-project Builds](https://guides.gradle.org/creating-multi-project-builds/)
* [Executing Multi-Project Builds](https://docs.gradle.org/current/userguide/intro_multi_project_builds.html)
