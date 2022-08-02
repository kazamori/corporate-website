---
title: "星野リゾート様 業務委託"
slug: hoshinoresorts-subcontract
date: 2022-04-26T08:16:35+09:00
with_date: true
tags: [subcontract, adviser, hoshinoresorts]
---

[星野リゾート](https://www.hoshinoresorts.com/) 様でシステム開発のお手伝いをした内容を紹介します。

> {{< figure src="/cases/2022/images/hoshino-resorts-logo.png" link="https://www.hoshinoresorts.com/" target="_blank" alt="Hoshino Resorts LOGO" width="30%" >}}

> 『100年後に旅産業は世界で最も大切な平和維持産業になっている』
> 
> 旅をすることでその場所に住んでいる人たちと触れ合い、友人になることができる。『世界の人たちを友人として結んでいく』、それは他の産業にはできない、旅の魔法なのです。

星野リゾート社は、創業108年 (1914年創業) という老舗の会社です。エンジニアチームリーダーの藤井氏が [Regional Scrum Gathering Tokyo 2022](https://2022.scrumgatheringtokyo.org/index.html) などのイベントで発表するなど IT 活用にも力を入れています。

* [非ITの宿泊業なのに、なぜDXを推進できるのか？](https://confengine.com/conferences/regional-scrum-gathering-tokyo-2022/proposal/16167/itdx)
* [講演資料はこちら](https://www.slideshare.net/ssuser91c7c7/itdx-250944800)

「情報システムグループの歴史とユニットメンバーの推移」のスライドでも過去3度に渡る同社の内製化への取り組みが紹介されています。同社の開発体制は情報システムグループが中心となり、独自の組織文化を強みにアジャイル開発で自分たちの業務システムを内製しています。開発方法論としてスクラムを取り入れています。同社の組織文化の1つである多能工としての働き方は、開発者であれば、フロントエンドからバックエンド・インフラのすべてのシステム領域の開発や保守に携われることを意味します。複数の技術領域にまたがって活躍できるキャリアをフルスタックエンジニアと呼ぶのであれば、そのような働き方に近いと言えます。

開発における要素技術としては次になります。

フロントエンド
* [Nuxt](https://nuxtjs.org/ja/)

バックエンド
* [Spring Boot](https://spring.io/projects/spring-boot)

インフラ
* [Kubernetes](https://kubernetes.io/)
* [Dapr](https://dapr.io/)
  * [Dapr Adopters](https://github.com/dapr/community/blob/master/ADOPTERS.md) に dapr を本番環境またはテスト段階で利用している組織として掲載
* [GitHub Actions](https://github.co.jp/features/actions)

CI/CD に GitHub Actions を活用しており、当社もこの技術領域には大きな関心をもっています。そのため、カスタムアクションの開発にも積極的に取り組んでおり、開発の自動化や効率化を推進しています。カスタムアクションは GitHub というプラットフォーム上で組織の垣根を超えて再利用可能な拡張であり、オープンソース的な取り組みを支持する組織からみて優れた仕組みにみえます。

* [backlog-github-integration-action](https://github.com/kazamori/backlog-github-integration-action): GitHub と Backlog の連携
* [hr-library-auto-update](https://github.com/HoshinoResort/hr-library-auto-update): maven を使った依存パッケージの自動更新

さらに同社の大橋氏が Github Actions を活用した新しい Java の開発スタイルを JJUG CCC 2022 Spring で発表します。素早くプロダクトをリリースするために継続的デリバリーを強く意識した開発スタイルに取り組んでいる事例になります。

* [開発者にやさしく、柔軟性、安全性を高めたGithub ActionsベースのCI/CDを構築する](https://fortee.jp/jjug-ccc-2022-spring/proposal/153b87bf-85c4-4274-9954-4e3f614724cb)
* [【登壇報告】JJUG CCC 2022 Spring で語りきれなかった技術的なお話](https://note.com/hoshino_technote/n/n6afff42aeee0)

当社はフルリモートワークで開発しています。オンライン上のコミュニケーションは [Slack](https://slack.com/)、ミーティングは [ハドルミーティング](https://slack.com/intl/ja-jp/help/articles/4402059015315-Slack-%E3%81%A7%E3%83%8F%E3%83%89%E3%83%AB%E3%83%9F%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B) と [Google Meet](https://workspace.google.com/products/meet/)、課題管理は [Backlog](https://backlog.com/ja/) といったツールを主に使っています。
