---
title: "Terraformで、誤って子モジュールにprovider設定を書いた後にルートモジュールに移動させても、stateが変化しないことがある"
emoji: "🚸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: 
  - "Terraform"
published: true
publication_name: "mixi"
---

発生条件が特殊で、 Web をいくら検索しても同じ問題が発生している人を見かけなかったので[^1]、記事にしてみます。

[^1]: 検索言語は日本語と英語だけですが…

## 前提知識
### provider alias について
AWS や GCP の terraform provider は、 リージョンごとに1つの provider を定義する仕様になっており、2つ以上のリージョンのリソースを作りたいときは provider を2つ以上使う必要があります。

例えば、以下のように provider に alias を付けておけば  同一の provider を複数個使うことができ、複数 region の AWS resource を作れます。

```tf
provider "aws" {
  region = "ap-northeast-1" # Tokyo
}

provider "aws" {
  alias  = "us-east-1" # N. Virginia
  region = "us-east-1"
}

# 利用するときは、 resource 定義時に provider alias を指定する

resource "aws_instance" "tokyo_instance" {
  # 略
}

resource "aws_wafv2_web_acl" "nvirginia_instance" {
  provider = aws.us-east-1
  
  # 略
}
```
- 参考: https://developer.hashicorp.com/terraform/language/providers/configuration#alias-multiple-provider-configurations の説明が詳しい

### 子モジュールでの provider alias の使い方

Terraform は一般的にルートモジュールと、ルートモジュールから使われる子モジュールから構成されることが多いと思います。
```
project/
├── main.tf
└── modules/
    └── example_module/
        └── main.tf
```

このとき、子モジュールで provider alias を指定するのは推奨されません。



```tf
# project/main.tf
module "example" {
  source = "./modules/example_module"
}
```
```tf
# project/modules/example_module/main.tf

# ↓良くない例
provider "aws" {
  region = "ap-northeast-1" # Tokyo
}

# 良くない例
provider "aws" {
  alias  = "us-east-1" # N. Virginia
  region = "us-east-1"
}

resource "aws_instance" "tokyo_instance" {
  # 略
}

resource "aws_wafv2_web_acl" "nvirginia_instance" {
  provider = aws.us-east-1
  
  # 略
}
```

このような terraform 定義を書いてしまったときに一番困るのは、 子モジュールを削除するときです。

- 子モジュール削除の差分の例：
```diff
- module "example" {
-   source = "./modules/example_module"
- }
```

これを `terraform apply` すると、 「子モジュール内にある resource を消したいが、 resource を消すために必要な provider が同時に消されているため、どのように resource を消したらいいかがわからない」という旨のエラーが発生します。

> Error: Provider configuration not present
> To work with module.[モジュール名].aws_wafv2_web_acl.[リソースID] (orphan) its original provider configuration at module.[モジュール名].provider["registry.terraform.io/hashicorp/aws"].us-east-1 is required, but it has been removed. This occurs when a provider configuration is removed while objects created by that provider still exist in the state. Re-add the provider configuration to destroy module.[モジュール名].aws_wafv2_web_acl.[リソース名] (orphan), after which you can remove the provider configuration again.

![](/images/terraform-move-provider-from-child-module/1.png)

そのため、子モジュールには直接 provider を指定せず、ルートモジュールから子モジュールに provider を渡すことが推奨されています。

- ドキュメントでは推奨・非推奨というよりも、禁止(must not)で書かれている。が、実際には誤って設定してもエラーや警告は一切出ないので、知らないと気付かない……
> A module intended to be called by one or more other modules must not contain any `provider` blocks.
> 1つ以上の他のモジュールから呼び出されるモジュールには、`provider` ブロックを含めることはできません。
> https://developer.hashicorp.com/terraform/language/modules/develop/providers

```tf
# project/main.tf
module "example" {
  source = "./modules/example_module"
  providers = {
    aws           = aws 
    # ↑ alias を使っていない provider は明示的に渡さなくても勝手に子モジュールに渡されるので、この1行は書いても書かなくても良い
    aws.us-east-1 = aws.us-east-1
  }
}
```
```tf
# project/modules/example_module/main.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 5.0"
      configuration_aliases = [ aws, aws.us-east-1]
      # ↑ このモジュールでは外から aws.us-east-1 が渡されないといけないことを宣言する
      # https://developer.hashicorp.com/terraform/language/providers/configuration#alias-multiple-provider-configurations
    }
  }
}

resource "aws_instance" "tokyo_instance" {
  # 略
}

resource "aws_wafv2_web_acl" "nvirginia_instance" {
  provider = aws.us-east-1
  
  # 略
}
```

