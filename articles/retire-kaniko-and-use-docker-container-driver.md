---
title: "Kaniko が開発停止されてしまったので docker buildx + docker-container driver に乗り換えた話"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kaniko", "CloudBuild", "Docker", "GCP", "GoogleCloud"]
published: true
publication_name: "mixi"
---

## 3行で
- kaniko ( https://github.com/GoogleContainerTools/kaniko ) は docker のレイヤーキャッシュを使ってビルドを高速化するツールだが、2025年6月からメンテモードに入ってしまい、使用が非推奨になった
- docker buildx build によるレイヤーキャッシュ付きビルドで高速化に成功した
- 公式ドキュメントではデフォルトの docker driver でもレイヤーキャッシュが使えると書いてあるが、どうやっても使えず、 docker-container driver を使うようにした

## Kaniko とは

Kaniko ( https://github.com/GoogleContainerTools/kaniko ) とは、 Docker image の CI でのビルドをレイヤーキャッシュによって高速化することができるツールです。
通常、 CI で Docker image をビルドすると毎回キレイまっさらなビルドマシンが用意される[^1]のですが、 kaniko は Docker のレイヤーキャッシュをキャッシュ用 docker registry に保存して勝手に再利用してくれるので、キャッシュが再利用できるときはビルド時間が短縮できます。

[^1]: cf: CircleCI にはビルドマシン自身に Docker レイヤーキャッシュをあらかじめ積んでおいてくれる機能があります。この機能は可能なら非同期に動作するらしいので、この記事で説明する docker buildx build より優秀かもしれません（未検証） https://circleci.com/docs/guides/optimize/docker-layer-caching/

Kaniko は Google Cloud Build のドキュメントからも使用例が示されており、この記事の執筆時点（2025年9月2日）では、以下の「ビルドを高速化する際のベスト プラクティス」のページに説明が残っています。

![](/images/retire-kaniko-and-use-docker-container-driver/1.png)
https://cloud.google.com/build/docs/optimize-builds/speeding-up-builds?hl=ja [^2][^3]


[^2]: 魚拓 https://megalodon.jp/2025-0902-1908-38/https://cloud.google.com:443/build/docs/optimize-builds/speeding-up-builds?hl=ja
[^3]: 記事執筆時点では、この記事にある「Kaniko キャッシュの利用」のリンクは切れています。また、英語版ドキュメントではそもそも「ビルドを高速化する際のベスト プラクティス」のページに Kaniko に関する記述は残っていませんでした。 https://cloud.google.com/build/docs/optimize-builds/speeding-up-builds?hl=en

Cloud Build のドキュメントがあったからという理由で、Cloud Build での CI に Kaniko を導入したという方も多いのではないかと思います。

Cloud Build での設定例:
```yaml
steps:
  - name: gcr.io/kaniko-project/executor:latest
    args:
      # 出力先（Artifact Registry）
      - --destination=${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_IMAGE}:$SHORT_SHA
      # ビルド対象
      - --dockerfile=Dockerfile
      # キャッシュ有効化（キャッシュ用リポジトリを指定）
      - --cache=true
      # cache-repo に指定した docker registry にキャッシュが保存され、使用される
      - --cache-repo=${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/cache

substitutions:
  _REGION: asia-northeast1   # 例: asia-northeast1 / us-central1 など
  _REPO: my-repo             # Artifact Registry のリポジトリ名
  _IMAGE: myapp              # イメージ名
  ```

## Kaniko の開発停止

しかし、2025年6月、 Kaniko の GitHub リポジトリがアーカイブされ、これ以上の開発が行われないことが発表されます。

![](/images/retire-kaniko-and-use-docker-container-driver/2.png)
（https://github.com/GoogleContainerTools/kaniko の README.md ）

というわけで、いまのところまだ元気に動いている Kaniko ですが、いつ動かなくなってもおかしくはないので、他の「CI 上でも Docker レイヤーキャッシュが使える」ような新たなビルド方法を探す必要がありました。

## docker buildx build + docker-container driver に移行する

通常の `docker build` コマンドでは、ローカルで起動している Docker デーモンにバンドルされた[BuildKit](https://github.com/moby/buildkit) によってビルドされます。この Docker デーモンにバンドルされた BuildKit は、 `docker` [build driver](https://docs.docker.com/build/builders/drivers/)と呼ばれています。
build driver は `docker` だけではなく、以下の4種類が存在します。

- `docker`: Docker デーモンにバンドルされた BuildKit ライブラリを使用する
- `docker-container`: Docker を使って専用の BuildKit コンテナを作成する
- `kubernetes`: Kubernetes クラスター内に BuildKit の Pod を作成する
- `remote`: 手動で管理された BuildKit デーモンに直接接続する

これらのうち、 `docker` 以外の3種はキャッシュをエクスポート・インポートすることが可能で、 `docker build` 時に  `--cache-from` `--cache-to` というオプションを使うことができます。

![](/images/retire-kaniko-and-use-docker-container-driver/3.png)
（ https://docs.docker.com/build/builders/drivers/ より引用）
- ドキュメントには `docker` build driver でも一部の cache backend では cache export が使えると書いてあります。
    - > The default docker driver supports the inline, local, registry, and gha cache backends
    - https://docs.docker.com/build/cache/backends/
    - このドキュメントは 2024年6月に追加されている https://github.com/docker/docs/pull/20322
- しかし、 docker を最新バージョンにしたりして試行錯誤しても、 「Cache export は docker build driver でサポートされてない」旨のエラーが出てうまく動きませんでした……
    - > ERROR: failed to build: Cache export is not supported for the docker driver.


:::details docker build driver で cache export を 動作検証していたときの cloudbuild.yaml の一部

```yaml
  - id: build-image
    name: docker:28.3.3-cli
    entrypoint: 'docker'
    args:
      - buildx
      - build
      - --cache-from
      - type=registry,ref=${_REGION}-docker.pkg.dev/$PROJECT_ID/[cache_image_name]
      - --cache-to
      - type=registry,ref=${_REGION}-docker.pkg.dev/$PROJECT_ID/[cache_image_name]
      - --tag
      - ${_REGION}-docker.pkg.dev/$PROJECT_ID/[image_name]:${COMMIT_SHA}
      - --push
      - .
```

↓
こんなエラーが出る

```
Status: Downloaded newer image for docker:28.3.3-cli
docker.io/library/docker:28.3.3-cli
ERROR: failed to build: Cache export is not supported for the docker driver.
Switch to a different driver, or turn on the containerd image store, and try again.
Learn more at https://docs.docker.com/go/build-cache-backends/
```
:::


というわけで、 docker ビルドでキャッシュをエクスポート・インポートするためには、以下の手順を踏む必要がありました。

- ビルドに使う build driver を、デフォルトの `docker` から `docker-container` に変更する
- ビルドコマンドを `docker build` から `docker buildx build --builder docker-container` に変更する
- `--cache-from` `--cache-to` オプションを使い、キャッシュ先 docker registry を指定する

## 完成した cloudbulid.yaml

ここまで文字だらけで説明してきましたが、この節だけ読む方もいらっしゃると思うので、 docker buildx build によるビルドへの移行前後の cloudbuild.yaml の内容を示します。

**Before:**

```yaml
  - id: build-app
    name: gcr.io/kaniko-project/executor:latest
    args:
      - --destination=${_REGION}-docker.pkg.dev/$PROJECT_ID/app:${COMMIT_SHA}
      - --cache-repo=${_REGION}-docker.pkg.dev/$PROJECT_ID/app/buildcache
      - --cache=true
      - --cache-ttl=168h
    waitFor: ['-']
```

**After:**
```yaml
   # docker buildx build --cache-from と --cache-to を使うための準備
  - id: setup-buildx
    name: gcr.io/cloud-builders/docker:latest
    entrypoint: docker
    args:
      - buildx
      - create
      - --use
      - --name=docker-container
      - --driver=docker-container
    waitFor: ['-']

  - id: build-app
    name: gcr.io/cloud-builders/docker:latest
    entrypoint: 'docker'
    args:
      - buildx
      - build
      - --builder
      - docker-container
      - --cache-from
      - type=registry,ref=${_REGION}-docker.pkg.dev/$PROJECT_ID/app:buildcache
      - --cache-to
      - type=registry,ref=${_REGION}-docker.pkg.dev/$PROJECT_ID/app:buildcache,mode=max # mode=max: 全レイヤをキャッシュする
      - --tag
      - ${_REGION}-docker.pkg.dev/$PROJECT_ID/app:${COMMIT_SHA}
      - --push
      - .
    waitFor: ['setup-buildx']
```

- `--cache-from, --cache-to`  で、 `type=registry` のときに設定すべき値は https://docs.docker.com/build/cache/backends/registry/ を参照してください。
- Kaniko にあった `--cache-ttl` のようなオプションは、  `--cache-from, --cache-to` にはありません。別途自分で registry を定期的に掃除するなりなんなりする必要があります。

docker buildx build への移行により、  `setup-buildx` ステップが増えてしまいましたが、このステップは2〜3秒で終わり、 docker buildx build 部分は Kaniko と同じくらいのビルド時間でビルドできるようになったので、全体ではほとんどビルド時間が増えずに済みました。
