---
title: "HCP Terraformで、子モジュールにprovider設定を書いた後にルートモジュールに移動させても、stateが変化しないことがある"
emoji: "🚸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: 
  - "Terraform"
  - "HCPTerraform"
published: true
publication_name: "mixi"
---

発生条件が特殊で、 Web をいくら検索しても同じ問題が発生している人を見かけなかったので[^1]、記事にしてみます。

[^1]: 検索言語は日本語と英語だけですが…

## 3行で
- [HCP Terraform](https://www.hashicorp.com/ja/products/terraform) で、誤って子モジュールに provider alias の宣言をしていたのをルートモジュールに移動させたところ、 terraform state に反映されなかった
- HCP Terraform の "Refresh state" を実行したところ、 terraform state が更新され、ルートモジュールの provider が使われるようになった
- local backend では再現しなかったので、 HCP Terraform のみの問題らしい

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
    └── [module_name]/
        └── main.tf
```

このとき、以下のように子モジュールで provider alias を直接指定するのは推奨されません。

```tf
# project/main.tf
module "provider_test" {
  source = "./modules/provider_test"
}
```
```tf
# project/modules/provider_test/main.tf

# ↓良くない例
provider "null" {
  alias = "test1"
}

provider "null" {
  alias = "test2"
}

resource "null_resource" "test1" {
  provider = null.test1
}

resource "null_resource" "test2" {
  provider = null.test2
}
```

このような terraform 定義を書いてしまったときに一番困るのは、 子モジュールを削除するときです。

- 子モジュール削除の差分の例：
```diff
# project/main.tf
- module "provider_test" {
-   source = "./modules/provider_test"
- }
```

これを `terraform apply` すると、 「子モジュール内にある resource を消したいが、 resource を消すために必要な provider が同時に消されているため、どのように resource を消したらいいかがわからない」という旨のエラーが発生します。

> Error: Provider configuration not present
> To work with module.provider_test.null_resource.test1 (orphan) its original provider configuration at module.provider_test.provider["registry.terraform.io/hashicorp/null"].test1 is required, but it has been removed. This occurs when a provider configuration is removed while objects created by that provider still exist in the state. Re-add the provider configuration to destroy module.provider_test.null_resource.test1 (orphan), after which you can remove the provider configuration again.

![](/images/terraform-move-provider-from-child-module/1.png)

そのため、子モジュールには直接 provider を指定せず、ルートモジュールから子モジュールに provider を渡すことが推奨されています。

- ドキュメントでは推奨・非推奨というよりも、禁止(must not)で書かれている。が、実際には誤って設定してもエラーや警告は一切出ないので、知らないと気付かない……
> A module intended to be called by one or more other modules must not contain any `provider` blocks.
> 1つ以上の他のモジュールから呼び出されるモジュールには、`provider` ブロックを含めることはできません。
> https://developer.hashicorp.com/terraform/language/modules/develop/providers

```tf
# project/main.tf
provider "null" {
  alias = "test1"
}
provider "null" {
  alias = "test2"
}
module "provider_test" {
  source = "./modules/provider_test"
  providers = {
    null.test1 = null.test1
    null.test2 = null.test2
  }
}
```
```tf
# project/modules/provider_test/main.tf
terraform {
  required_providers {
    null = {
      configuration_aliases = [ null.test1, null.test2 ]
      # ↑ このモジュールでは外から null.test1, null.test2 の provider が渡されないといけないことを宣言する
      # https://developer.hashicorp.com/terraform/language/providers/configuration#alias-multiple-provider-configurations
    }
  }
}

resource "null_resource" "test1" {
  provider = null.test1
}

resource "null_resource" "test2" {
  provider = null.test2
}
```

## 発生した問題
ある[HCP Terraform](https://www.hashicorp.com/ja/products/terraform) を使っている terraform workspace では、上で説明した 「provider alias は子モジュールで宣言するのではなくルートモジュールから与えなければならない」ということを知らずに、子モジュールに provider 設定をそのまま書いてしまっていました。
その状態で子モジュールを削除しようとしたところ、前述の「子モジュール内にある resource をどのように消したらいいかがわからない」エラーが発生しました。

![](/images/terraform-move-provider-from-child-module/1.png)


そのため、ルートモジュールが provider alias を与えるよう terraform 定義を修正しました。

（ルートモジュールの差分）
```diff
# project/main.tf
module "provider_test" {
  source = "./modules/provider_test"
+   providers = {
+     null.test1 = null.test1
+     null.test2 = null.test2
+   }
}
```


（子モジュールの差分）
```diff
# project/modules/provider_test/main.tf
- provider "null" {
-   alias = "test1"
- }
- 
- provider "null" {
-   alias = "test2"
- }
+ terraform {
+   required_providers {
+     null = {
+       configuration_aliases = [ null.test1, null.test2 ]
+     }
+   }
+ }

resource "null_resource" "test1" {
  provider = null.test1
}
resource "null_resource" "test2" {
  provider = null.test2
}
```

これを terraform apply した後、再度子モジュールを削除しようとしました。

（ルートモジュールの差分）
```diff
# project/main.tf
- module "provider_test" {
-   source = "./modules/provider_test"
-   providers = {
-     null.test1 = null.test1
-     null.test2 = null.test2
-   }
- }
```

しかし、 **provider alias をルートから与えるように terraform 定義を変えたはずなのに、同じエラーが発生して子モジュールが削除できませんでした**。
![](/images/terraform-move-provider-from-child-module/1.png)

## 原因
terraform state の中身を見て原因を調査したところ、どうやら、 **provider の与え方を変えたがそれ以外にリソースに差分がないとき、 terraform apply ではリソースに紐づけられた provider の情報が更新されないことがわかりました**。

terraform stateを確認すると、 `terraform apply` 後も子モジュールに定義した provider である `module.provider_test.provider["registry.terraform.io/hashicorp/null"].test1` が `null_resource.test1`の provider としてセットされていることがわかります。

![](/images/terraform-move-provider-from-child-module/4.png)
（HCP Terraform (旧Terraform Cloud)で見られる terraform state のスクリーンショット）

provider 情報に差分があっても、リソース自身に差分がないときは、 terraform apply しても provider 情報が更新されないようです。

## 解決

[`terraform apply -refresh-only`](https://developer.hashicorp.com/terraform/cli/commands/refresh) を実行すると、リソースに差分がなくても terraform state が更新され、 provider が子モジュールのものからルートモジュールのものに更新されました。

↓ provider が
`module.provider_test.provider["registry.terraform.io/hashicorp/null"].test1` から
`provider["registry.terraform.io/hashicorp/null"].test1` に変化した様子。
![](/images/terraform-move-provider-from-child-module/5.png)

- 実際には `terraform apply -refresh-only` を直接実行しておらず、 HCP Terraform の "Refresh state" を実行して解決しました。
    - ドキュメントによれば、 HCP Terraform の "Refresh state" は、 CLI の `terraform apply -refresh-only` と等価です。 https://developer.hashicorp.com/terraform/cloud-docs/run/modes-and-options#refresh-only-mode

![](/images/terraform-move-provider-from-child-module/6.png)

`terraform apply` では provider の差分が反映されないのに、 `terraform apply -refresh-only` なら反映されるの、その仕様でいいのか…？という感じがしますね……

## HCP Terraform 以外の環境では

原因の切り分けのため、 HCP Terraform 以外の環境でも同じことが起きるか検証してみました。

local backend の設定の例：
```tf
# project/main.tf
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
module "provider_test" {
  source = "./modules/provider_test"
}
```

local backend で、 provider を子モジュールからルートモジュールに移動させた後、 `terraform apply` を実行したところ、問題なく provider が更新されました。

```diff
# terraform.tfstate
diff --git a/terraform.tfstate b/terraform.tfstate
index 15093c5..2df160f 100644
--- a/terraform.tfstate
+++ b/terraform.tfstate
@@ -1,7 +1,7 @@
 {
   "version": 4,
   "terraform_version": "1.12.0",
-  "serial": 1,
+  "serial": 2,
   "lineage": "488986e2-5a88-2a30-2792-5a1f935dacbf",
   "outputs": {},
   "resources": [
@@ -10,7 +10,7 @@
       "mode": "managed",
       "type": "null_resource",
       "name": "test1",
-      "provider": "module.provider_test.provider[\"registry.terraform.io/hashicorp/null\"].test1",
+      "provider": "provider[\"registry.terraform.io/hashicorp/null\"].test1",
       "instances": [
         {
           "schema_version": 0,
@@ -28,7 +28,7 @@
       "mode": "managed",
       "type": "null_resource",
       "name": "test2",
-      "provider": "module.provider_test.provider[\"registry.terraform.io/hashicorp/null\"].test2",
+      "provider": "provider[\"registry.terraform.io/hashicorp/null\"].test2",
       "instances": [
         {
           "schema_version": 0,
```

その後、 子モジュールを削除しても、問題なくリソースが削除されました。

```
Terraform will perform the following actions:

  # module.provider_test.null_resource.test1 will be destroyed
  # (because null_resource.test1 is not in configuration)
  - resource "null_resource" "test1" {
      - id = "1000537405425120607" -> null
    }

  # module.provider_test.null_resource.test2 will be destroyed
  # (because null_resource.test2 is not in configuration)
  - resource "null_resource" "test2" {
      - id = "3560490887494319295" -> null
    }

Plan: 0 to add, 0 to change, 2 to destroy.
```

以上から、どうやら Terraform 全体の問題ではなく、 **HCP Terraform のみで発生する問題のようでした**。


## まとめ

**HCP Terraform において**、 terraform provider を誤って子モジュールに書いたとき、ルートモジュールに provider を移動させても、リソースに差分がなければ変更が反映されないようです。
この問題は "Refresh state" を実行することで解決します。
Terraform backend が HCP Terraform 以外の場合は、この問題は発生しないようです。
ちょっと微妙な仕様だと思うので、 HCP Terraform においてリソースに差分がないときでも、常に terraform state を最新の状態に更新するようにしてほしいなと思いました。

## 備考

- Web 検索してもこの情報はでてこないので、 ChatGPT 等の AI は役に立たなかった[^3]

[^3]: プロンプトが悪い説はある

## 検証したバージョン

- terraform 1.11.3
  - HCP Terraform, local backend ともに同じバージョンで検証しています。
