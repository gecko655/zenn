---
title: "BigQuery等に取り込まれたデータの品質を `dbt test` でチェックし、いい感じのテストレポートを作る"
emoji: "🪣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dbt", "DataQuality", "データ品質", "BigQuery", "データエンジニアリング"]
published: true
publication_name: "mixi"
---

## この記事について3行くらいで
- あるプロジェクトで BigQuery のテーブル＆ビューを [dbt](https://www.getdbt.com/) で管理しており、そのデータの品質チェックに `dbt test` コマンドを使うことを考えます。
  - ここでは、データウェアハースに格納されているデータの間違いの少なさのことを「[データ品質(data quality)](https://ja.wikipedia.org/wiki/%E3%83%87%E3%83%BC%E3%82%BF%E5%93%81%E8%B3%AA)」と呼んでいます。
- `dbt test` は、データ品質に問題があることをテストしてくれますが、具体的にどのデータに問題があるかを標準出力する機能がなく、原因究明の際に再度データの調査が必要で不便です。
- この記事では `dbt test` と `dbt show` を使った、データ品質テストの実現方法と実際にできたテストレポートの中身を紹介します。

## dbt test について

以下のようなdbt テーブル定義があるときのことを考えます。(先程のテーブル定義と同様です)

```yml
# models/mart/example_table.yml
# BigQuery ビューのメタ情報
version: 2

models:
  - name: example_table
    description: 説明用テーブル
    columns:
      - name: col_a
        description: 説明用カラム
        tests:
          - unique:
              name: example_table.col_aはユニークであるべき
```

```sql
-- models/mart/example_table.sql
-- BigQuery ビュー定義 （テスト用なので適当です）
SELECT 1 AS col_a
UNION ALL
SELECT 1 AS col_a
UNION ALL
SELECT 2 AS col_a
```


以上の2つのファイルを作成し、 `dbt run` を実行すると BigQueryビューができます。
（BigQueryへの接続設定ファイル等も準備する必要がありますが、ここでは説明は省略しています。）


また、この状態で `dbt test` を実行すると、カラム `col_a` には [unique](https://docs.getdbt.com/reference/resource-properties/tests#unique) のテストが設定されているため、 `col_a` のデータに重複がないことのテストが行われます。

成功時：
```bash
❯ dbt test -t dev
Running with dbt=1.5.0
Found 1 models, 1 tests, 0 snapshots, 0 analyses, 0 macros, 0 operations, 0 seed files, 1 sources, 0 exposures, 0 metrics, 0 groups

Concurrency: 4 threads (target='dev')

1 of 1 START test example_table.col_aはユニークであるべき ............................ [RUN]
1 of 1 PASS example_table.col_aはユニークであるべき.................................. [PASS in 0.74s]

Finished running 1 tests

Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

失敗時
```bash
❯ dbt test -t dev
Running with dbt=1.5.0
Found 1 models, 1 tests, 0 snapshots, 0 analyses, 0 macros, 0 operations, 0 seed files, 1 sources, 0 exposures, 0 metrics, 0 groups

Concurrency: 4 threads (target='dev')

1 of 1 START test example_table.col_aはユニークであるべき ............................ [RUN]
1 of 1 FAIL example_table.col_aはユニークであるべき .................................. [FAIL in 0.74s]

Completed with 1 error and 0 warnings:

Failure in test example_table.col_aはユニークであるべき(models/mart/example_table.yml)
  Got 1 result, configured to fail if != 0

  compiled Code at target/compiled/project_name/models/mart/example_table.yml/example_table.col_aはユニークであるべき.sql

Done. PASS=0 WARN=0 ERROR=1 SKIP=0 TOTAL=1
```

テスト失敗時は、そのテストで使用されたクエリも同時に出力されます。 `unique` のテストで使用されたデータチェックのクエリはこんな感じでした。
```sql
-- target/compiled/project_name/models/mart/example_table.yml/example_table.col_aはユニークであるべき.sql
with dbt_test__target as (

  select col_a as unique_field
  from `gcp_project`.`bq_dataset`.`example_table`
  where col_a is not null

)

select
    unique_field,
    count(*) as n_records

from dbt_test__target
group by unique_field
having count(*) > 1
```

ちなみに、dbt には `unique` の他にも `relationships` (あるカラムにある値は他のテーブルのカラムにすべて存在する)等のいくつかのgeneric testが定義済みで、すぐに使えるようになっています。
- generic test の一覧  https://docs.getdbt.com/reference/resource-properties/tests#out-of-the-box-tests

## dbt test の問題点と dbt show について

`dbt test` では、テストが失敗したこととテストに使ったSQLの内容は教えてくれますが、テストに失敗した原因となったデータについては教えてくれません。

- 前節の例だと、 `col_a` に `1` という値が複数行格納されてますよ！ と言ってほしい。
- 不正なデータを、[不正なデータ用BigQueryテーブルを作って格納してくれる機能](https://docs.getdbt.com/reference/resource-configs/store_failures)はあるが、BigQueryテーブルをいちいち見に行かないといけないのが面倒。できれば先頭数行を標準出力に出してほしい。

このような「 `dbt test` でテストが落ちたときに不正なデータを標準出力する」機能について、GitHub上 で dbt のメンテナの方と議論していたのですが、 `dbt test` コマンドと  `dbt show` コマンドを組み合わせれば問題解決できることに気づきました。
https://github.com/dbt-labs/dbt-core/discussions/8407

- [dbt show](https://docs.getdbt.com/reference/commands/show) は dbt 1.5.0 の新機能
- [dbt 1.5.0](https://github.com/dbt-labs/dbt-core/releases/tag/v1.5.0) は2023年4月リリース

`dbt show` コマンドは、1つのmodelまたはtestを指定し、以下を行ないます。
- modelを指定したときはそのmodelへのSELECTクエリを、testを指定したときはそのtestのSQLクエリを実行する。
- ↑で実行したSQLクエリで**得られたデータの先頭数行を標準出力に表示する**
    - デフォルトでは5行

使用例:
```bash
# dbt run コマンドの実行結果を表示する
❯ dbt show -t dev --select example_table 
Running with dbt=1.5.0
Found 1 models, 1 tests, 0 snapshots, 0 analyses, 0 macros, 0 operations, 0 seed files, 1 sources, 0 exposures, 0 metrics, 0 groups

Concurrency: 4 threads (target='dev')

Previewing node 'example_table':
| col_a |
| ----- |
|     1 |
|     1 |
|     2 |
```

```bash
# dbt test コマンドの実行結果を表示する
# test を dbt show するときはテスト名をそのまま --select オプションに渡す
❯ dbt show -t dev --select example_table.col_aはユニークであるべき
Running with dbt=1.5.0
Found 1 models, 1 tests, 0 snapshots, 0 analyses, 0 macros, 0 operations, 0 seed files, 1 sources, 0 exposures, 0 metrics, 0 groups

Concurrency: 4 threads (target='dev')

Previewing node 'example_table.col_aはユニークであるべき':
| unique_field | n_records |
| ------------ | --------- |
|            1 |         2 |

```

この例では、 「"1" という値が2回出現しているせいでテストに失敗している」旨が表示されています。

これで、簡単なコマンドによって失敗したテストの原因の値を得られることがわかりました。
これを使ってCIレポートとして使えるような出力になるよう shell 芸をすれば、テストが実現できそうです。

## 作ったもの

最終的にできたスクリプトはこのような形になりました。
- dbt は jsonでログを吐けるので、 `jq` を使って `dbt test` の出力を解釈し、落ちたテストの名前を取得しています。
- `dbt test` 終了後に、落ちたテストに対して `dbt show --select [テスト名]` を実行して、その出力のうちデータの部分のみを `jq` で取り出して表示しています。
```bash
# Delete previous log if exists
rm -f logs/dbt.log

# Execute the test and show human readable output while output json log file
if dbt --log-format-file json --no-use-colors test -t dev ; then
  echo success
  exit 0
fi

# テストに失敗したので、失敗したテストのデータを表示する
echo 'データ品質テストに失敗しました。テストに失敗したデータを表示します'

# Extract the failed test node names from log file
# Z022 is `RunResultFailure` log
# https://github.com/dbt-labs/dbt-core/blob/5814928e3836467ffa90297e871d8a99f8c8b855/core/dbt/events/types.py#L2121-L2128
failed_test_names="$(cat logs/dbt.log | jq -r 'select(.info.code == "Z022") | .data.node_name')"

# Show failed values for each failed tests
# Q041 is `ShowNode` log
# https://github.com/dbt-labs/dbt-core/blob/5814928e3836467ffa90297e871d8a99f8c8b855/core/dbt/events/types.py#L1859-L1875
for failed_test_name in $(echo $failed_test_names);  do
  echo '-----'
  echo "'${failed_test_name}'のテストに失敗したデータ:"
  dbt --log-format json --no-use-colors show -t dev -s "$failed_test_name" \
    | jq -r 'select(.info.code == "Q041") | .data.preview'
done
exit 1
```

出力はこんな感じです

```bash
Running with dbt=1.5.0
Found 1 models, 1 tests, 0 snapshots, 0 analyses, 0 macros, 0 operations, 0 seed files, 1 sources, 0 exposures, 0 metrics, 0 groups

Concurrency: 4 threads (target='dev')

1 of 1 START test example_table.col_aはユニークであるべき ............................ [RUN]
1 of 1 FAIL example_table.col_aはユニークであるべき .................................. [FAIL in 0.74s]

Completed with 1 error and 0 warnings:

Failure in test example_table.col_aがユニークではありません (models/mart/example_table.yml)
  Got 1 result, configured to fail if != 0

  compiled Code at target/compiled/project_name/models/mart/example_table.yml/example_table.col_aがユニークではありません.sql

Done. PASS=0 WARN=0 ERROR=1 SKIP=0 TOTAL=1
データ品質テストに失敗しました。テストに失敗したデータを表示します
-----
'example_table.col_aはユニークであるべき'のテストに失敗したデータ:
| unique_field | n_records |
| ------------ | --------- |
|            1 |         2 |
```

これで、 `dbt test` でテストが落ちた原因をCIのテストレポートに表示できることようになりました。
また、エラーメッセージを日本語で書けるので、このままデータ入力担当者に共有してデータを直してもらうことができるようになりました。

その後、このテストを元データテーブルが更新されるたびに実行する定期実行ジョブを[Cloud Run Jobs](https://cloud.google.com/run/docs/overview/what-is-cloud-run#jobs)などを使って作成して、データ品質に問題があるときにアラートを出せるようにしました。
（Cloud Run Jobsの説明は長くなってしまうためここでは省略します）

## まとめ
`dbt show` コマンドが4ヶ月前のリリースで追加されていたおかげで、テストに落ちた原因の値が標準出力できるようになりました。
仮に `dbt show` がなければ、 `dbt test` による品質テストを諦めて他のソリューションを使うこと(例えば[BigQuery Scheduled Query](https://cloud.google.com/bigquery/docs/scheduling-queries)でクエリを定期実行するなど)を考えようかなと思っていたのですが、これで dbt でデータスキーマとデータ品質テストの管理が単一のファイルで管理できるようになったので良かったです。
また、タイトルや説明では BigQuery のビューを対象として説明をしましたが、 dbt が対応している他の 分析 DB でも同様な方法が使えるのではないかと思います。(BigQuery Scheduled Query では他の分析 DB に適用できなかったはず。)
本当は shell 芸なしでテスト結果のデータを表示する機能が欲しいですが、 dbt の今後に期待したいと思います。

めでたしめでたし。

## 付録
- dbt test 全般について https://docs.getdbt.com/docs/build/tests
    - この記事では `not_null` `relationships` 等の "generic test" しか例に出しませんでしたが、 `dbt test` では任意のSQLが使えます( "singular test" )