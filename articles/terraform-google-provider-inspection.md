---
title: "Terraform の Google Provider の挙動が怪しいと思った時の調査手順と実際に起こった不具合の例"
emoji: "🌏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Terraform", "GoogleCloud", "GCP", "GoogleAPI"]
published: true
publication_name: "mixi"
---
ここ最近、Google Cloudに対して実行した `terraform apply` の挙動がおかしい事象に2件遭遇しました。
この記事では、Terraform の Google Provider の不具合を疑ったときの調査手順について汎用的に使えそうな手法をまとめ、実際にその方法で調査を行なった不具合の例を紹介します。

## 調査手順

### Terraform の挙動を確認する
terraform google provider が Google API に正しいリクエストを送っているかどうかを確認するには、 `TF_LOG=1 terraform apply` のように apply を実行して、リクエスト&レスポンス内容を確認すると良いです。
- TF_LOG について https://developer.hashicorp.com/terraform/internals/debugging

例えば、 `resource google_bigquery_table` の更新がある terraform 定義を `TF_LOG=1 terraform apply` すると、以下のようなログを見ることができます。

```txt
# [project] は実際のプロジェクト名が、[dataset], [table]にはBigQueryのデータセット名, テーブル名が入ります。

---[ REQUEST ]---------------------------------------
PUT /bigquery/v2/projects/[project]/datasets/[dataset]/tables/[table]?alt=json&prettyPrint=false HTTP/1.1
Host: bigquery.googleapis.com
User-Agent: google-api-go-client/0.5 Terraform/1.3.7 (+https://www.terraform.io) Terraform-Plugin-SDK/2.10.1 terraform-provider-google/4.48.0
Content-Length: 999
Content-Type: application/json
X-Goog-Api-Client: gl-go/1.18.1 gdcl/0.105.0
Accept-Encoding: gzip

{
 "schema": {
  "fields": [
   {
    "mode": "REQUIRED",
    "name": "timestamp",
    "type": "TIMESTAMP"
   },
   {
    "mode": "NULLABLE",
    "name": "data",
    "type": "string"
    }
   ]
 },
 "tableReference": {
  "datasetId": [dataset],
  "projectId": [project],
  "tableId": [table]
 },
 "timePartitioning": {
  "field": "timestamp",
  "requirePartitionFilter": true,
  "type": "DAY"
 }
}

---[ RESPONSE ]--------------------------------------
HTTP/2.0 200 OK
Alt-Svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000,h3-Q050=":443"; ma=2592000,h3-Q046=":443"; ma=2592000,h3-Q043=":443"; ma=2592000,quic=":443"; ma=2592000; v="46,43"
Cache-Control: private
Content-Type: application/json; charset=UTF-8
Date: Thu, 12 Jan 2023 09:32:36 GMT
Etag: e5wYpFYGtJ0s2ir44PqAuw==
Server: ESF
Vary: Origin
Vary: X-Origin
Vary: Referer
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 0

{
 "kind": "bigquery#table",
 "etag": "e5wYpFYGtJ0s2ir44PqAuw==",
 "id": "[project]:[dataset].[table]",
 "tableReference": {
  "datasetId": [dataset],
  "projectId": [project],
  "tableId": [table]
 },
 "schema": {
  [略]
 },
 [略]
}
```

