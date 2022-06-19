---
title: "Java で作るカスタム GitHub Actions"
slug: custom-github-actions-by-java
date: 2022-06-19T11:00:30+09:00
with_date: true
tags: [java, event, github]
---

[JJUG CCC 2022 Spring](https://ccc2022spring.java-users.jp/viewer) で発表しました。今回はオンラインイベントでした。

* [Java で作るカスタム GitHub Actions](https://fortee.jp/jjug-ccc-2022-spring/proposal/0c85f6b2-d44d-40c2-8e6d-ddc1fe821273)

<iframe class="speakerdeck-iframe" frameborder="0" src="https://speakerdeck.com/player/122d264be23b4492aeecfd00063cbabc" title="Custom GitHub Actions by Java" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true" style="border: 0px; background: padding-box padding-box rgba(0, 0, 0, 0.1); margin: 0px; padding: 0px; border-radius: 6px; box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 40px; width: 560px; height: 314px;" data-ratio="1.78343949044586"></iframe>

* https://speakerdeck.com/kazamori/custom-github-actions-by-java

## カスタム GitHub Actions

あまり深く名前を考えていなかったものもあり、シンプルに機能を表す Backlog-GitHub integration action という名前で公開しています。

* https://github.com/marketplace/actions/backlog-github-integration-action
* https://github.com/kazamori/backlog-github-integration-action

オープンソースで開発しており、どなたでも利用できます。

他の既存のカスタムアクションとの違いは、Pull Request と Commit 連携の両方をこのカスタムアクションが扱えるところです。機能的な違いはほとんどないと思います。もし必要な機能がないということがあれば、[GitHub Discussions](https://github.com/kazamori/backlog-github-integration-action/discussions) で要望をあげてください。皆さんが使うような汎用的な機能であれば、なるべく開発したいと考えています。

## カスタムアクション開発の経緯

[星野リゾート様の開発のお手伝い]({{< relref "hoshinoresorts-subcontract" >}}) において、課題管理システムに [ヌーラボ社の Backlog](https://backlog.com/ja/) を利用しています。私はこのお手伝いで初めて Backlog を使い始めました。そして Backlog では GitHub 連携の機能を提供されていないことに気付きました。そのため、GitHub 上の Pull Request や Commit が Backlog 上の課題とリンクされていませんでした。

課題と開発状況を直接的にリンクすることによるメリットは、これまでのソフトウェア開発方法論で [チケット駆動開発](https://ja.wikipedia.org/wiki/%E3%83%81%E3%82%B1%E3%83%83%E3%83%88%E9%A7%86%E5%8B%95%E9%96%8B%E7%99%BA) が実践してきたことであり、その意義は開発のワークフローを効率化することを前提として多岐に渡ります。本稿ではその詳細について説明しませんが、いまどきの開発方法論では「できて当たり前」のようなものであると私は考えています。「できて当たり前」の機能がないのが気になったので開発しましたというのが経緯になります。

他にも [ヌーラボ社の Backlog コミュニティ](https://ja.community.nulab.com/u/tetsuyamorimoto-5hx/activity) で要望をあげたりもしています。

## なぜ Java で開発したのか

発表時間の都合で説明しなかったのですが、実は当初 Go 言語で開発しようと考えていました。有志の開発者が作った Go の Backlog クライアントを使って検証していたら、私の環境では、なぜかある一部の機能がうまく動かなくてエラーになりました。コア部分の実装でエラーが発生していて、そのデバッグを30分ほどしてみたものの、エラーの原因がわからなくてそのまま採用を断念しました。ヌーラボ社が公式提供しているライブラリが [backlog4j](https://github.com/nulab/backlog4j) という Java クライアントのみだったので Java で作ろうと決心した次第です。

## 所感

JJUG CCC 2022 Spring はオンラインイベントだったので事前に発表内容をビデオで提出するという段取りでした。発表内容をビデオで撮って提出するというのは私にとって初めての試みでした。1ヶ月前にビデオを提出しないといけないという、早めの締め切りに対して準備が必要というデメリットはありますが、当日は質疑応答のみでよいので当日の負担が減ります。またビデオなら何度か撮り直しできるのでうまく発表できるように調整できます。私は3回ビデオを撮り直しました。15分という時間にあわせるため、説明を省略したり、曖昧なところを補足したりできます。シンプルに同じ内容を話すだけでも1回目よりは2回目の方が、2回目よりは3回目の方が流暢に説明できます。

スタッフ側も内容を事前確認でき、当日に発表者の体調不良などがあってもセッション自体がなくなることもありません。オンラインイベントなら事前にビデオを撮るというやり方のメリットがわかりました。当日は zoom で待機しながら自分のビデオの発表を聴きつつ、そのビデオの配信が終わってから質疑応答になりました。朝早い時間というのもあり、おそらく視聴者も少なくスタッフからの質問に答えるというのが主でした。失敗点を1つあげると、Zoom と YouTube Live を組み合わせて配信されていたので Zoom にログインするとそこでもビデオ配信が行われていたため、YouTube Live の画面を閉じてしまいました。そして、質問が YouTube のチャット欄に書き込まれたときに質問を確認できないことに気付きました。

これは私もオンライン勉強会をしていてよくあることなのですが、質問を書き込む場所が複数あると運営の負荷が格段に上がります。限られた時間で質疑応答を行わないといけない中で質問を確認する場所が複数あるだけで手間暇が増えてしまいます。多くの参加者から質問を受け付けたいが、質問自体は一元管理したいというトレードオフの関係になっています。そして、チャット欄はプラットフォームに紐付いた機能なので質問を共有化することが難しいというシステム上の制約もあります。プラットフォームのチャット欄を閉じてしまい、[slido](https://www.slido.com/) のようなサービスを使うというのも解決策の1つではあります。

オンラインイベントの質疑応答だけでも厄介な課題があるなと、久しぶりにオンラインイベントに登壇して気付きを得たのが収穫でした。

## まとめ

空き時間のあるときに課題管理に関するツールやプロダクトの開発に時間を割いていこうと考えています。おそらく後で振り返ると Backlog-GitHub integration action はその第一歩とも言えるべきツールとなるでしょう。当社の強みの1つとして課題管理という分野に注力していこうと思います。
