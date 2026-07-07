---
title: "Amazon Aurora の Blue Green Deployment はマネージドな切り戻しができない"
emoji: "🛟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Aurora", "MySQL", "RDS"]
published: true
publication_name: "mixi"
---

## はじめに

Amazon Aurora MySQL のエンジンバージョンアップに [Blue/Green Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-overview.html) を使う検証をしました。

Blue/Green Deployments は、Aurora クラスターのエンジンバージョンアップやパラメータグループ変更を、短い停止時間で本番環境へ切り替えるための便利な機能です。
一方で、 **switchover 後に「やっぱり元の Aurora クラスターへ戻したい」となったときのマネージドな切り戻し機能はありません**。

AWS 公式ブログでも、switchover 後の rollback strategy として、ユーザー自身が旧クラスターへ binlog replication を張る手順が紹介されています。

https://aws.amazon.com/blogs/database/implement-a-rollback-strategy-after-an-amazon-aurora-mysql-blue-green-deployment-switchover/

この記事では、Aurora MySQL の Blue/Green Deployments を使ってみてわかったこと、特に switchover 後に切り戻しが必要になった場合に何が起きるのか、どのような手順を用意しておく必要があるのかをまとめます。

## Amazon Aurora の Blue Green Deployment について

Aurora の Blue/Green Deployments は、本番 DB クラスターを blue 環境として扱い、そのコピーである green 環境を作成する機能です。
AWS のドキュメントでは、green 環境に対してエンジンバージョンやパラメータグループの変更を行い、動作確認したうえで、本番向き先を green 環境へ切り替えるワークフローとして説明されています。

https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-overview.html

Aurora MySQL で Blue/Green Deployments を使う場合は、blue から green へ変更を反映するために binary logging が必要です。
[公式ドキュメント](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-creating.html)にある通り、基本的には `binlog_format=ROW` にしておけば使えます。

DB クラスターのレプリケーションと切り替えをフルマネージドで実行できるので、エンジンバージョンアップ時の作業をかなり楽にできます。

## Blue Green Deployment の使用例

切り戻しをしない通常の Blue/Green Deployments の流れを簡単に見ます。

