---
title: "Repro株式会社様 システム開発"
slug: repro-subcontract
date: 2020-10-14T09:10:08+09:00
with_date: true
tags: [subcontract, adviser, repro]
---

[Repro](https://repro.io/) 様でシステム開発のお手伝いをした内容を紹介します。

> {{< figure src="/cases/2020/images/repro-logo-colored.png" link="https://repro.io/" target="_blank" alt="Repro LOGO" width="25%" >}}

> Reproは顧客データを活用し、 メールやプッシュ通知、Webやアプリ内ポップアップなどのチャネルを横断した付加価値の高いコミュニケーションを実現するカスタマーエンゲージメントプラットフォームです。

Repro 社はこの分野で業界トップレベルのシェアと顧客満足度を獲得しています。また [Ruby活用事例](https://www.ruby.or.jp/ja/showcase/case49.html) にもあるように、Ruby (Ruby on Rails) によるプラットフォーム開発で有名な企業でもあります。

一方で大規模なデータを扱うミドルウェアとして Java で実装された Cassandra, Presto, Kafka なども導入しています。そして、さらなるサービスの拡大、プロダクト品質の向上のため、データ処理基盤を整備し直す際に [Kafka Streams](https://kafka.apache.org/documentation/streams/) を採用することに決めました。Kafka Streams は [Kafka](https://kafka.apache.org/) 上に分散ストリームアプリケーションを簡単に開発するためのフレームワークです。

Repro 社では、ユーザーの行動をイベントとして受け取り、そのデータを処理するイベントストリームと呼ばれるシステムがあります。Kafka Streams 上に Repro 社のイベントストリームのアプリケーションを実装し、リアルタイムなユーザー行動の分析に活用しています。

このアプリケーション開発をCTO 橋立氏が陣頭にたって開発しており、その経験を 2020 AWS Summit Online で発表しています。講演動画と講演資料は次になります。

* [CUS-47：Building Semi-Realtime Processing System with Kafka and Kafka Streams on Amazon ECS](https://resources.awscloud.com/vidyard-all-players/cus-47-aws-summit-online-2020-repro-inc)
* [講演資料はこちら](https://pages.awscloud.com/rs/112-TZM-766/images/CUS-47_AWS_Summit_Online_2020_Repro-inc.pdf)

実際に大規模なデータを扱う上での Kafka の運用経験、ならびに Kafka Streams アプリケーションの開発経験を24分という短い時間に要点のみを説明されています。Kafka/Kafka Streams をこれから導入しようと考えている開発者向けにとても示唆に富む内容と言えます。

Repro 社の Kafka Streams アプリケーションの開発に当社も参加して、具体的には次のようなことを行いました。

* [Effective Java 第3版](https://www.maruzen-publishing.co.jp/item/?book_no=303054) に書かれている設計手法を実際の開発やコードレビューで実践
* Java 開発のプラクティスに準拠していない設計のリファクタリング
* Repro 社の Kafka Streams アプリケーションの開発

当社はフルリモートワークで開発しています。オンライン上のコミュニケーションは [Slack](https://slack.com/)、ミーティングは [Google Meet](https://workspace.google.com/products/meet/)、課題管理は [ClickUp](https://clickup.com/)、ドキュメントは [esa](https://esa.io/) といったツールを主に使っています。