このログを読むと、 `resource "google_bigquery_table"` の更新時は `PUT /bigquery/v2/projects/[project]/datasets/[dataset]/tables/[table]` API ( https://cloud.google.com/bigquery/docs/reference/rest/v2/tables/update ) を叩いていることがわかります。

terraform google provider の実装に問題がある場合は、ここで誤ったリクエストを叩いているのが見つかるはずです。

### Google API を直接叩く
Terraform google provider の API リクエストに問題がなく、 GoogleのAPIの挙動を疑いたいときは、 curl コマンドでAPIを直接叩いて挙動を確認するのが良いです。

以下のように、 `-H "Authorization: Bearer $(gcloud auth application-default print-access-token)"` のように認証ヘッダーをセットすると、自分のGoogleアカウントが権限を持っていればAPIレスポンスを確認することができます。
- 事前に [`gcloud auth application-default login`](https://cloud.google.com/sdk/gcloud/reference/auth/application-default/login)でGoogleアカウントにログインしておく必要がある。
- 認証ヘッダーについての説明: https://cloud.google.com/docs/authentication/rest#rest-request

GET API の例
```bash
curl -s -X GET \
    -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
    "https://[確認したいAPI]"
```
PUT APIの例
```bash
curl -s -X PUT \
    -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
    -H 'Content-Type: "application/json"' \
    "https://[確認したいAPI]"
    --data '{"msg": "foo"}'
```

Google API の挙動に問題がある場合は、リクエストを正しく処理できてない様子を確認できるはずです。


## terraform-google-provider & Google APIの不具合の例

ここからは、ここ最近で実際に起きた不具合の例を紹介していきます。


### 事例1: [稼働時間チェック](https://cloud.google.com/monitoring/uptime-checks/introduction)のサーバーの IP アドレスが `terraform apply` を実行するたびに変わる＆IPの重複が発生する (2023-01-10)

`data "google_monitoring_uptime_check_ips"` (稼働時間チェックの送信元IP一覧取得API) と `resource "google_compute_security_policy"` (Cloud Armorのセキュリティポリシー) を用いて、稼働時間チェックが通るような Cloud Armorを設定するような terraform 定義を以下のように書いていました。

```tf
data "google_monitoring_uptime_check_ips" "ips" {
}

locals {
  # ip_addressを含むstructのlistから、ソート済みなCIDRのlistに変換
  google_monitoring_uptime_check_cidrs_sorted = sort(
    [for ip_info in data.google_monitoring_uptime_check_ips.ips.uptime_check_ips : "${ip_info.ip_address}/32"]
  )
}
resource "google_compute_security_policy" "foo" {
  name        = "foo"

  dynamic "rule" {
    for_each = {
      #  chunklistで10個ずつのCIDRリストのリストに変換し（ rule.match.config.src_ip_ranges に最大10個までしかIPを登録できないため）
      #  index => ip_addresses の形のmapに変換している
      for idx, ip_addresses in chunklist(local.google_monitoring_uptime_check_cidrs_sorted, 10) :
      idx => ip_addresses
    }
    iterator = it
    content {
      action   = "allow"
      priority = tostring(1101 + it.key)
      match {
        versioned_expr = "SRC_IPS_V1"
        config {
          src_ip_ranges = it.value
        }
      }
      description = "Allow access from GCP uptime check ips (${it.key})"
    }
  }
  rule {
    action   = "deny(404)"
    priority = "2147483647"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    description = "All Deny"
  }
}
```
- https://registry.terraform.io/providers/hashicorp/google/latest/docs/data-sources/monitoring_uptime_check_ips
- https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_security_policy

この定義が、1月10日頃から **何も変更していないのに** `terraform apply` すると勝手に更新される事象が発生しました。

調査した結果、 `data "google_monitoring_uptime_check_ips"` で使っている `GET https://monitoring.googleapis.com/v3/uptimeCheckIps` API (https://cloud.google.com/monitoring/api/ref_v3/rest/v3/uptimeCheckIps/list )が **リクエストするたびに結果が変わり、IPに重複があるリストを返すことがある** ことがわかりました。
- 5回リクエストを投げると、1回は正常なレスポンスを返し、4回はIPに重複がある謎のリストを返すような状態だった

このことをGoogleのサポートに問い合わせたところ、「このAPIのバックエンドサーバーのうち、一部のリージョンのものがUSAのエントリを重複して返してしまう」という不具合が発生しているとのことで、この記事を書いている最中(2023-01-25)も問題が継続しています

::: details 不具合調査の詳細
以下のようなCloud Buildジョブ定義を実行すると再現します。
```yaml
steps:
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk:latest
    entrypoint: 'bash'
    args:
      - -c
      - |
        curl -s -X GET \
        -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
        "https://monitoring.googleapis.com/v3/uptimeCheckIps" > uptime-check-ips.json
        wc uptime-check-ips.json
```
これを5回実行すると、以下のようにサイズの異なるjsonが返ってくることがあります。
```bash
❯ for i in $(seq 5); do gcloud builds submit --config test.yaml 2>/dev/null | grep uptime-check-ips.json; done
 274  455 5723 uptime-check-ips.json
 274  455 5723 uptime-check-ips.json
 382  617 7779 uptime-check-ips.json
 382  617 7779 uptime-check-ips.json
 274  455 5723 uptime-check-ips.json
```

サイズの大きい方のレスポンスの中身を見てみると、
```json
{
  "uptimeCheckIps": [
    {
      "region": "USA",
      "location": "Iowa",
      "ipAddress": "146.148.59.114"
    },
// 中略
    {
      "location": "Iowa",
      "ipAddress": "146.148.59.114"
    },
// 中略
  ]
}
```
のように同一のIPが重複して入っている状態になっていました。


:::





### 事例2: BigQueryのカラムのdescriptionが更新できない (2023-01-13)
以下のような、カラム定義にネストがあるBigQuery テーブルを terraformで定義します。

```tf
resource "google_bigquery_table" "foo" {
  project = ...
  dataset_id = ...
  table_id = each.key
  schema = <-EOT
  {
    "fields": [
      {
        "name": "nested1",
        "type": "RECORD",
        "mode": "NULLABLE",
        "fields": [
          {
            "name": "col1",
            "type": "STRING",
            "mode": "NULLABLE",
            "description": "old col1 description"
          }
        ],
        "description": "old nested1 description"
      }
    ]
  }
EOT
```

この状態から、ネストの内側のカラム(col1)のdescriptionを変更すると、なぜかdescriptionが更新されませんでした。
```diff
resource "google_bigquery_table" "foo" {
  project = ...
  dataset_id = ...
  table_id = each.key
  schema = <-EOT
  {
    "fields": [
      {
        "name": "nested1",
        "type": "RECORD",
        "mode": "NULLABLE",
        "fields": [
          {
            "name": "col1",
            "type": "STRING",
            "mode": "NULLABLE",
-           "description": "old col1 description"
+           "description": "new col1 description" # この差分を terraform apply しても更新されない
          }
        ],
        "description": "old nested1 description"
      }
    ]
  }
EOT
```
調査した結果、`PUT https://bigquery.googleapis.com/bigquery/v2/projects/{projectId}/datasets/{datasetId}/tables/{tableId}` API (https://cloud.google.com/bigquery/docs/reference/rest/v2/tables/update )が、 **ネストしたカラム定義の内側のカラムのみ更新しようとすると正しく更新されない** 問題があることがわかりました。

このことをGoogleのサポートに問い合わせたところ、「USにおけるデータセットでは再現されず、"asia-northeast1"に作成されたデータセットでのみ再現する」とのことで、この記事を書いている最中も(2023-01-25)Google側で調査を行なっている状態です。


::: details 不具合調査の詳細

実際に `PUT https://bigquery.googleapis.com/bigquery/v2/projects/{projectId}/datasets/{datasetId}/tables/{tableId}` API を投げた様子

```bash
❯ curl -s -X PUT -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
  -H 'Content-Type: "application/json"' \                                                                   
  "https://bigquery.googleapis.com/bigquery/v2/projects/[project]/datasets/[dataset]/tables/[table]" \
  --data '{"schema": {
    "fields": [
      {
        "name": "nested1",
        "type": "RECORD",
        "mode": "NULLABLE",
        "fields": [
          {
            "name": "col1",
            "type": "STRING",
            "mode": "NULLABLE",
            "description": "new col1 description" # ←変更箇所
          }
        ],
        "description": "old nested1 description"
      }
    ]
  }}
'
# ↓ レスポンス
{
  "kind": "bigquery#table",
  "etag": "lhmkd/11gjez4zI9OGSXLA==",
  "id": "[project]:[dataset].[table]",
  "selfLink": "https://bigquery.googleapis.com/bigquery/v2/projects/[project]/datasets/[dataset]/tables/[table]",
  "tableReference": {
    "projectId": "[project]",
    "datasetId": "[dataset]",
    "tableId": "[table]"
  },
  "schema": {
    "fields": [
      {
        "name": "nested1",
        "type": "RECORD",
        "mode": "NULLABLE",
        "fields": [
          {
            "name": "col1",
            "type": "STRING",
            "mode": "NULLABLE",
            "description": "old col1 description" # ←変更されていない
          }
        ],
        "description": "old nested1 description"
      }
    ]
  },
  "numBytes": "0",
  "numLongTermBytes": "0",
  "numRows": "0",
  "creationTime": "1673519307663",
  "lastModifiedTime": "1673519742642",
  "type": "TABLE",
  "location": "asia-northeast1",
  "numTimeTravelPhysicalBytes": "0",
  "numTotalLogicalBytes": "0",
  "numActiveLogicalBytes": "0",
  "numLongTermLogicalBytes": "0",
  "numTotalPhysicalBytes": "0",
  "numActivePhysicalBytes": "0",
  "numLongTermPhysicalBytes": "0"
}
```

:::

### 事例3(参考): BigQueryのMaterialized Viewのクエリ定義が更新できない (2021-05-01)
事例1,2は両方 Google API 側が悪かった例であり、 terraform-google-provider側が悪かった例も示しておきたいので、少し古いですが2年前に発生した事例を紹介しておきます。

BigQueryの[Materialized view](https://cloud.google.com/bigquery/docs/materialized-views-intro)に以下のようなクエリの変更をすると…

```diff
resource "google_bigquery_table" "dm_monthly_first_logins" {
  dataset_id = "dest_dataset"
  table_id = "dest_table"
  materialized_view {
    enable_refresh = true
    query = <<-EOT
SELECT
  user_id,
- DATE_TRUNC(date, MONTH) momth, -- typoしてた
+ DATE_TRUNC(date, MONTH) month,
  min(date) monthly_first_login_date,
FROM src_project.src_dataset.src_table
GROUP BY 1,2
    EOT
  }
}
```
以下のようなエラーが発生します。
> Error: googleapi: Error 400: Schema update for materialized views is not supported., invalid

当時の terraform google provider は Materialized viewの更新時に `PATCH https://bigquery.googleapis.com/bigquery/v2/projects/{projectId}/datasets/{datasetId}/tables/{tableId}` API (https://cloud.google.com/bigquery/docs/reference/rest/v2/tables/patch )を使用していましたが、Materialized viewは更新できないためクエリを更新するときは destroy & createするのが Google API の正しい呼び方のようでした。

この問題は terraform-google-provider の Issue に報告し、すでに修正されています。
- https://github.com/hashicorp/terraform-provider-google/issues/8579
  - 修正PR: https://github.com/GoogleCloudPlatform/magic-modules/pull/4543

## まとめ
- terraform apply の挙動がおかしいときは、terraform providerとクラウドAPIのどちらが悪いか調査すると良い
- `TF_LOG=1 terraform apply によって、どのようなAPI呼び出しをしているかを確認することができる
- Google APIは、 `curl` コマンドの  `-H "Authorization: Bearer $(gcloud auth application-default print-access-token)"` オプションによって簡単に認証を通すことができる。
