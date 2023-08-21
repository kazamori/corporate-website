---
title: "OSSTech株式会社様 システム開発"
slug: osstech-subcontract
date: 2023-08-21T17:52:36+09:00
with_date: true
tags: [subcontract, manager, osstech]
---

[OSSTech](https://www.osstech.co.jp/) 様でシステム開発のお手伝いをした内容を紹介します。

> {{< figure src="/cases/2023/images/osstech-logo.svg" link="https://www.osstech.co.jp/" target="_blank" alt="OSSTech Corporation LOGO" width="70%" >}}
> <br />
> OSSTechは、国内随一の技術力を持つオープンソースソフトウェア（OSS）開発者を中心に、自社でソースコードの開発・修正を実施。高機能かつ高品質なOSS製品と、統合認証 / シングルサインオン / アイデンティティ管理 / eKYCソリューションおよびサポートサービスを提供しています。

OSSTech 社は OSS を活用して国内トップレベルの統合認証に関するプロダクトやソリューションを提供しているパッケージベンダーです。ID 連携のための新規プロダクト開発において、当社はプロジェクトマネージャーの役割でその開発に携わりました。

当社は新規プロダクトのインフラ設計とプロジェクトマネジメントを担当し、課題管理とイテレーション開発を基本とした開発方法論を提案し、実際のプロダクト開発を通してメンバーとともにその実践を行いました。**よいプロダクトを開発するには「よい開発文化」** を根付かせることが必要です。課題管理はその具体的手段となります。課題管理をうまく行うことでメンバーはほぼフルリモートワークでプロダクト開発しています。

その成果として [Unicorn Cloud ID Manager](https://www.osstech.co.jp/product/ucidm/) (以下UCIDM) という、既存の社内システムから外部サービスへ ID 連携するためのシステムが出来上がりました。先日そのプレスリリースが行われました。

[新製品発表、クラウド対応ＩＤ管理製品を新たに提供開始](https://prtimes.jp/main/action.php?run=html&page=releasedetail&company_id=38710&release_id=16)

{{< figure src="/cases/2023/images/osstech-ucidm-logo.svg" link="https://www.osstech.co.jp/product/ucidm/" target="_blank" alt="OSSTech Unicorn Cloud ID Manager LOGO" width="65%" >}}

> オンプレミス環境のActive DirectoryやOpenLDAPからSaaS（クラウド環境）へユーザー情報を自動で連携し、シングルサインオン環境を効率的に運用するための新製品をリリース

UCIDM は ID 連携するときに既存の社内システムの変更を最小限にして、システム管理者の運用コスト削減を狙いとしています。もっとも有効な用途の1つに OSSTech 社のシングルサインオン製品と組み合わせ ID 連携サービスをワンストップで提供できます。またインフラ視点からの特徴の1つとして、UCIDM を導入する企業のインフラはオンプレミス／クラウドを問わず、どちらでも導入できるよう設計されており、コンテナ技術を用いて機能開発の素早さと高品質のデプロイを両立した仕組みとなっています。

開発における要素技術としては次になります。バックエンドは Go 言語、フロントエンドは Svelte で開発しています。

フロントエンド

* [Svelte](https://svelte.dev/)
* [SvelteKit](https://kit.svelte.dev/)

バックエンド

* [Echo](https://echo.labstack.com/)
* [go-ldap](https://github.com/go-ldap/ldap)

インフラ

* [Docker](https://www.docker.com/)

開発を進めながら、技術情報の共有のために OSSTech 社のテックブログに書いた記事が次になります。OSSTech 社は組織として技術情報の発信、OSS 活動やコントリビューションを奨励しています。

* [フロントエンドの技術選定](https://blog.osstech.co.jp/posts/2023/02/frontend-tech-selection/)
* [Docker とは何か？](https://blog.osstech.co.jp/posts/2023/05/what-is-docker/)
* [go-ldap へのコントリビューション](https://blog.osstech.co.jp/posts/2023/08/go-ldap-contribution/)

当社はフルリモートワークで開発しています。オンライン上のコミュニケーションは [Slack](https://slack.com/)、ミーティングは [ハドルミーティング](https://slack.com/intl/ja-jp/help/articles/4402059015315-Slack-%E3%81%A7%E3%83%8F%E3%83%89%E3%83%AB%E3%83%9F%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B) と [Google Meet](https://workspace.google.com/products/meet/)、課題管理とドキュメントは [GitLab](https://about.gitlab.com/) といったツールを主に使っています。
