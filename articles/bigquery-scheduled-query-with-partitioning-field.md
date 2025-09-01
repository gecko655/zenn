---
title: "BigQuery scheduled query では partitioning_field を指定できるが、公式ドキュメントがわかりにくい"
emoji: "😕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["BigQuery", "GCP", "Terraform"]
publication_name: "mixi"
published: true
---

公式ドキュメントが説明不足でつらい気持ちになったので、この記事を書くことにしました。

## BigQuery scheduled query とは

BigQuery scheduled query とは、 BigQuery 上でクエリをスケジュール実行し、その結果を BigQuery テーブルに投入する GCP のサービスです。
https://cloud.google.com/bigquery/docs/scheduling-queries

また、BigQuery Data Transfer Service とは、Cloud Storage Amazon S3, Azure Blob Storage, 等の自社・他社を問わないデータソースから BigQuery へデータをスケジュールで投入してくれる GCP のサービスです。
https://cloud.google.com/bigquery/docs/dts-introduction

これらは一見関係ないように見えますが、どちらも同一の API [projects.locations.transferConfigs.create](https://cloud.google.com/bigquery-transfer/docs/reference/datatransfer/rest/v1/projects.locations.transferConfigs/create) によって作成できることから、GCP 内部的には同じもののようです。

## やりたかったこと

BigQuery のクエリで得られるある データを、 BigQuery の 特定のテーブルに、投入タイムスタンプ付きで毎日投入していくようなジョブのことを考えます。
ジョブは様々な理由でたまに失敗することがあり、失敗するとその日の投入データが半分だけ中途半端に投入されてしまう場合があります。そのような状態になったら、その日の投入データを一旦削除してから同じデータを再投入する必要があります。

BigQuery には、timestamp 型が格納されているカラムを指定してパーティショニングを行なう[時間単位カラムパーティショニング](https://cloud.google.com/bigquery/docs/partitioned-tables#date_timestamp_partitioned_tables)という機能があります。
BigQuery scheduled query で1日分のデータを投入する際は、その日1日分のデータの入ったパーティションを削除してからデータ投入できるオプションがあると、ジョブを複数回実行したとしても得られる結果が同じになるという観点から（[冪等性](https://aws.amazon.com/jp/builders-flash/202104/serverless-idempotency/)）、便利そうです。


## BigQuery scheduled query で時間単位カラムパーティションの削除＆データの再投入をする方法

結果から言うと、時間単位カラムパーティションの削除＆データの再投入を BigQuery scheduled query で行なうには、 BigQuery Data Transfer Service の作成 API ( [projects.locations.transferConfigs.create](https://cloud.google.com/bigquery/docs/reference/datatransfer/rest/v1/projects.locations.transferConfigs/create) ) で以下を設定すればよいようでした。
- `destination_table_name_template` に `[table_name]${run_date}` のような文字列を指定する。
    - `[table_name]` は宛先のパーティショニングしたテーブルの名前に置き換えます。
    - `${run_date}` の部分はプレースホルダーではなく、そのまま `${run_date}` と書きます。
        - 参考: https://cloud.google.com/bigquery/docs/gcs-transfer-parameters
- `write_disposition` に `WRITE_TRUNCATE` を指定する。
    - データを削除してから再投入することを意味する。
- `params.partitioning_field` に、時間単位カラムパーティショニングでパーティショニング指定しているカラム名を指定する。

Terraform 定義[^2]で書くと以下のようになります。

[^2]: Terraform は実質的に Google Cloud API を叩いているだけで、 `google_bigquery_data_transfer_config` resource は   `projects.locations.transferConfigs.create` API と対応するので、ここでは Terraform の定義例を  API を叩く例の代わりとしています。

```tf
resource "google_bigquery_data_transfer_config" "example" {
  // 関係ない設定値は省略しています
  
  data_source_id         = "scheduled_query"

  params = {
    destination_table_name_template = "table_name$${run_date}"
    write_disposition               = "WRITE_TRUNCATE"
    partitioning_field              = "imported_timestamp" // ← これ
    query                           = <<-EOT
      SELECT
        @run_time AS imported_timestamp,
        **** /* 省略 */
      FROM
        ****
      EOT
  }
  schedule = "every day 01:00" // UTC = 10:00 JST
}

resource "google_bigquery_table" "destination_table" {
  // 関係ない設定値は省略しています

  table_id = "table_name"

  time_partitioning { // ← 宛先テーブルは時間単位カラムパーティショニングである必要がある。
    field = "imported_timestamp" // ← field を指定することで時間単位カラムパーティショニングになる。 https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/bigquery_table#field-1
    type  = "DAY"
  }

  schema = jsonencode([
    {
      name: "imported_timestamp",
      type: "TIMESTAMP",
      mode: "REQUIRED",
      description : "BigQuery にデータを取り込んだ時刻"
    },
    ...
  ])
}
```
ここで、もし `params.partitioning_field` オプションを付けないと、以下のように
「BigQuery scheduled query で指定されている宛先テーブルは[時間単位カラムパーティショニング](https://cloud.google.com/bigquery/docs/partitioned-tables#date_timestamp_partitioned_tables)だが、 転送の宛先パーティションの設定は[取り込み時間パーティショニング](https://cloud.google.com/bigquery/docs/partitioned-tables#ingestion_time)が宛先のときのものになっている」という旨のエラーが発生します。

> Incompatible table partitioning specification. Destination table exists with partitioning specification interval(type:DAY,field:imported_timestamp), but transfer target partitioning specification is interval(type:DAY,field:). Please retry after updating either the destination table or the transfer partitioning specification.

## `params.partitioning_field`  オプションのドキュメントが……

というわけで、パーティションに指定しているカラムを BigQuery scheduled query 設定の `params.partitioning_field` に指定したとき、かつそのときに限り、時間単位カラムパーティショニングなテーブルに対して冪等なジョブが定義できることがわかりました。

しかし、この `params.partitioning_field` オプションの公式ドキュメントはとてもわかりにくいです。

まず、 API ドキュメントには、「`params` オプションの設定はガイド記事を読め」と書いてあります。

![](/images/bigquery-scheduled-query-with-partitioning-field/1.png)
> Parameters specific to each data source. For more information see the bq tab in the 'Setting up a data transfer' section for each data source.

https://cloud.google.com/bigquery/docs/reference/datatransfer/rest/v1/projects.locations.transferConfigs#TransferConfig

そして、ここで読めと言われていると思われる BigQuery scheduled query のガイド記事を読むと、いくつか partitioning_field に関する記述があります。
https://cloud.google.com/bigquery/docs/scheduling-queries

しかし、 **その記述はどれもほぼまともな説明をしていません…**
- Web console の例
  ![](/images/bigquery-scheduled-query-with-partitioning-field/2.png)
    - Destination table partitioning field という入力項目がある
    - ここに何を入力すればいいのかは説明されていないが、「宛先テーブルでパーティショニングに指定されたカラム名を入力すればいいのかな〜」くらいのことは類推できそう。
- `bq` コマンドの例
  ![](/images/bigquery-scheduled-query-with-partitioning-field/3.png)
  ↑ 「**取り込み時間パーティショニングテーブル** で partitioning_field を指定するとエラーが発生する」 （エラー発生しないのはどんなときかは一切書かれていない）
- API の例
  ![](/images/bigquery-scheduled-query-with-partitioning-field/4.png)
  ↑「API ドキュメントを見ろ」（API ドキュメントにはガイド記事を見ろと書いてあったが…？？？[たらい回し](https://dic.nicovideo.jp/a/%E3%81%9F%E3%82%89%E3%81%84%E5%9B%9E%E3%81%97)か？？？）

- Java, Python の例
  ![](/images/bigquery-scheduled-query-with-partitioning-field/5.png)
  ![](/images/bigquery-scheduled-query-with-partitioning-field/6.png)
  ↑空文字が設定されている例しかなく、具体的な値が指定されている例がない

また、以上で紹介した記述以外に、公式ドキュメントは見つかりませんでした。

というわけで、 `partitioning_field` というオプションが時間単位カラムパーティショニングなテーブルで使えることに気づくためには
- ドキュメントの bq, Java, Python の部分から `partitioning_field` という **オプションが存在することに気づく**
- ドキュメントの **Web Console の部分から**、この設定が時間単位カラムパーティショニングなテーブルのパーティション設定に使うっぽいことを **類推する**

という事が必要なようです。

実際、私がこの機能が存在することに気づいたときには、この機能がほしいと思ったときから3ヶ月経過していました……

また、ググった感じこの仕様について言及しているのは https://github.com/hashicorp/terraform-provider-google の issue ( [#5317](https://github.com/hashicorp/terraform-provider-google/issues/5317) ) だけで、かつ `params.partitioning_field`  のことを `params.partition_field` と書いています（ "ing" が足りない）[^1]。

この機能自体、だれも理解して使っていないのかもしれません……


[^1]: もしかしたら、ドキュメントに書いてないだけで `params.partitioning_field`  の代わりに `params.partition_field`  を指定しても動くのかもしれません。面倒なので調べていませんが……


