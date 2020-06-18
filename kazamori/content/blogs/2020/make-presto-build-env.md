---
title: "Presto のビルド環境を構築する"
slug: make-presto-build-env
date: 2020-06-17T23:02:04+09:00
with_date: true
tags: ["presto", "java", "build"]
---

Presto の開発をするときにビルド環境を構築する方法についてまとめます。

## Presto のビルド環境の構築

### 依存ライブラリのインストール

リポジトリを clone します。

```bash
$ git clone git@github.com:prestosql/presto.git
```

Presto は様々なコンポーネントやコネクターといったサブモジュールを1つのリポジトリに含みます。

```bash
$ cd presto
$ ls
CONTRIBUTING.md            presto-ml
LICENSE                    presto-mongodb
README.md                  presto-mysql
bin                        presto-oracle
docker                     presto-orc
mvnw                       presto-parquet
pom.xml                    presto-parser
presto-accumulo            presto-password-authenticators
presto-accumulo-iterators  presto-phoenix
presto-array               presto-pinot
presto-atop                presto-plugin-toolkit
presto-base-jdbc           presto-postgresql
presto-benchmark           presto-product-tests
presto-benchmark-driver    presto-product-tests-launcher
presto-benchto-benchmarks  presto-prometheus
presto-bigquery            presto-proxy
presto-blackhole           presto-raptor-legacy
presto-cassandra           presto-rcfile
presto-cli                 presto-record-decoder
presto-client              presto-redis
presto-docs                presto-redshift
presto-elasticsearch       presto-resource-group-managers
presto-example-http        presto-server
presto-geospatial          presto-server-main
presto-geospatial-toolkit  presto-server-rpm
presto-google-sheets       presto-session-property-managers
presto-hive                presto-spi
presto-hive-hadoop2        presto-sqlserver
presto-iceberg             presto-teradata-functions
presto-jdbc                presto-testing
presto-jmx                 presto-testing-server-launcher
presto-kafka               presto-tests
presto-kinesis             presto-thrift
presto-kudu                presto-thrift-api
presto-local-file          presto-thrift-testing-server
presto-main                presto-tpcds
presto-matching            presto-tpch
presto-memory              presto-verifier
presto-memory-context      src
presto-memsql
```

ビルドするためにまずこれらのサブモジュールの依存ライブラリをインストールします。

