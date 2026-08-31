---
title: "Amazon Aurora のインスタンスの promotion_tier を設定しても、クラスタ新規作成時は無視される"
emoji: "🌩️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Aurora", "Terraform"]
published: true
publication_name: "mixi"
---

## Amazon Aurora の promotion_tier について

`promotion_tier` は Amazon Aurora のインスタンスに設定できるパラメータで、以下の意味を持ちます。

- フェイルオーバー時にどの reader を writer に昇格させるかを決定する優先度
- 値が小さいほど優先される（0 が最優先）

https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html

つまり：

> **promotion_tier は「フェイルオーバー時の昇格順位」を制御するためのもの**

です。

## Terraform での設定例

以下のように、Terraform で複数のインスタンスに `promotion_tier` を設定できます。

```hcl
resource "aws_rds_cluster" "example" {
  cluster_identifier = "example-cluster"
  engine             = "aurora-mysql"
  master_username    = "admin"
  master_password    = "password"
}

resource "aws_rds_cluster_instance" "instance_a" {
  identifier         = "instance-a"
  cluster_identifier = aws_rds_cluster.example.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.example.engine

  promotion_tier = 0  # 最優先
}

resource "aws_rds_cluster_instance" "instance_b" {
  identifier         = "instance-b"
  cluster_identifier = aws_rds_cluster.example.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.example.engine

  promotion_tier = 15  # 低優先
}
```

Terraform のリソース仕様はこちら：  
https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/rds_cluster_instance#promotion_tier-1

この設定を見ると、「instance_a が writer になるはず」と考えがちです。

## 実際の挙動

しかし、新規作成時の Aurora クラスタでは次のような挙動になります：

- writer は **最初に作成されたインスタンス** になる
- `promotion_tier` は **この選定には使われない**
- Terraform では複数インスタンスが並列に作成されるため、
  - **どのインスタンスが最初に作られるかは保証されない**

その結果、**promotion_tier を指定したにもかかわらず、promotion_tier が小さい（優先度が高い）インスタンスが writer にならない**ことがあります。

## なぜこのような挙動になるのか

AWS の公式ドキュメントでは、以下のように説明されています：

- クラスタには1つの primary（writer）が存在する  
  https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.html

- writer は「クラスタで最初に作成されたインスタンス」である  
  https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.CreateInstance.html#aurora-create-writer

一方で：

- `promotion_tier` はフェイルオーバー時の昇格優先度としてのみ説明されている  
  https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html

つまり：

- promotion_tier は「初期選定」ではなく「障害時の挙動」を制御するパラメータ
- 作成時の writer 選定ロジックは別（かつ詳細は公開されていない）

## Terraform での落とし穴

Terraform ではさらに以下が問題になります：

- 複数の `aws_rds_cluster_instance` を定義すると並列に作成される
- 作成順は保証されない
- writer を明示的に指定する方法がない

そのため、「promotion_tier を設定したからこのインスタンスが writer になるはず」という期待は、Terraform では保証されません。

## 対策

### 方法①：1台ずつ作る

Terraform ではリソースは基本的に並列で作成されるため、そのままだとどのインスタンスが先に作成されるかは保証されません。  
そのため、**`depends_on` を使って明示的に作成順を制御する**ことで、新規作成時に writer となるインスタンスを明示的に指定できます。
- 同様の議論： https://stackoverflow.com/questions/78568753

```hcl
# まず writer 用インスタンスだけ作る
resource "aws_rds_cluster_instance" "writer" {
  identifier         = "writer"
  cluster_identifier = aws_rds_cluster.example.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.example.engine
}

# その後 reader を追加
resource "aws_rds_cluster_instance" "reader" {
  identifier         = "reader"
  cluster_identifier = aws_rds_cluster.example.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.example.engine

  depends_on = [aws_rds_cluster_instance.writer]
}
```
- ただし、↑のようにインスタンスに writer/reader のような名前をつけると、インスタンス作成後 DB がフェイルオーバーして writer/reader がひっくり返ったときに名前と実態が一致しなくなります。 instance_a, instance_b とか、 primary, additional みたいな命名のほうが良いかもしれません。

### 方法②：作成後にフェイルオーバーする
新規作成時は promotion_tier が使われませんが、フェイルオーバー時には使われますので、
新規作成後に実際に1回フェイルオーバーすれば、 promotion_tier 通りに writer インスタンスが選ばれるはずです。

```bash
aws rds failover-db-cluster \
  --db-cluster-identifier example-cluster \
  --target-db-instance-identifier instance-a
```

CLI ドキュメント：  
https://docs.aws.amazon.com/cli/latest/reference/rds/failover-db-cluster.html

## まとめ

- `promotion_tier` は **フェイルオーバー時の昇格優先度**
- クラスタ新規作成時の writer 選定には使われない
- Terraform ではインスタンス作成順が保証されない
- **Terraform では promotion_tier を設定したつもりでも、実際には promotion_tier の小さい（優先度の高い）が writer に選ばれない事があるので注意が必要**