Blue/Green Deployments を作成する前に、[公式ドキュメント](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-creating.html#blue-green-deployments-creating-preparing)に従って `binlog_format=ROW` と binlog retention period を設定しておきます。

```bash
# binlog_format=ROW は parameter group を変更して下さい。

# binlog retention period は mysql コマンドで変更します。
mysql \
  -uaurora_user \
  -p \
  -h example-cluster.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e "CALL mysql.rds_set_configuration('binlog retention hours', 24);"
```

事前準備ができたら、Aurora MySQL の global cluster `example-global-cluster` に対して、エンジンバージョンとパラメータグループを変更する Blue/Green Deployment リソースを作成します。

```bash
# create-blue-green-deployment のレスポンスで得られる BlueGreenDeploymentIdentifier
bgd_id="$(
  aws rds create-blue-green-deployment \
    --region ap-northeast-1 \
    --blue-green-deployment-name example-global-cluster-bgd \
    --source arn:aws:rds::123456789012:global-cluster:example-global-cluster \
    --target-engine-version 8.4.mysql_aurora.8.4.7 \
    --target-db-cluster-parameter-group-name example-cluster-pg-84 \
    --target-db-parameter-group-name example-instance-pg-84 \
    --query 'BlueGreenDeployment.BlueGreenDeploymentIdentifier' \
    --output text
)"
```

リソース作成が完了すると、Blue/Green Deployment 自体の status が `AVAILABLE` になります。
`describe-blue-green-deployments` では、切り替え元と切り替え先のクラスターを確認できます。

```bash
aws rds describe-blue-green-deployments \
  --region ap-northeast-1 \
  --blue-green-deployment-identifier "${bgd_id}"
```

```json
{
  "BlueGreenDeployments": [
    {
      "BlueGreenDeploymentIdentifier": "example-bgd",
      "BlueGreenDeploymentName": "example-global-cluster-bgd",
      "Status": "AVAILABLE",
      "SwitchoverDetails": [
        {
          "SourceMember": "arn:aws:rds:ap-northeast-1:123456789012:cluster:example-cluster",
          "TargetMember": "arn:aws:rds:ap-northeast-1:123456789012:cluster:example-cluster-green-abcdef",
          "Status": "AVAILABLE"
        }
      ]
    }
  ]
}
```

準備ができたら switchover します。

```bash
aws rds switchover-blue-green-deployment \
  --region ap-northeast-1 \
  --blue-green-deployment-identifier "${bgd_id}"
```

switchover が完了すると、`Status` は `SWITCHOVER_COMPLETED` になります。

```json
{
  "BlueGreenDeployments": [
    {
      "BlueGreenDeploymentIdentifier": "example-bgd",
      "Status": "SWITCHOVER_COMPLETED",
      "Source": "arn:aws:rds::123456789012:global-cluster:example-global-cluster-old1",
      "Target": "arn:aws:rds::123456789012:global-cluster:example-global-cluster",
      "SwitchoverDetails": [
        {
          "SourceMember": "arn:aws:rds:ap-northeast-1:123456789012:cluster:example-cluster-old1",
          "TargetMember": "arn:aws:rds:ap-northeast-1:123456789012:cluster:example-cluster",
          "Status": "SWITCHOVER_COMPLETED"
        }
      ]
    }
  ]
}
```

switchover 後、 **green 環境が元の `example-cluster` という名前になります**。
一方、旧 blue 環境は `example-cluster-old1` のような名前にリネームされます。

つまり、アプリケーションが参照している `example-cluster.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com` のような endpoint は、switchover 後には新しい green 環境を指すようになります。
アプリケーション側の DB 接続設定を変更すること無く switchover が実現できるのは Blue/Green Deployments の非常に便利なところです。

ちなみに、旧 blue 環境のネーミングの仕様はこうなっています。
> RDS renames the DB cluster and DB instances in the blue environment by appending -old `n` to the current name, where `n` is a number.
> https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-switching.html

問題がなければ、最後に Blue/Green Deployment を削除します。

```bash
aws rds delete-blue-green-deployment \
  --region ap-northeast-1 \
  --blue-green-deployment-identifier "${bgd_id}"
  
# 旧 blue 環境はこのとき削除されないので自分で消す
```

以上で、 Blue/Green Deployments によってマネージドな方法で DB クラスターのバージョンアップができました。

事前準備を済ませておけば、基本的には AWS CLI を順番に実行するだけなので大変便利です。

## Blue Green Deployment の切り戻し方法

Blue/Green Deployments は switchover まではかなり面倒を見てくれます。
しかし、switchover 後に「新しい green 環境で問題が見つかったので、元の blue 環境へ戻したい」となった場合、Blue/Green Deployments の API には「元に戻す」操作がありません。
つまり、切り戻しはマネージドなボタン一発ではなく、ユーザーが自分で手順を構築して実施する必要があります。

AWS 公式ブログでも、switchover 後の rollback strategy は、旧 blue 環境へ自分で replication を設定する手順として紹介されています。

https://aws.amazon.com/blogs/database/implement-a-rollback-strategy-after-an-amazon-aurora-mysql-blue-green-deployment-switchover/

このとき問題になるのは、主に以下です。

### ユーザーが自分で binlog replication を張る必要がある

switchover 後に旧 blue へ戻したい場合、旧 blue はその時点では古い環境です。
switchover 後に新しい本番環境へ書き込まれたデータは、旧 blue へは自動では戻りません。

そのため、新しい本番環境を source、旧 blue 環境を replica として binlog replication を張り、旧 blue を追いつかせる必要があります。

Aurora MySQL では `mysql.rds_set_external_source` や `mysql.rds_start_replication` を使って自力で replication 設定をすることになります。

### Global Database において endpoint は元に戻せない

switchover 後、元の cluster identifier と endpoint は新しい green 環境が引き継ぎます。
旧 blue 環境は `-old1` のような suffix が付いた別名になります。

「switchover 時に変更された endpoint の変更をもとに戻せば、アプリケーションの接続先を変えずに rollback できるのでは」と考えたくなります。
しかし、少なくとも [Global Database](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html) 配下の cluster では **マネージドな Aurora Blue/Green Deployment の操作以外で endpoint を変更することが基本的にできません**。

もしどうしても元の endpoint を旧 blue 側に戻すなら、global cluster 以外の DB cluster / DB instance などを全削除し、global cluster をリネームし、その後に DB cluster を再生成するような作業が必要になります。
これは rollback 手順としては重すぎますし、停止時間も大きくなります。

したがって rollback が発生した場合は、アプリケーションの接続先を `example-cluster-old1.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com` のような旧 blue 側 endpoint に向ける必要があります。

これはかなり大きな運用上の制約です。
マネージドな Blue/Green Deployments の switchover では、green 環境が元の endpoint を引き継ぐところまで AWS 側で実行してくれます。
しかし、自力で binlog replication を張って切り戻す場合、この endpoint の引き継ぎだけはどうやっても同じようには再現できません。
「Blue/Green Deployments で切り替えたから、失敗しても endpoint は元通りに戻せる」と考えていると危険です。

## Blue Green Deployment の切り戻し発生時の手順

実際に、switchover 後に問題が見つかり、旧 blue 環境へ切り戻す場合の手順を見ていきます。
この手順のほとんどは、以下の AWS 公式ブログの手順を参考にしていますが、後述するように一部この手順には不備があるため、独自の手順も含んでいます。

https://aws.amazon.com/jp/blogs/database/implement-a-rollback-strategy-after-an-amazon-aurora-mysql-blue-green-deployment-switchover/

なお、この手順はかなり長いので、読み飛ばして [#まとめ](#まとめ) まで飛んでも構いません。

### 1. switchover を完了させる

まず、「Blue Green Deployment の使用例」で紹介した手順と同じように、 Blue Green Deployment リソースを作成して switchover します。

```bash
# Blue Green Deployment リソースの作成手順は省略
# create-blue-green-deployment のレスポンスで得られる BlueGreenDeploymentIdentifier
bgd_id="example-bgd"

aws rds switchover-blue-green-deployment \
  --region ap-northeast-1 \
  --blue-green-deployment-identifier "${bgd_id}"
```

完了を確認します。

```bash
aws rds describe-blue-green-deployments \
  --region ap-northeast-1 \
  --blue-green-deployment-identifier "${bgd_id}" \
  --query 'BlueGreenDeployments[0].Status' \
  --output text
```

```text
SWITCHOVER_COMPLETED
```

この時点で、現行本番は `example-cluster`、旧 blue は `example-cluster-old1` になっています。

### 2. Aurora cluster event から switchover 時点の binlog position を取得する

binlog replication を始めるにあたり、 switchover 時点の binlog position を取得する必要があります。
AWS 公式ブログでは、「[switchover 後、 green クラスタの中で `show master status` を実行して binlog position を取得する](https://aws.amazon.com/jp/blogs/database/implement-a-rollback-strategy-after-an-amazon-aurora-mysql-blue-green-deployment-switchover/#:~:text=After%20the%20switchover%2C%20capture%20the%20binary%20log%20file%20name%20and%20position%20from%20the%20new%20blue%20environment)」と書かれています。

しかし、これでは switchover 実行〜 `show master status` を人間が実行するまでの間に green 環境で発生した書き込みが取りこぼされてしまいます。

これについては timee さんのブログ記事で同じ問題が指摘されており、 switchover 時点の binlog position は Aurora cluster event に記録されているので、そちらを使う方が安全なようです。

https://tech.timee.co.jp/entry/2024/09/17/100000

新しい本番 cluster に対して、switchover 時の binlog coordinates event を探します。

```bash
message="$(
  aws rds describe-events \
    --region ap-northeast-1 \
    --source-type db-cluster \
    --source-identifier example-cluster \
    --duration 60 \
    --query "reverse(sort_by(Events[?contains(Message, 'Binary log coordinates in green environment after switchover')], &Date))[0].Message" \
    --output text
)"

echo "$message"
```

以下のような message が得られます。

```text
Binary log coordinates in green environment after switchover: file mysql-bin-changelog.000123 and position 456789
```

この `mysql-bin-changelog.000123` と `456789` が、旧 blue へ replication を張るときの開始位置です。

パースして取り出します。

```bash
binlog_file="$(printf '%s\n' "$message" | sed -n 's/.*file \([^ ]*\) and position \([0-9][0-9]*\).*/\1/p')"
binlog_position="$(printf '%s\n' "$message" | sed -n 's/.*file \([^ ]*\) and position \([0-9][0-9]*\).*/\2/p')"

echo "binlog_file=${binlog_file}"
echo "binlog_position=${binlog_position}"
```

```text
binlog_file=mysql-bin-changelog.000123
binlog_position=456789
```


### 3. Blue/Green Deployment を削除する

Blue/Green Deployment を削除します。
これにより、Blue/Green Deployment が管理していた保護や replication 関係から離れます。

```bash
aws rds delete-blue-green-deployment \
  --region ap-northeast-1 \
  --blue-green-deployment-identifier "${bgd_id}"
```

### 4. 旧 blue クラスターを read_only にする

手順3で Blue/Green Deployment を削除すると、[旧 production DB cluster の read_only 設定が外れます](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-deleting.html)。

rollback をする場合、旧 blue クラスターに直接書き込まれると困るので、 旧 blue クラスターへの書き込みを止めます。
DB cluster parameter group の `read_only` を `1` にします。

```bash
aws rds modify-db-cluster-parameter-group \
  --region ap-northeast-1 \
  --db-cluster-parameter-group-name example-old-cluster-pg \
  --parameters "ParameterName=read_only,ParameterValue=1,ApplyMethod=immediate"
```

このように parameter group を手で操作するため、 parameter group は rollback 対象 DB 専用のものを使っておくと良いかもしれません。

### 5. replication user を用意する

新しい本番クラスターから旧 blue へ replication するためのユーザーを用意します。

```bash
mysql \
  -uaurora_user \
  -p \
  -h example-cluster.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e "CREATE USER 'repl_user'@'%' IDENTIFIED WITH mysql_native_password BY 'change-me';"

mysql \
  -uaurora_user \
  -p \
  -h example-cluster.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e "GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'repl_user'@'%';"
```

### 6. 旧 blue へ外部 source を設定して replication を開始する

旧 blue クラスターに接続し、新しい本番クラスターを source とする replication を設定します。

ここで指定する binlog file / position は、先ほど Aurora cluster event から取得した値です。

```bash
mysql \
  -uaurora_user \
  -p \
  -h example-cluster-old1.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e "CALL mysql.rds_set_external_source(
    'example-cluster.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com',
    3306,
    'repl_user',
    'change-me',
    'mysql-bin-changelog.000123',
    456789,
    0
  );"
```

replication を開始します。

```bash
mysql \
  -uaurora_user \
  -p \
  -h example-cluster-old1.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e "CALL mysql.rds_start_replication;"
```

### 7. replication が追いついたことを確認する

旧 blue 側で replica status を見ます。

```bash
mysql \
  -uaurora_user \
  -p \
  -h example-cluster-old1.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e 'SHOW REPLICA STATUS\G'
```

以下を確認します。

- `Replica_IO_Running: Yes`
- `Replica_SQL_Running: Yes`
- `Seconds_Behind_Source: 0`
- `Last_IO_Error` が空
- `Last_SQL_Error` が空

さらに、旧 blue の `Exec_Source_Log_Pos` と、新しい本番クラスターの `SHOW BINARY LOG STATUS` の position が一致していることを確認します。

旧 blue 側:

```bash
mysql \
  -uaurora_user \
  -p \
  -h example-cluster-old1.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e 'SHOW REPLICA STATUS\G' |
  grep '\(Source_Log_File:\|Exec_Source_Log_Pos:\)'
```

```text
Source_Log_File: mysql-bin-changelog.000123
Exec_Source_Log_Pos: 789012
```

現行クラスター側:

```bash
mysql \
  -uaurora_user \
  -p \
  -h example-cluster.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e 'SHOW BINARY LOG STATUS\G' |
  grep '\(File:\|Position:\)'
```

```text
File: mysql-bin-changelog.000123
Position: 789012
```

停止メンテナンス中などで DB への接続が完全にない状態で、両者の file / position が一致していれば、旧 blue が現行クラスターへ追いついたと判断できます。

### 8. 旧 blue の replication を停止する

旧 blue が追いついたら replication を止め、replication 設定自体も削除します。

```bash
mysql \
  -uaurora_user \
  -p \
  -h example-cluster-old1.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e "CALL mysql.rds_stop_replication;"

mysql \
  -uaurora_user \
  -p \
  -h example-cluster-old1.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -e "CALL mysql.rds_reset_external_source;"
```

### 9. 旧 blue を書き込み可能に戻す

rollback 先にする旧 blue で設定した `read_only` を解除します。

```bash
aws rds reset-db-cluster-parameter-group \
  --region ap-northeast-1 \
  --db-cluster-parameter-group-name example-old-cluster-pg \
  --parameters "ParameterName=read_only,ApplyMethod=immediate"
```

### 10. アプリケーションの接続先を旧 blue endpoint へ変更する

ここが Blue/Green Deployments の切り戻しでつらいところです。

旧 blue の endpoint は元の `example-cluster...` ではなく、`example-cluster-old1...` のような endpoint になっています。
そのため、アプリケーションや接続先生成設定を変更し、旧 blue endpoint を向かせる必要があります。

```text
before:
example-cluster.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com

after:
example-cluster-old1.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com
```

この変更を行ったうえで、アプリケーションを再開します。

### 11. Terraform state を整理する

rollback 後、RDS の実リソース名は元の Terraform コードが期待していた名前とずれています。

今回なら、 Terraform state では `example-cluster` が管理されていて、実際に本番で使っているのは `example-cluster-old1` です。

そのため、運用方針に応じて以下のような対応が必要になります。

- 旧 blue である `example-cluster-old1` を Terraform に import する
- 現行では使わなくなった `example-cluster` を削除対象として管理する
- アプリケーションの DB 接続先設定を IaC 側へ反映する
- engine version や parameter group の差分を整理する

この記事では具体的な Terraform の操作は省略します。


---

以上の 1〜11 の手順によって、switchover 後に旧 blue 環境へ切り戻すことができます。
正常系の手順が数回の AWS CLI 実行で済むのに対し、切り戻しはその何倍もの手間がかかることがわかるかと思います。

## まとめ

Aurora の Blue/Green Deployments は、Aurora MySQL のエンジンバージョンアップやパラメータ変更を短い停止時間で行うには便利な機能です。
特に、green 環境を事前に作り、本番 endpoint を switchover で切り替えられる点は強力です。

しかし、switchover 後に旧環境へ戻すマネージドな rollback 操作はありません。
切り戻しをするには、ユーザー自身が以下のような作業を行う必要があります。

- Aurora cluster event から switchover 時点の binlog file / position を取得する
- 新しい本番クラスターから旧 blue クラスターへ binlog replication を張る
- 旧 blue が追いついたことを確認する
- アプリケーションの接続先を `-old1` 側 endpoint へ変更する
- Terraform などの IaC state を整理する

つまり、Blue/Green Deployments は「切り替え」をかなりマネージドにしてくれますが、「切り戻し」は全くマネージドではありません。

本番で Aurora の Blue/Green Deployments を使うなら、switchover 手順だけでなく、切り戻しが発生したときの endpoint、binlog position、parameter group、replication user、IaC state まで含めて、事前に一通りの検証をして、手順を把握しておく必要がありそうです。

願わくば、 Aurora Blue/Green Deployments の機能に、 switchover 後に旧 blue 環境へ戻すマネージドな rollback 操作が追加されることを期待したいところです。
これが実装されればより安心して Blue/Green Deployments を使うことができると思います。