[Maven Wrapper](https://github.com/takari/maven-wrapper) という、
そのプロジェクトが使っている特定バージョンの Maven をダウンロードして使ってビルドする仕組みがあります。
`mvnw` という名前のラッパースクリプトが提供されていて実体はシェルスクリプトです。
Maven のバージョンが異なるとビルドできない可能性もあるため、`mvnw` を使ってビルドをします。

ここではビルド時間を短縮するためにテストは実行しないようにオプションを指定します。
テストコードのコンパイルも省略する `-Dmaven.test.skip` を
使うとエラーになった気がするので `-DskipTests` を指定します。

また現時点の最新バージョンをビルドするには JDK 11 以上が要求されます。

```bash
$ ./mvnw install -DskipTests
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  08:00 min
[INFO] Finished at: 2020-06-16T14:27:14+09:00
[INFO] ------------------------------------------------------------------------
```

私の環境では8分かかりました。サブモジュールや依存ライブラリが多いので完了するまで時間がかかります。

### PrestoServer の起動

Presto リポジトリにある README には IntelliJ IDEA 内のプロジェクトで PrestoServer を起動する方法があります。
IntelliJ IDEA で作業することに慣れている方はこれでいいと思いますが、ここでは CLI からビルドする方法も紹介します。

```bash
$ cd presto-server-main
$ ls
etc  pom.xml  src  target
$ tree src/
src/
└── main
    └── java
        └── io
            └── prestosql
                └── server
                    └── PrestoServer.java
```

このディレクトリは PrestoServer の `Main Class` のみがあります。

[Exec Maven Plugin](https://www.mojohaus.org/exec-maven-plugin/index.html) がすでにインストールされているのでこのプラグインを使って CLI から起動します。

PrestoServer を起動するときの JVM オプションを渡す方法として *JAVA_TOOL_OPTIONS* を使います。
Exec Maven Plugin が起動する Java アプリケーションに JVM オプションを渡す方法が私はわからなかったのでこうしています。
JVM オプションを渡した方は他に適切な方法があるかもしれません。

```bash
$ export JAVA_TOOL_OPTIONS="-ea -XX:+UseG1GC -XX:G1HeapRegionSize=32M -XX:+UseGCOverheadLimit -XX:+ExplicitGCInvokesConcurrent -Xmx2G -Dconfig=etc/config.properties -Dlog.levels-file=etc/log.properties -Djdk.attach.allowAttachSelf=true"
```

準備は整いました。次のようにして Exec Maven Plugin を実行します。

```bash
$ ../mvnw exec:java -Dexec.mainClass="io.prestosql.server.PrestoServer" -Dexec.workingdir="$(pwd)"
...
2020-06-18  INFO  io.prestosql.server.PrestoServer.main()  io.prestosql.server.Server  Working directory: path/to/presto/presto-server-main
2020-06-18  INFO  io.prestosql.server.PrestoServer.main()  io.prestosql.server.Server  Etc directory: path/to/presto/presto-server-main/etc
2020-06-18  INFO  io.prestosql.server.PrestoServer.main()  io.prestosql.server.Server  ======== SERVER STARTED ========
```

デフォルトで多くのプラグインが設定されているので起動に1分ほどかかるかもしれません。

サーバーが起動がしたら http://localhost:8080 にアクセスします。
ログイン画面が表示されれば正常に起動しています。
デフォルト設定ではユーザー認証の設定が行われていないため、任意のユーザー名でログインできます。
例えば、`admin` と入力してログインするとダッシュボードが表示されます。

### PrestoServer のカスタマイズ

`etc` 配下に `properties` ファイルがあります。
特定のプラグインを開発するときは関連するプラグインのみを
設定することで PrestoServer の起動時間を短縮できるでしょう。

```bash
$ tree etc/
etc/
├── access-control.properties
├── catalog
│   ├── blackhole.properties
│   ├── example.properties
│   ├── hive.properties
│   ├── iceberg.properties
│   ├── jmx.properties
│   ├── localfile.properties
│   ├── memory.properties
│   ├── memsql.properties
│   ├── mysql.properties
│   ├── postgresql.properties
│   ├── prometheus.properties
│   ├── raptor.properties
│   ├── sqlserver.properties
│   ├── thrift.properties
│   ├── tpcds.properties
│   └── tpch.properties
├── config.properties
├── jvm.config
└── log.properties
```

例えば、[Cassandra Connector](https://prestosql.io/docs/current/connector/cassandra.html)
のみを読み込みたいなら次のように設定します。

```bash
$ vi etc/config.properties
plugin.bundles=\
  ../presto-cassandra/pom.xml
```

Cassandra Connector の catalog 設定をします。
ここでは Cassandra クラスターをローカル環境に構築している前提として設定しています。

```bash
$ rm -f etc/catalog/*
$ vi etc/catalog/cassandra.properties
connector.name=cassandra
cassandra.contact-points=127.0.0.1,127.0.0.2,127.0.0.3
cassandra.native-protocol-port=9042
```

PrestoServer を起動すると、Cassandra Connector のプラグインが読み込まれます。

```bash
$ ../mvnw exec:java \
    -Dexec.mainClass="io.prestosql.server.PrestoServer" \
    -Dexec.workingdir="$(pwd)"
...
2020-06-18T  INFO  io.prestosql.server.PrestoServer.main()  io.prestosql.server.PluginManager  -- Finished loading plugin ../presto-cassandra/pom.xml --
2020-06-18T  INFO  io.prestosql.server.PrestoServer.main()  io.prestosql.metadata.StaticCatalogStore  -- Loading catalog etc/catalog/cassandra.properties --
2020-06-18T  INFO  io.prestosql.server.PrestoServer.main()  io.prestosql.metadata.StaticCatalogStore  -- Added catalog cassandra using connector cassandra --
```

#### Cassandra Connector のビルド

Cassandra Connector に変更を行った変更を反映するには
`presto-cassandra` のディレクトリでソースコードをコンパイルする必要があります。

```bash
$ cd ../presto-cassandra/
$ ../mvnw compile
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.667 s
[INFO] Finished at: 2020-06-18T10:35:10+09:00
[INFO] ------------------------------------------------------------------------
```

コンパイル後に PrestoServer を再起動することで変更が反映されます。

#### ビルドスクリプト

ここまでの手順を自動化するためにビルド向けの簡単なシェルスクリプトを書きました。
`presto-server-main` をワーキングディレクトリとして実行することを想定しています。

```bash
$ vi build.sh
#!/bin/bash

DEBUG="$1"
unset JAVA_TOOL_OPTIONS

set -eu

cd ..
./mvnw compile -pl presto-cassandra

target="presto-server-main"
cd $target

JAVA_TOOL_OPTIONS="-ea -XX:+UseG1GC -XX:G1HeapRegionSize=32M -XX:+UseGCOverheadLimit"
JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:+ExplicitGCInvokesConcurrent -Xmx2G"
JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -Dconfig=etc/config.properties"
JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -Dlog.levels-file=etc/log.properties"
JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -Djdk.attach.allowAttachSelf=true"
if [ -n "$DEBUG" ]; then
    echo "DEBUG is true"
    JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=5005"
fi
export JAVA_TOOL_OPTIONS=$JAVA_TOOL_OPTIONS
../mvnw exec:java -Dexec.mainClass="io.prestosql.server.PrestoServer" -Dexec.workingdir="."
```

`presto-cassandra` をコンパイルして PrestoServer を再起動する。

```bash
$ bash build.sh
```

`presto-cassandra` をコンパイルしてリモートデバッグ向けのオプションを指定して PrestoServer を再起動する。

```bash
$ bash build.sh debug
...
Listening for transport dt_socket at address: 5005
```

### Presto CLI で接続する

前節で設定した Cassandra クラスターに接続するために [Command Line Interface](https://prestosql.io/docs/current/installation/cli.html) を使います。

冒頭で `mvnw` でインストールしたときに `executable.jar` がビルドされているのですぐに使えます。
PrestoServer が起動していればプロンプトが表示されます。

```bash
$ cd presto-cli
$ java -jar target/presto-cli-337-SNAPSHOT-executable.jar \
    --server localhost:8080 --catalog cassandra
presto:system>
```

Cassandra クラスターにリクエストする CQL を入力します。

```
presto:system> select cluster_name, cql_version, native_protocol_version from local;
 cluster_name | cql_version | native_protocol_version
--------------+-------------+-------------------------
 v3-11-5      | 3.4.4       | 4
(1 row)

Query 20200618_015308_00004_ekxiw, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.24 [1 rows, 1B] [4 rows/s, 4B/s]
```

### リファレンス

* [github.com/prestosql/presto](https://github.com/prestosql/presto)
* [Presto Documentation](https://prestosql.io/docs/current/index.html)


## Presto の Eclipse 向けの設定

余談ですが、あまり情報がなかったのでメモ程度に書いておきます。

Presto のコア開発者は IntelliJ IDEA を使っている人しかいないそうで IntelliJ IDEA 向けの IDE 設定のみ README で説明されています。
Eclipse で開発するのに不便だったので少し調べました。
それでも面倒であることには変わりがないので Eclipse で開発することにこだわりがないのであれば IntelliJ IDEA を使った方がよいでしょう。

### インポート設定

Eclipse の設定ファイルを作成しようとしたときに `presto-phoenix` というモジュールでエラーが発生したので一時的にコメントアウトします。

```diff
$ git di pom.xml
diff --git a/pom.xml b/pom.xml
index a060b3be22..ebf6fce54c 100644
--- a/pom.xml
+++ b/pom.xml
@@ -101,7 +101,7 @@
         <module>presto-base-jdbc</module>
         <module>presto-mysql</module>
         <module>presto-memsql</module>
-        <module>presto-phoenix</module>
+        <!-- <module>presto-phoenix</module> -->
         <module>presto-postgresql</module>
         <module>presto-redshift</module>
         <module>presto-sqlserver</module>
```

次のようにして Eclipse の設定ファイルを作成します。
Eclipse 向けの `.project` や `.classpath` ファイルなどを作成してくれるようです。

```bash
$ ./mvnw eclipse:eclipse
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:55 min
[INFO] Finished at: 2020-06-18T11:24:24+09:00
[INFO] ------------------------------------------------------------------------
```

処理が完了したら Eclipse のインポート機能を使って既存の Maven プロジェクトとしてインポートします。

### コーディングスタイル

Presto では [Code style for Airlift projects](https://github.com/airlift/codestyle) というコーディングスタイルを採用しています。

コードを読むだけであればコーディングスタイルの設定は不要ですが、
ソースコードを変更したり拡張したりする上でコーディングスタイルが異なると開発の効率が悪くなります。
この設定も IntelliJ IDEA 向けにしか提供されていません。

Presto ではデフォルトで [Checkstyle](https://checkstyle.sourceforge.io/) という静的解析ツールで
コンパイル時に所定のコーディングスタイルのルールに準拠しているかをチェックします。
ローカルでソースを変更してビルドしたいときにコーディングスタイルのチェックが行われるのは億劫かもしれません。

Checkstyle のチェックを無効にするには次のように設定します。

```diff
$ git di pom.xml
diff --git a/pom.xml b/pom.xml
index a060b3be22..8cddab9638 100644
--- a/pom.xml
+++ b/pom.xml
@@ -39,6 +39,7 @@
         <air.check.skip-spotbugs>true</air.check.skip-spotbugs>
         <air.check.skip-pmd>true</air.check.skip-pmd>
         <air.check.skip-jacoco>true</air.check.skip-jacoco>
+        <air.check.skip-checkstyle>true</air.check.skip-checkstyle>

         <project.build.targetJdk>11</project.build.targetJdk>
         <air.modernizer.java-version>8</air.modernizer.java-version>
```

コーディングスタイルのチェックが必要な場合、
Airlift プロジェクト向けの設定を誰かが公開してくれています。

* https://gist.github.com/electrum/4627581

この設定を Eclipse にインポートして既存のソースコードと差分が出ないように
少しカスタマイズするとかなり近いコーディングスタイルにはなると思います。
