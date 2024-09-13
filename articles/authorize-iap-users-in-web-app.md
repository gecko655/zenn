---
title: "Identity Aware Proxy で保護された Web アプリで、Scheduler からアクセスする＆アクセスユーザーの認証をする"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CloudScheduler", "IdToken", "IAP", "OIDC", "GoogleCloud"]
published: true
publication_name: "mixi"
---

## 注意

[2024-07-26 に、この記事の内容が覆る追記をしました](#追記-2024-07-26)。

## モチベーション
社内で利用するツールを実装＆Google Cloud にデプロイするため、以下のような Cloud Run + Cloud Load Balancing + Identity Aware Proxy の構成を考えます。

![](/images/authorize-iap-users-in-web-app/1.png)
- https://cloud.google.com/iap/docs/enabling-cloud-run の手順で作れる。

この構成の上で、「ユーザーには使わせたくないが、 Cloud Scheduler からはアクセスできるような API `/automated/job` 」を実装する要件が発生しました。
Identity Aware Proxy(IAP)や Cloud Load Balancing では IAP 認証済みアカウントによってアクセス可否を制御する機能がなさそう([後述](#cloud-load-balancing-で、-iap-認証済みアカウントによるアクセス分岐を実装できないの？))だったので、 Cloud Run のアプリ内でそのようなアクセス可否を実装する必要がありました。

この記事では、準備として「Cloud Scheduler から IAP 保護された Web アプリ(Cloud Run) にアクセスする方法」を説明し、その後本題の「IAP を通った後にバックエンドサービスに届くリクエストから、リクエストしたユーザー（OR サービスアカウント）を特定する方法」について説明します。

## Cloud Scheduler の設定
Cloud Scheduler から IAP で保護された Web アプリにアクセスするには、以下のようにします。
- Cloud Scheduler 用の service account を作成する
    - `roles/iap.httpsResourceAccessor` ロールを付与する
        - 対象のサービスの LB を指定する必要があるので指定する
- scheduler を作成する
    - http target に、 LB の URL を指定する
    - **http リクエストのヘッダーに、上で作成した service account の oidc token を付与するよう指定する**

この oidc token ですが、以下の2つの設定値を指定しておけば、あとは scheduler が勝手に oidc token を発行し、リクエストしてくれるようです。

- service_account_email: サービスアカウントのメールアドレス
- audience: **IAP に設定してある OAuth クライアントの client ID**
    - IAP の OAuth クライアントは、 IAP client https://console.cloud.google.com/security/iap で、トグルスイッチをオンにすると勝手に作成されるので、その client ID を指定してください。

IAP へのリクエスト時に付与すべき audience 設定に関する説明（わかりにくい）: https://cloud.google.com/iap/docs/authentication-howto

terraform で書くとこんな感じ。
```hcl
// LB
resource "google_compute_backend_service" "this" {
  // LB の設定は省略します
}

// scheduler の service account
resource "google_service_account" "scheduler" {
  account_id  = "scheduler"
  description = "Service account for scheduler (managed by terraform)"
}

resource "google_iap_web_backend_service_iam_member" "iap_access_scheduler" {
  web_backend_service = google_compute_backend_service.this.name
  role                = "roles/iap.httpsResourceAccessor"
  member              = "serviceAccount:${google_service_account.scheduler.email}"
}

// oauth-brands(oauth 同意画面) 及び IAP client は、 terraform apply 前に手動で作成する必要がある。
// oauth 同意画面 https://console.cloud.google.com/apis/credentials/consent
// IAP client https://console.cloud.google.com/security/iap で、トグルスイッチをオンにすると iap client が作成される。
data "google_iap_client" "iap_client" {
  // gcloud iap oauth-brands list で取得した値…… なのだが、
  // 実際は project number から生成できる値で固定。
  brand     = "projects/${data.google_project.this.number}/brands/${data.google_project.this.number}"
  client_id = var.iap_client_id
}

// scheduler job
resource "google_cloud_scheduler_job" "this" {
  name        = "week-day-automated-job"
  description = "Invoke a Cloud Run endpoint every week day"
  // 平日朝9時に実行
  schedule  = "0 9 * * MON-FRI"
  time_zone = "Asia/Tokyo"

  retry_config {
    retry_count = 1
  }

  http_target {
    http_method = "POST"
    uri         = "https://cloud-run-lb-host.com/automated/job"

    oidc_token {
      // Identity Aware Proxy の OIDC 認証を通すため、 service_account_email にサービスアカウントを、 audience に IAP の client id を指定する
      // https://cloud.google.com/iap/docs/authentication-howto#obtaining_an_oidc_token_in_all_other_cases
      service_account_email = google_service_account.scheduler.email
      audience              = data.google_iap_client.iap_client.client_id
    }
  }
}
```

##  バックエンドサービスに届くリクエスト
ブラウザや Cloud Scheduler が IAP を通してバックエンドサービスにアクセスしたとき、 リクエストは以下のようにバックエンドサービスまで転送されます。
- （ブラウザアクセスで、リクエストに IAP の認証クッキーが付いていない場合、 IAP はブラウザをGoogle ログインページへリダイレクトさせる）
- ブラウザは認証クッキー[^1]の付いたリクエストを、 Cloud Scheduler は OIDC idtoken ヘッダーの付いたリクエストを IAP に向けて送る
- IAP は、リクエストに付与されている認証クッキー OR idtoken ヘッダーを検証する
- IAP は、認証が問題なければ、 IAP は  Cloud Scheduler のものとは別の OIDC idtoken を発行し `x-goog-iap-jwt-assertion` ヘッダーに付与して、リクエストをバックエンドサービスに転送する
    - このとき、 **IAP の認証に使ったブラウザの認証クッキーや Cloud Scheduler の idtoken は、 IAP によって削除される**
    - https://cloud.google.com/iap/docs/signed-headers-howto
- バックエンドサービスにリクエストが届く

[^1]: `GCP_IAP_AUTH_TOKEN_[hex_string]`, `GCP_IAP_UID` の2つの cookie が、ブラウザにおける IAP の認証クッキーっぽいのですが、細かい仕様は公開されてなさそうな感じがするので詳しくは言及しません

バックエンドサービスでは、この `x-goog-iap-jwt-assertion` ヘッダーに付与された OIDC idtoken を検証することで、このリクエストが IAP を通ったものかどうか ＆ リクエストしたのはどのユーザーなのかを確認することができます。
このとき、idtoken の検証には引数に audience が必要ですが、 **scheduler 作成時に設定した audience と、 IAP を通過しバックエンドサービスに届く idtoken の audience は異なることに注意が必要です**。
IAP が作る token の audience はバックエンドサービスの種類によって異なります。
- Cloud Run + LoadBalancing においては `/projects/PROJECT_NUMBER/global/backendServices/SERVICE_ID` の形式。すなわち、 google backend service の id
    - terraform では `google_compute_backend_service.resource_id.id` で取得できる
    - LB を使っている場合は、 GKE などでも同じ ID が audience に設定されるっぽい
- App Engine においては `/projects/PROJECT_NUMBER/apps/PROJECT_ID`
- Cloud Run では `https://my-cloud-run-service.run.app/` のような値を audience に設定せよとのドキュメントもあるが、これは LB を通さずに Cloud Run にアクセスする時の話で、 IAP + LB + Cloud Run の構成ではこれは使えない。雑に 「Cloud Run oidc audience」でググるとこちらが先にヒットするので紛らわしい。
    - https://cloud.google.com/run/docs/authenticating/service-to-service
- Cloud Console 上で、「 JWT オーディエンスコードの取得」というボタンを押すと、 audience に設定すべき文字列が表示される

![](/images/authorize-iap-users-in-web-app/2.png)

https://cloud.google.com/iap/docs/signed-headers-howto#verifying_the_jwt_payload


実際にGo  ([Echo](https://echo.labstack.com/))で IAP の付与した idtoken ヘッダー `x-goog-iap-jwt-assertion` を検証し、 アクセスしたユーザーを認証する実装はこんな感じになります。
```go
package main

import (
	"github.com/labstack/echo/v4"
	"google.golang.org/api/idtoken"
	"net/http"
	"os"
)

const (
	googleIapAudience   = "/projects/123456789012/global/backendServices/1234567890123456"
	serviceAccountEmail = "scheduler@gcp_project.iam.gserviceaccount.com"
)

func DoSomethingWithRestrictingHumanAccess(c echo.Context) error {
	// https://cloud.google.com/iap/docs/signed-headers-howto
	// 認証ヘッダーの取得
	iapJWT := c.Request().Header.Get("x-goog-iap-jwt-assertion")
	// OIDC token の検証
	idTokenPayload, err := idtoken.Validate(c.Request().Context(), iapJWT, googleIapAudience)
	if err != nil {
		return c.String(http.StatusUnauthorized, "ログインしていません")
	}
	// アクセスしてきたユーザーの email が scheduler の service account のものであることを確認
	if idTokenPayload.Claims["email"].(string) != serviceAccountEmail {
		return c.String(http.StatusForbidden, "この URL は許可されていません")
	}
    
	// 実装

	return c.String(http.StatusOK, "ok")
}
```
認証の細かいことは `google.golang.org/api/idtoken`ライブラリにおまかせできるので便利ですね[^2]。

[^2]: Go 以外の言語でも同様のライブラリがあるっぽい https://cloud.google.com/iap/docs/signed-headers-howto#iap_validate_jwt-go
## まとめ

- Cloud Scheduler から IAP で保護された Web アプリへアクセスする際は、 oidc token をいい感じに設定するとアクセスできる。
- IAP で保護された Web アプリでは、 IAP 認証が通ったユーザーの認証をアプリ内でも行うことができる。
    - 実装自体は、 Google 製ライブラリのお陰で楽にできる。
- IAP に与えるリクエストに付与する idtoken の audience と、バックエンドサービスに届く idtoken の audience は異なることに注意。


## 備考
### Cloud Load Balancing で、 IAP 認証済みアカウントによるアクセス分岐を実装できないの？
今回のような認証の有無によってアクセス可否を制御する実装は、レイヤ的には Cloud Load Balancing や nginx 等のロードバランサ部分でできると嬉しいのですが、「ある URL path には IAP ユーザーのうちの一部だけにアクセスさせる」というような機能は、調べた限りなさそうでした。
- 設定できるとしたら URL Map に設定することになると思うのだが、 URL Map の作成 API にそのような設定項目は（多分）ない
    - https://cloud.google.com/load-balancing/docs/https/traffic-management-global#routing_requests_to_backends
    - https://cloud.google.com/compute/docs/reference/rest/v1/urlMaps/insert


## 追記 2024-07-26

この記事を公開した後に会社の同僚に指摘されたのですが、
`roles/iap.httpsResourceAccessor` role を付けている IAM (人間 OR service account) の role condition で
アクセス可能な path を指定したほうが簡単にアクセス制御が簡単なことがわかりました。

具体的には、 `roles/iap.httpsResourceAccessor` role の条件に `!request.path.startsWith("/automated/")` を指定するだけです。
- https://cloud.google.com/iam/docs/conditions-overview?hl=ja#example-url-host-path


```hcl
resource "google_iap_web_backend_service_iam_member" "iap_access_member" {
  for_each            = toset(var.iap_access_members)
  web_backend_service = google_compute_backend_service.this.name
  role                = "roles/iap.httpsResourceAccessor"
  member              = each.key

  condition {
    expression = "!request.path.startsWith(\"/automated/\")"
    title      = "deny access to /automated/*"
  }
}
```

- `/automated/hoge` にアクセスすると、 IAP からレスポンスが返ってくる（Webアプリに到達していない）
![](/images/authorize-iap-users-in-web-app/3.png)

- `/automated/*` 以外の path は今まで通りアクセスできる（スクショ省略）

アクセス可否の制御以外にユーザー認証を使わない場合、 IAP を通過したリクエストを真面目に認証するよりも role condition でアクセス可否を制御するのが簡単そうです。
