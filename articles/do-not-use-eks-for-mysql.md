---
title: "MySQL 用 pod に EKS Fargate を採用してはいけない"
emoji: "🐬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["efs", "eks", "fargate", "kubernetes", "mysql"]
published: true
publication_name: "mixi"
published_at: 2023-12-09 00:00
---

## 3行で

- AWS EKS には、自分で node の管理をする「[普通の managed node](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/managed-node-groups.html)(以下、managed nodeと呼びます)」と、 1podあたりnodeずつ自動で用意される「[Fargate node](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/fargate.html)」が存在する。
- Fargate node では、Persistent Volume の作成に大きな制限があり、ボリュームの実体に[EFS](https://aws.amazon.com/jp/efs/)しか選ぶことが出来ない。
- EFSは[EBS](https://aws.amazon.com/jp/ebs/)と比べ、大量の小さいファイルを読み書きするときの性能がかなり悪く、managed node + EBS と比べ Fargate + EFS の MySQL podの **読み書き性能が 1/25 くらいになってしまった**。諦めて managed node + EBS を使うべき。


## EKS の node について

AWS EKS にはいくつかの種類の node が存在します。
- Managed: https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/managed-node-groups.html
  - EKS側の設定によって自動でEC2サーバーが立ち、これを k8s node として使用できるやつ
- Self-managed: https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/worker.html
  - EC2 サーバーを自分で立てて管理し、これを k8s node として使用できるやつ
- Fargate: https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/fargate.html
  - pod ごとに独自の Fargate 仮想マシンが自動で起動し、これを k8s node として使用できるやつ
  - https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/what-is-fargate.html
  - "Fargate" とだけ検索すると EKS Fargate と [ECS Fargate](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/what-is-fargate.html)が両方出てきて厄介。（この記事では一貫して EKS Fargate についての話をします。）

Fargate node は managed, self-managed と異なり、 **いくら pod が増えてもインフラリソースが勝手に増える** という点で便利です。
一方で、すぐに分かる欠点として以下のようなものがあります。

-  EKS Fargate node で起動した pod の起動が遅い。
    - 約1分ほどかかる
        - [ECS fargateは30秒くらいで起動するらしい](https://michimani.net/post/aws-how-long-does-fargate-take-to-start-2nd/)が、EKS Fargate はそれよりかなり遅い
    - pod の新規作成時だけでなく、すでに起動中の pod を更新する際も毎回新しいインフラリソースをプロビジョニングしているようで、毎回余計に1分ほど待たされることになる
        - ゲームのマスターデータを編集＆反映するのに pod の再起動が必要なタイプのプロジェクトだと、毎回1分余計に待たされるのはかなりつらい
        - https://github.com/aws/containers-roadmap/issues/649 でなんとかしろと言われているが、1年ほどレスポンスがない
    - ちなみに、Fargate pod の起動高速化についてググると[こういう記事](https://aws.amazon.com/jp/blogs/containers/reducing-aws-fargate-startup-times-with-zstd-compressed-container-images/)が見つかるが、これは docker image のサイズ圧縮に関する記事で、インフラリソースのスピンアップ時間の話ではなかった。そういうことじゃないんだ……
- 意図せずインフラ費用が高額になる可能性がある
    - いらなくなった pod を消し忘れて残したままにしても、インフラリソースが枯渇することがないので気づきにくく、インフラ費用を払い続けることになってしまう


MIXIの某ゲームプロジェクトでは、ゲームのサーバー実装の開発やデータの設計を各担当者が円滑に行えるようにするために、開発環境用サーバーを各担当者や開発案件ごとに1つずつ立てたり消したりすることがプロジェクトメンバーなら誰でもできるようにしています。
このバックエンドに EKS Fargate を用いることで、インフラ管理者が EKS インフラの枯渇を気にしなくても勝手にインフラリソースが増減するようにしています。

## Fargate の Persistent Volume について

AWS EKS の managed node では、 [Persistent Volume](https://kubernetes.io/ja/docs/concepts/storage/persistent-volumes/)にEBS, EFS などのストレージが使えます。特別な要件がなければ、EBSを使うのが無難そうです。
- EKS managed node で使えるストレージオプションについて: https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/storage.html

しかし、 Fargate ではなぜか EBS が使えません。
> Amazon EBS ボリュームを Fargate Pods にマウントすることはできません。
> https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/fargate.html

> **注:** Windows ワーカーノードと AWS Fargate は EBS CSI ドライバーをサポートしていません。
> https://repost.aws/ja/knowledge-center/eks-troubleshoot-ebs-volume-mounts

Fargate で EBS が使えない理由は不明ですが、おそらくEBSはEC2のインスタンスレベルでアタッチすることは出来ても、EC2インスタンス内に仮想化されたFargate nodeにアタッチすることができないっぽいです（特に理由がどこかで説明されているわけではないのでなにもわからない）。

そのため、永続化したディスクが必要な MySQL pod を Fargate で起動するためには、EFS ボリュームをマウントした上で MySQL のデータディレクトリを EFS 上に保存する必要があります。

## Fargate + EFS の性能について

EFS を「サーバーアプリケーションが激しく読み書きするボリューム」に使うことに違和感を覚えつつも、実際に MySQL pod を Fargate + EFS で MySQL の pod を立ててみました。
機能面では想定通りに動作し、MySQL pod を再起動しても MySQL のデータが揮発せずに永続化出来ていることが確認できました。

しかし、機能は想定通りだったものの、 **明らかに処理速度が MySQL らしからぬ遅さをしていました**。

そこで、 Fargate + EFS と managed node + EBS の2つの環境をそれぞれ用意し、ファイル読み書きの速度を計測＆比較してみたところ、 Fargate + EFS は managed node + EBS と比べ **およそ25倍処理が遅い**ことがわかりました。

**Fargate + EFS**:
469MB のファイルの読み込みに6.7秒 -> 約70MB/s

```bash
# mysql のデータを読み込む時間をshell上で計測する
bash-4.2# time cat /var/lib/mysql/some_path/* | wc -c
469179049

real    0m6.696s
user    0m0.020s
sys    0m0.782s
```
**managed node + EBS**:
469MB のファイルの読み込みに0.26秒 -> 約1800MB/s

```bash
# mysql のデータを読み込む時間をshell上で計測する
bash-4.2# time cat /var/lib/mysql/some_path/* | wc -c
468097705

real    0m0.265s
user    0m0.018s
sys    0m0.295s
```
（若干サイズが違うのはデータの用意が雑で Fargate + EFS のほうがユーザーデータが若干大きいためです。気にしないでください）

これに関して AWS のサポートに質問したところ、以下のような回答を得ました。
- EFSのパフォーマンスは、**サイズの小さな大量のファイルが存在している場合に悪くなる**。
- Fargate で Persistent Volume を使う際は EFS 以外の選択肢は存在しない。
- 以上のことから **MySQL pod で Fargate を使うことは現時点ではおすすめできない**。 managed node + EBS での構築が適切である。

EFS で小さなファイルの読み書き性能が悪くなる点については、 https://docs.aws.amazon.com/ja_jp/efs/latest/ug/performance.html#optimize-open-close の説明が詳しいです。

> 小さなファイルの読み取りや書き込みを行う場合、2 回のラウンドトリップが余分に必要になります。
1 回の往復 (ファイルオープン、ファイルクローズ) には、メガバイトの大容量データの読み取りまたは書き込みと同じくらいの時間がかかる場合があります。
> https://docs.aws.amazon.com/ja_jp/efs/latest/ug/performance.html#optimize-open-close

したがって、 MySQL 等のデータベース pod （及び 大量のファイルを Persistent Volume に読み書きしなければいけないような pod） を EKS で立てる際には、 Fargate は採用できず managed node + EBS で構築する必要があるようでした。


## まとめ

開発環境の pod 一式をすべて Fargate node に載せることで EKS managed node の管理から完全に解放されると思っていたのですが、残念ながら MySQL pod において性能が全然出ず、やむを得ず MySQL pod のみ EKS managed node での管理をすることになってしまいました。
MySQL 用 managed node が枯渇してきたら managed node 数を1個増やす等の作業をしなければいけないようです。

今後の EKS Fargate の発展に期待したいところです。もし Fargate が EBS に対応したり、何らかの別の方法で開発環境の MySQL pod を完全マネージドなインフラに移行できたら、来年のアドベントカレンダーに書こうかなと思っています。

