---
title: "WITH RECURSIVE 句を含んだ BigQuery view は作らないほうが良い"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["BigQuery", "SQL", "dbt"]
publication_name: "mixi"
published: true
---

## 3行で

- BigQuery の [WITH RECURSIVE 句](https://cloud.google.com/bigquery/docs/recursive-ctes)は、再帰計算ができて便利
- `WITH RECURSIVE` 句を [テーブル関数](https://cloud.google.com/bigquery/docs/table-functions) 内で使うことは仕様上できない
- `WITH RECURSIVE` 句を含む `VIEW` を作ってしまうと、その `VIEW` は BigQuery のテーブル関数から参照できなくなる

## WITH RECURSIVE 句とは

https://cloud.google.com/bigquery/docs/recursive-ctes

例えば、以下のような「従業員とその直属の上司の一覧」のテーブルがあったとします。

| emp_id | name  | manager_id |
|--------|-------|------------|
| 1      | Alice | NULL       |
| 2      | Bob   | 1          |
| 3      | Carol | 2          |
| 4      | Dave  | 2          |
| 5      | Eve   | 3          |

このデータから、「Alice は Eve の上司のうちの一人」であることを確認するには、Eve から上司をたどっていく必要があります。
このようなとき、`WITH RECURSIVE` 句で再帰的なサブクエリを書くのが便利です。

```sql
WITH RECURSIVE employees AS (
  SELECT 1 AS emp_id, 'Alice' AS name, NULL AS manager_id UNION ALL
  SELECT 2, 'Bob',   1 UNION ALL
  SELECT 3, 'Carol', 2 UNION ALL
  SELECT 4, 'Dave',  2 UNION ALL
  SELECT 5, 'Eve',   3
),

-- 再帰的に上司をたどり、配列に蓄積
manager_paths AS (
  -- 基点: 各従業員から開始（最初は自分自身の上司リストは空）
  SELECT
    emp_id,
    name,
    manager_id,
    ARRAY<STRING>[] AS manager_list
  FROM employees

  UNION ALL

  -- 再帰ステップ: 上司を配列に追加していく
  SELECT
    e.emp_id,
    e.name,
    e.manager_id,
    ARRAY_CONCAT(m.manager_list, [m.name]) AS manager_list
  FROM manager_paths m
  JOIN employees e
    ON e.manager_id = m.emp_id
)

-- 最終的に各従業員の「すべての上司リスト」を表示
SELECT
  emp_id,
  name,
  manager_list
FROM manager_paths
-- 最も上司を多く記録したレコード以外（計算途中のデータ）を捨てる
QUALIFY
  ROW_NUMBER() OVER (
    PARTITION BY emp_id
    ORDER BY ARRAY_LENGTH(manager_list) DESC
  ) = 1
ORDER BY emp_id;
```

実行すると、各従業員名とその上司が一覧になったテーブルが得られます。

| emp_id | name  | manager_list              |
|--------|-------|---------------------------|
| 1      | Alice | []                        |
| 2      | Bob   | ["Alice"]                 |
| 3      | Carol | ["Bob", "Alice"]          |
| 4      | Dave  | ["Bob", "Alice"]          |
| 5      | Eve   | ["Carol", "Bob", "Alice"] |

## WITH RECURSIVE 句はテーブル関数で使えない

> WITH RECURSIVE isn't allowed in functions.

https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#cte_general_rules

`WITH RECURSIVE` 句を含むクエリは、BigQuery の関数内に含めることはできません。

すなわち、先程のクエリを[テーブル関数](https://cloud.google.com/bigquery/docs/table-functions)化して、「ある従業員の ID を引数として、その上司のデータをすべて返す関数」を書くことはできません。

例えば、以下のようなクエリを書くと

```sql
CREATE OR REPLACE TABLE FUNCTION [dataset].find_all_managers(emp_id INT64)
AS (
WITH RECURSIVE
/*
  省略（再帰を含むクエリ）
*/
);
```

![](/images/bigquery-do-not-use-with-recursive-in-view/1.png)

以下のように怒られてしまいます。

> WITH RECURSIVE is only allowed at the top level of the SELECT, CREATE TABLE AS SELECT, CREATE VIEW, INSERT, EXPORT DATA statements.

この仕様は、実際に関数から呼び出そうとした時に初めて意識することが多く、`VIEW` を定義した時点では見落としやすいです。

## BigQuery view で WITH RECURSIVE 句を使うと……

BigQuery の関数内に `WITH RECURSIVE` があってはならない制約は、`VIEW` の中のどこかに `WITH RECURSIVE` が入っていても同様です。多段に `VIEW` が定義されている場合では、その `VIEW` 群のどこか 1 箇所でも `WITH RECURSIVE` が入っていたら、関数内からその `VIEW` は呼び出せません。

- [dbt](https://www.getdbt.com/) を使っている場合、 dbt の [view materialization](https://docs.getdbt.com/docs/build/materializations#view) で多段な `VIEW` をモデル定義していると、`VIEW` が他の `VIEW` を大量に参照することがよくありますが、そのどこか 1 箇所でも `WITH RECURSIVE` が混入すると、その `VIEW` は関数で使えなくなります。

つまり、`WITH RECURSIVE` は「アドホッククエリでは便利」でも、「あとから関数や他の `VIEW` で使い回したい共通ロジック」を `VIEW` 化する用途には向いていません。

## WITH RECURSIVE 句をやめるには

`WITH RECURSIVE` 句を使って書いてしまったクエリは、最大の再帰回数を絞ることを条件に、`LEFT JOIN` をその再帰回数だけクエリに記述することで `WITH RECURSIVE` 句を使わないクエリに修正できるはずです。

たとえば、冒頭の上司を全員取得するクエリは、上司をさかのぼる回数を 5 階層に制限することで、以下のように書き直せます。

```sql
WITH employees AS (
  SELECT 1 AS emp_id, 'Alice' AS name, NULL AS manager_id UNION ALL
  SELECT 2, 'Bob',   1 UNION ALL
  SELECT 3, 'Carol', 2 UNION ALL
  SELECT 4, 'Dave',  2 UNION ALL
  SELECT 5, 'Eve',   3
)
SELECT
  e0.emp_id,
  e0.name,
  -- 上位の上司（遠い順→近い順）で配列化。NULL は除外。
  ARRAY(
    SELECT n FROM UNNEST([e5.name, e4.name, e3.name, e2.name, e1.name]) AS n
    WHERE n IS NOT NULL
  ) AS manager_list
FROM employees e0
LEFT JOIN employees e1 ON e0.manager_id = e1.emp_id  -- 1階層上
LEFT JOIN employees e2 ON e1.manager_id = e2.emp_id  -- 2階層上
LEFT JOIN employees e3 ON e2.manager_id = e3.emp_id  -- 3階層上
LEFT JOIN employees e4 ON e3.manager_id = e4.emp_id  -- 4階層上
LEFT JOIN employees e5 ON e4.manager_id = e5.emp_id  -- 5階層上
ORDER BY 1;
```

このクエリで、冒頭の例と同じデータが得られます。

| emp_id | name  | manager_list              |
|--------|-------|---------------------------|
| 1      | Alice | []                        |
| 2      | Bob   | ["Alice"]                 |
| 3      | Carol | ["Bob", "Alice"]          |
| 4      | Dave  | ["Bob", "Alice"]          |
| 5      | Eve   | ["Carol", "Bob", "Alice"] |

これで、このクエリを使ったテーブル関数を作れるようになり[^1]、

[^1]: 説明を簡単にするためここでは `VIEW` を作らずにいきなりテーブル関数を作っていますが、`VIEW` を介しても同じ挙動になります。

```sql
CREATE OR REPLACE TABLE FUNCTION [dataset].find_all_managers(_emp_id INT64)
AS (
WITH employees AS (
  SELECT 1 AS emp_id, 'Alice' AS name, NULL AS manager_id UNION ALL
  SELECT 2, 'Bob',   1 UNION ALL
  SELECT 3, 'Carol', 2 UNION ALL
  SELECT 4, 'Dave',  2 UNION ALL
  SELECT 5, 'Eve',   3
),
tbl AS (
  SELECT
    e0.emp_id,
    e0.name,
    ARRAY(
      SELECT n FROM UNNEST([e5.name, e4.name, e3.name, e2.name, e1.name]) AS n
      WHERE n IS NOT NULL
    ) AS manager_list
  FROM employees e0
  LEFT JOIN employees e1 ON e0.manager_id = e1.emp_id
  LEFT JOIN employees e2 ON e1.manager_id = e2.emp_id
  LEFT JOIN employees e3 ON e2.manager_id = e3.emp_id
  LEFT JOIN employees e4 ON e3.manager_id = e4.emp_id
  LEFT JOIN employees e5 ON e4.manager_id = e5.emp_id
)
SELECT
  *
FROM tbl
WHERE emp_id = _emp_id
);
```

そして呼び出すこともできます。

- `emp_id = 5` (`Eve`) の上司一覧を関数で呼び出した例

![](/images/bigquery-do-not-use-with-recursive-in-view/2.png)

## まとめ

`WITH RECURSIVE` 句は、`VIEW` 定義に使うのには適しません。アドホックなクエリ上で使うのに留めたほうが安全です。

すでに `WITH RECURSIVE` 句を使ってしまったクエリは、再帰回数を有限に制限してよいのであれば、力技で `WITH RECURSIVE` 句を使わないクエリに変換できるはずです。
