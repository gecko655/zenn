---
title: "コードレビューでよくあったコメント9選 〜Terraform編〜"
emoji: "🌍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "コードレビュー"]
published: true
publication_name: mixi
---
:::message
この記事は [MIXI DEVELOPERS Advent Calendar 2024](https://qiita.com/advent-calendar/2024/mixi)シリーズ2の4日目の記事です。
:::

## この記事について

kotapjp さんの記事「Go初学者へのコードレビューでよくあったコメント20選」や
riddle_tec さんの記事「コードレビューでよくお願いする、コメントの追加のパターン7選」が好評だったようなので、
便乗して「Terraform初学者へのコードレビューでよくあったコメント9選」を書いてみました。

https://zenn.dev/mixi/articles/f07be7f476e2f3
https://zenn.dev/mixi/articles/9ca5a84b73144c

思いつく限り雑多にレビュー観点を並べてみましたが、「こういうレビューもよくあるよね」とか「以前 @gecko655 にこういうレビュー指摘食らったことあるのにこの記事に書かれてないんですけど 😡 」とかあればコメントいただけると大変助かります。

## json の入力が必要な箇所では jsonencode() を使う
```tf
resource "aws_iam_role" "the_lambda_role" {
  name = ""
  assume_role_policy = <<-EOT
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOT
  managed_policy_arns = [
    aws_iam_policy.lambda_execution_role.arn
  ]
}
```
ではなくて
```tf
resource "aws_iam_role" "the_lambda_role" {
  name = ""
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
  managed_policy_arns = [
    aws_iam_policy.lambda_execution_role.arn
  ]
}
```
のように `jsonencode()` を使うよう修正するというものです。
IDE 上でのシンタックスハイライトや静的解析によるより正確な[^1]補完が効くようになるはずです。
また、json 文字列を terraform resource の入力に直接使った場合、リソースによっては内部的に json を整形する処理が走り、整形前と整形後の json の形が変化していると判定されてリソースの差分が出続けてしまうことがあります。 `jsonencode()` を使うことでこれを防ぐことができます。

[^1]: 現代においては GitHub Copilot 等の AI によって文字列 json による設定でもそこそこ補完が出てしまうので、言葉を濁しています。

https://developer.hashicorp.com/terraform/language/functions/jsonencode

## apply 後に決定されるリソースのフィールド値を、他のリソースの for_each で使用しない

> The keys of the map (or all the values in the case of a set of strings) must be known values, or you will get an error message that for_each has dependencies that cannot be determined before apply, and a -target may be needed.

https://developer.hashicorp.com/terraform/language/meta-arguments/for_each#limitations-on-values-used-in-for_each

```tf
locals {
  iam_policy_arns = concat(
    "arn:aws:iam::aws:policy/SomeOtherPolicy",
    aws_iam_policy.sample.arn, # arn は apply 後に決まる
  )
}

resource "aws_iam_policy" "sample" {
  # 略
}

resource "aws_iam_role_policy_attachment" "ec2" {
  for_each = toset(local.iam_policy_arns) # エラーになる
  policy_arn = each.key

  # 略
}
```

このように、apply 後に決まる値 `aws_iam_policy.sample.arn` を他のリソースで利用し、全リソースを一気に apply しようとすると以下のようなエラーが出ます。

> The "for_each" value depends on resource attributes that cannot be determined until apply, so Terraform cannot predict how many instances will be created. To work around this, use the -target argument to first apply only the resources that the for_each depends on.

これが、 `aws_iam_policy.sample` を apply →  `aws_iam_role_policy_attachment.ec2` を apply のように継ぎ足しでリソースを足していくとエラーが出ないのがとても厄介で、コードレビュー直後は気づかずにマージ＆ apply されてしまうが、後でリソース全体を別環境で一度に再生成するときにエラーが出て困ることになります。
後で気づくと困るのでコードレビューで落としましょう。

count + length に書き換えることで解決します。

```tf
locals {
  iam_policy_arns = concat(
    "arn:aws:iam::aws:policy/SomeOtherPolicy",
    aws_iam_policy.sample.arn, # arn は apply 後に決まる
  )
}

resource "aws_iam_policy" "sample" {
  # 略
}

resource "aws_iam_role_policy_attachment" "ec2" {
  count = length(local.iam_policy_arns)
  policy_arn = local.iam_policy_arns[count.index]
  # 略
}
```

同様の議論:
- https://qiita.com/kinchiki/items/2c294e225d673e65111c
- https://discuss.hashicorp.com/t/the-for-each-value-depends-on-resource-attributes-that-cannot-be-determined-until-apply/25016/4

なお、この問題を避けることがとても難しいリソースが世の中に存在します。
そのようなリソースでは、1度に全リソースを `terraform apply` することを諦めて2回に分けて apply するしかないかもしれません。

:::details 参考: AWS ACM Certificate の domain_validation_options の例
https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/acm_certificate#referencing-domain_validation_options-with-for_each-based-resources では以下のような terraform 定義が例示されています。
- 証明書を発行するための DNS 検証レコードを作成する terraform 定義 https://docs.aws.amazon.com/ja_jp/acm/latest/userguide/dns-validation.html

```tf
resource "aws_acm_certificate" "example" {
  # 略
}
resource "aws_route53_record" "example" {
  for_each = {
    for dvo in aws_acm_certificate.example.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  name            = each.value.name
  records         = [each.value.record]
  type            = each.value.type
  # 略
}
```
この定義では、 `aws_acm_certificate.example.domain_validation_options` の値を `for_each` で使っているため、 **apply 時にエラーが発生します。** （公式ドキュメントの例そのままなのに！）

> The "for_each" value depends on resource attributes that cannot be determined until apply, so Terraform cannot predict how many instances will be created.

この問題は2020年ごろから指摘されているものの直る見込みはないようです。

https://github.com/hashicorp/terraform-provider-aws/issues/14447

GitHub Issue 内では、 `aws_acm_certificate.example.domain_validation_options` の配列が _（ドキュメントにそんなことは書かれていないが経験的に）_ 毎回必ず1個だけ返ってくることを前提に、以下のように書き換える workaround が提案されていますが…… これはちょっと気持ち悪いですね…。

```tf
resource "aws_route53_record" "example" {
  name            = aws_acm_certificate.example.domain_validation_options.*.resource_record_name[0]
  records         = [aws_acm_certificate.example.domain_validation_options.*.resource_record_value[0]]
  type            = aws_acm_certificate.example.domain_validation_options.*.resource_record_type[0]
  zone_id         = aws_route53_zone.example.zone_id
  ttl             = 60
}
```
:::

## 可能なら count ではなく for_each を使う ＆ for_each のキー名はリソースを一意に表すものにする。
[#apply 後に決定されるリソースのフィールド値を、他のリソースの for_each で使用しない](#apply-後に決定されるリソースのフィールド値を、他のリソースの-for_each-で使用しない)と競合するのですが、 for_each のキー名はなるべくそのリソースの一意な名前を表す文字列にすべきです。

例えば、2つの ML にそれぞれ同一 role を3つ付与したい場合を考えます。

```tf
locals {
  analysis_members = [
    "group:some-group1@mixi.co.jp",
    "group:some-group2@mixi.co.jp"
  ]
  analysis_roles = [
    "roles/bigquery.dataViewer",
    "roles/bigquery.jobUser",
    "roles/logging.viewer",
  ]
}
```

これを、 setproduct と count を使うとこんな風に書けそうです。

```tf
locals {
  ml_role_pairs = setproduct(local.analysis_members, local.analysis_roles)
}
resource "google_project_iam_member" "example" {
  count(length(ml_role_pairs))
  project = var.project_id
  role    = local.ml_role_pairs[0][count.index]
  member  = local.ml_role_pairs[1][count.index]
}
```
これを apply すると、作成されるリソースの参照名([reference](https://developer.hashicorp.com/terraform/language/expressions/references#references-to-resource-attributes))は `google_project_iam_member.example[0]` 〜 `google_project_iam_member.example[5]` のようにキーが数値（インデックス）になります。

キーがインデックスになっていると、リソースの定義順が変わったり、リソースの間に新たなリソースが追加されたり削除されたりすると、インデックスがずれるためリソースが削除＆再生成になってしまいよろしくありません。
また、リソースの参照名 `google_project_iam_member.example[0]` から、どれがどのリソースなのかが判別できないのも良くないです。

以下のように、 for_each でキーを `"${ML 名}-${role 名}"` となるように書き換えることで改善できます。
```tf
resource "google_project_iam_member" "example" {
  for_each = {
    for pair in setproduct(local.analysis_members, local.analysis_roles) : "${pair[0]}-${pair[1]}" => {
      member = pair[0],
      role   = pair[1],
    }
  }
  project = var.project_id
  role    = each.value.role
  member  = each.value.member
}
```
こうすると、リソースの参照名は `google_project_iam_member.example[group:some-group1@mixi.co.jp-roles/bigquery.dataViewer]` となり、リソースの定義順序が変わったとしても `group:some-group1@mixi.co.jp` に与えた `roles/bigquery.dataViewer` の role は削除＆再生成されません。
また、リソースの参照名が中身を表していてわかりやすくなっています。

どうしても count + index を使わざるを得ないことも往々にしてあるのですが、そうでなければ for_each で key に意味のある名前を付けられるとよいです。

同様の議論
- https://qiita.com/ldr/items/04a5d046dc1ab6222dcf
- https://tellme.tokyo/post/2022/06/12/terraform-count-for-each/

## module 内のリソース定義では、一意性が求められるようなすべての名前に variable を含める

例えば、 module 内で以下のようなリソース定義があった場合、 S3 の bucket 名は AWS 全体で一意でなければいけないので、この module は1度しか使うことができません。

```tf
resource "aws_s3_bucket" "example" {
  bucket = "some-bucket"
}
```

リソースの名前規則はそれぞれの terraform の利用ポリシーに依存することだと思うのでここではサラッと流しますが、少なくとも「 module ごとに異なる値が設定されることが想定される variable」が、「一意性が必要となる名前」に設定されていることをコードレビューでチェックしましょう。

例えば、 `var.env` が module 定義ごとにそれぞれ別の値が指定されるような variable なのであれば、↓のようにすれば S3 bucket 名が衝突せずに済みます。
```tf
variable "env" {
  description = "環境名"
  type        = string
}
resource "aws_s3_bucket" "example" {
  bucket = "${var.env}-some-bucket"
}
```

どのリソースのどの名前が一意性を必要とするかは…… そのクラウドサービスに詳しい人の知識に頼るか、いちいち調べる必要があります。頑張りましょう。


ただし、「そのクラウドアカウント内で一意であれば良い名前で、かつそのモジュールは クラウドアカウントあたり1つしか定義しない」というような場合はこの限りではないこともあります。

```tf
resource "google_service_account" "example" {
  account_id   = "example-account" 
}
```
`google_service_account` の `account_id` は GCP プロジェクトごとに一意である必要がありますが、このモジュール自体を GCP プロジェクト1つあたり1回しか定義しないような想定ならば、このままにしてもよいかと思います。


## terraform-docs に準拠したコメントを書く
https://github.com/terraform-docs/terraform-docs

terraform にも他の言語と同様に公式のコメント規約があります。

terraform 定義中に書いたドキュメントは、 [terraform-docs が提供する GitHub Actions](https://github.com/marketplace/actions/terraform-docs-gh-actions) で自動で README.md 生成＆コミットするようにしてあるととても便利です。

## terraform fmt を実行してからレビューを依頼する
terraform には、コードのフォーマットを行なう公式のフォーマッタが存在し、 `terraform fmt` コマンドで起動できます。
`terraform fmt` を実行済みのコードをプルリクエストするようにルール化し、フォーマットされてなさそうなコードがレビュー依頼されたらまずは `terraform fmt` してからレビュー依頼するよう返してしまいましょう。

ただ、 `terraform fmt` のチェックは機械でもできることなので、 `terraform fmt` のレビュー指摘は可能であれば CI 等にまかせてしまいたいところですね[^3]。

[^3]: 例えば https://github.com/marketplace/actions/terraform-fmt を使って自動で `terraform fmt` ＆コミットができる

terraform のコードフォーマットについて: 
https://developer.hashicorp.com/terraform/language/style#code-formatting

## terraform の実行マシン上での外部コマンド実行を必要とする定義は書かない & そのような module は使わない
HCP Terraform (Terraform Cloud) の環境下では、 standard shell utilities 以外は使えることが保証されていません。
- https://discuss.hashicorp.com/t/what-distribution-does-terraform-cloud-use/5491/2
    - > Terraform Enterprise does not guarantee availability of any particular language runtimes or external programs beyond standard shell utilities
- https://developer.hashicorp.com/terraform/cloud-docs/run/install-software#avoid-installing-extra-software
    - > Whenever possible, don't install software on the worker.

HCP Terraform を使っているならば、外部コマンドに依存した terraform 定義はそもそも使えないはずです。また、独自の terraform 環境を持っている場合も将来 HCP Terraform に移行するのが辛くなるので新規導入はできればしないほうが良いです。

- なお、株式会社 MIXI では HCP Terraform の使用が推奨されています。

例えば、自前で構築した terraform 実行環境で `jq` がインストールされていることを前提として、↓のような provisioner を書くのは推奨されません。[^2]
[^2]: もっとも、 json 解析したいのであれば `jq` ではなく [jsondecode()](https://developer.hashicorp.com/terraform/language/functions/jsondecode)が使えるのでこのようなコードが書かれることはあまりないと思いますが…

```tf
resource "null_resource" "example" {
  provisioner "local-exec" {
    command = "cat hoge.txt | jq '.piyo'"
  }
}
```

また、 [modules/terraform-aws-modules/lambda/aws](https://registry.terraform.io/modules/terraform-aws-modules/lambda/aws/latest) という AWS 公式の lambda 関数デプロイ用の module があるのですが、この module は中身で[外部コマンド `python3` `python.exe` を使っています](https://github.com/terraform-aws-modules/terraform-aws-lambda/blob/1fe3e4ac2552ac4fd20126aac874186f27de8edb/package.tf#L2)。
このような外部コマンドを含んだ terraform 定義を書いてしまうと、 HCP Terraform では動作しません。


## output に、 terraform 適用後に terraform 外に設定する必要のある値を出力する

terraform リソースの変更があったときに、外部の設定も同時に変更しなければいけないことに気付けるようになるので、変化が terraform 外に波及するような値は output に設定しておくと便利です。

[AWS load balancer controller に cognito の設定をするときの例](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/#auth-idp-cognito)
```tf
# k8s manifest の ingress 上で設定する cognito の情報
output "cognito_user_pool_domain" {
  value = aws_cognito_user_pool_domain.example.domain
}
output "cognito_user_pool_id" {
  value = aws_cognito_user_pool.example.id
}
output "cognito_user_pool_client_id" {
  value = aws_cognito_user_pool_client.example.id
}
```

↑で作ったリソースの ID 等を kubernetes manifest に設定する例：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: some-ingress
  annotations:
    alb.ingress.kubernetes.io/auth-idp-cognito: |
      {
        "UserPoolArn": "[ここに cognito_user_pull_id を書く]",
        "UserPoolClientId": "[ここに cognito_user_pull_client_id を書く]",
        "UserPoolDomain": "[ここに cognito_user_pool_domain を書く]"
      }
```

- output に変化があったときの `terraform plan & apply` の出力例：
```
Changes to Outputs:
  ~ cognito_user_pool_id                    = "xxxxxxxx" -> "yyyyyyyy"
```
このような output が出たときに、対応する terraform 外の設定値を同時に変えなければいけないことに気付けるようになるはずです。


## (GoogleCloud) `google_project_iam_member` 等の最後に `_member` がつく IAM policy を使う

この記事では Provider に依らない一般的な terraform 定義の書き方に関するレビュー観点を列挙していくつもりだったのですが、 Provider 依存なものの人間がレビューしないと事故を起こしかねないのでこれだけは紹介させてください。

https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_iam

この件はあまりにも有名で、ぼくが理由をここに書くよりも https://blog.g-gen.co.jp/entry/how-to-use-iam-resources-of-terraform などすでにいい記事がウェブ上に多数存在するので、そちらをご覧ください。

## まとめ
主観的なものも多分に含まれていたと思いますが、私がよく指摘するコメントをまとめた記事でした。
この記事に書かれていたもので共感するものがあれば、コードレビューや自身のコードの書き方に活かしていただければ幸いです。