## 発生した問題

上で説明した 「provider alias は子モジュールで宣言するのではなくルートモジュールから与えなければならない」ということを知らずに、子モジュールに provider 設定をそのまま書いてしまっている状態の terraform プロジェクトがありました。
その状態で子モジュールを削除しようとしたところ、前述の「子モジュール内にある resource をどのように消したらいいかがわからない」エラーが発生しました。

![](/images/terraform-move-provider-from-child-module/1.png)


そのため、ルートモジュールが provider alias を与えるよう terraform 定義を修正しました。

（ルートモジュールの差分）
```diff
module "example" {
  source = "./modules/example_module"
+   providers = {
+     aws           = aws 
+     aws.us-east-1 = aws.us-east-1
+   }
}
```


（子モジュールの差分）
```diff
- provider "aws" {
-   region = "ap-northeast-1" # Tokyo
- }
- provider "aws" {
-   alias  = "us-east-1" # N. Virginia
-   region = "us-east-1"
- }
+ terraform {
+   required_providers {
+     aws = {
+       source                = "hashicorp/aws"
+       version               = "~> 5.0"
+       configuration_aliases = [ aws, aws.us-east-1]
+     }
+   }
+ }

resource "aws_instance" "tokyo_instance" {
  # 略
}

resource "aws_wafv2_web_acl" "nvirginia_instance" {
  provider = aws.us-east-1
  
  # 略
}
```

これを terraform apply して、再度子モジュールを削除しようとしました。

（ルートモジュールの差分）
```diff
- module "example" {
-   source = "./modules/example_module"
-   providers = {
-     aws           = aws 
-     aws.us-east-1 = aws.us-east-1
-   }
- }
```

しかし、 **provider alias をルートから与えるように terraform 定義を変えたはずなのに、同じエラーが発生して子モジュールが削除できませんでした**。
![](/images/terraform-move-provider-from-child-module/1.png)

## 原因
terraform state の中身を見て原因を調査したところ、どうやら、 **provider の与え方を変えたがそれ以外にリソースに差分がないとき、 terraform apply ではリソースに紐づけられた provider の情報が更新されないことがわかりました**。

terraform stateを確認すると、 `terraform apply` 後も子モジュールに定義した provider である `module.[モジュール名].provider["registry.terraform.io/hashicorp/aws"].us-east-1` が `aws_wafv2_web_acl`の provider としてセットされていることがわかります。

![](/images/terraform-move-provider-from-child-module/4.png)
（HCP Terraform (旧Terraform Cloud)で見られる terraform state のスクリーンショット）

provider 情報に差分があっても、リソース自身に差分がないときは、 terraform apply しても provider 情報が更新されないようです。

## 解決

[`terraform apply -refresh-only`](https://developer.hashicorp.com/terraform/cli/commands/refresh) を実行すると、リソースに差分がなくても terraform state が更新され、 provider が子モジュールのものからルートモジュールのものに更新されました。

↓ provider が
`module.[モジュール名].provider["registry.terraform.io/hashicorp/aws"].us-east-1` から
`provider["registry.terraform.io/hashicorp/aws"].us-east-1` に変化した様子。
![](/images/terraform-move-provider-from-child-module/5.png)

- 実際には `terraform apply -refresh-only` を直接実行しておらず、 HCP Terraform の "Refresh state" を実行して解決しました。
    - ドキュメントによれば、 HCP Terraform の "Refresh state" は、 CLI の `terraform apply -refresh-only` と等価です。 https://developer.hashicorp.com/terraform/cloud-docs/run/modes-and-options#refresh-only-mode

![](/images/terraform-move-provider-from-child-module/6.png)

`terraform apply` では provider の差分が反映されないのに、 `terraform apply -refresh-only` なら反映されるの、その仕様でいいのか…？という感じがしますね……

## まとめ

terraform provider を誤って子モジュールに書いたとき、ルートモジュールに provider を移動させても、リソースに差分がなければ変更が反映されないようです。
この問題は `terraform apply -refresh-only` で解決します。
ちょっと微妙な仕様だと思うので、 terraform apply 時にリソースの差分がなくても勝手に refresh してほしいなと思いました。

## 備考

- Web 検索してもこの情報はでてこないので、 ChatGPT 等の AI は役に立たなかった[^3]

[^3]: プロンプトが悪い説はある

## 検証したバージョン

- terraform 1.10.2
- AWS provider 5.93.0
