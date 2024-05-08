---
title: "GitHub Actions は、 default ブランチで実行しておかないとキャッシュが効かないことがある"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHubActions", "GitHub"]
published: true
publication_name: "mixi"
---

## 要約
- GitHub Actions で pull_request イベントにフックして起動するジョブを定義したが、キャッシュを使うようにジョブ定義したのに実際は使えていなかった
- このジョブ定義では pull_request イベントでのみジョブを実行していたが、その場合は pull request の head ブランチ、 base ブランチ、及び default ブランチでビルドされたキャッシュしか使うことができない
- default ブランチへの push 時でもビルドを行うよう修正して解決した

## 発生した現象
あるゲームプロジェクトでは、ゲームのマスターデータを GitHub リポジトリで管理しており、テキストファイルでコミットされています。
このマスターデータの整合性チェックを GitHub Actions の pull_request イベントのジョブで実行することで、 PR マージ前にデータの問題に気づけるようにしています。

- validator の実装は validator ディレクトリの中にあり、 golang で書かれています。
- validate したいデータは data ディレクトリにテキストファイルで置いてあります。

具体的には、以下のような GitHub Actions のジョブを定義して使っていました。

```yml
name: Validator Action
on:
  pull_request:
    branches: [ master ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: 1.22
          cache-dependency-path: |
            validator/go.sum
      - name: run validator
        run:
          cd validator && go run main.go ../data
```
- `actions/setup-go` は内部で `actions/cache` を使っており、 go.sum でロックされた依存関係を勝手に cache してくれます。
    - https://github.com/actions/setup-go#caching-dependency-files-and-build-outputs
    - validator の実装は頻繁に変わることや、 go.sum の依存関係のみ cache できればビルドは数秒でおわることから、go のビルドバイナリ自体は cache していません。
- キャッシュキーは [`setup-go-${platform}-${linuxVersion}go-${versionSpec}-${fileHash}`](https://github.com/actions/setup-go/blob/cdcb36043654635271a94b9a6d1392de5bb323a7/src/cache-restore.ts#L34)
    - e.g. `setup-go-Linux-ubuntu22-go-1.22.2-a32690c44e54ba350b0456a5dce2309910719b280e66bb84c1a06643ae3180c9`
    - 末尾の `fileHash` 部分は go.sum ファイルの SHA256 hash

このジョブ定義は一見問題なく動きそうに見えますが、実際にプルリクエストを作成して、PR によってキックされたジョブのログを見てみると、キャッシュキーが変わるような変更をしていないのになぜか cache を使えてないジョブが頻繁に発生します。
![](/images/build-github-action-workflow-on-default-branch/1.png)

また、ジョブを再実行すると必ず cache を使ったジョブが走ります。
![](/images/build-github-action-workflow-on-default-branch/2.png)

## 原因
GitHub Actions の cache のドキュメントをよく読むと、以下のようなことが書いてあります。

> Access restrictions provide cache isolation and security by creating a logical boundary between different branches or tags. Workflow runs can restore caches created in either the current branch or the default branch (usually main). If a workflow run is triggered for a pull request, it can also restore caches created in the base branch, including base branches of forked repositories.
https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache

GitHub Actions の cache は、以下のいずれかのブランチで起動したジョブによって作成されたキャッシュでないと利用できないようでした。
- 1: そのジョブが起動したブランチ
- 2: default ブランチ
- 3: (pull_request イベントで起動したジョブの場合) そのプルリクエストの base ブランチ(マージ先ブランチ)

今回のような pull_request イベントでのみ起動するジョブにおいては、 default ブランチ（master ブランチ）ではジョブは起動することがないため 2,3 は使えず、プルリクエスト作成後初回のジョブでは毎回キャッシュがない状態でビルドせざるを得ない状態になっていました。

## 対策
master ブランチへの push 時もビルドを行うよう修正しました。
これで、プルリクエスト作成後初回のジョブで、 cache が効くようになりました。

![](/images/build-github-action-workflow-on-default-branch/3.png)
↑ ビルド時間が40秒ほど短くなった様子。

```diff
name: Validator Action
on:
  pull_request:
    branches: [ master ]
+ push:
+   branches: [ master ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: 1.22
          cache-dependency-path: |
            path/to/go.sum
      - name: run validator
        run:
          cd path/to && go run main.go ../data
```

- 今回のジョブは GitHub Action のイベントペイロードのうち `github.event.pull_request` の部分をジョブ中で使っていなかったので簡単に `push: { branches: [ master ]}` を追加できたが、使っていた場合は push イベントで起動した際にエラー発生するので対策する必要があります。
- GitHub Actions のキャッシュは 「アクセスされずに7日経過」「合計サイズ10GB超過」のような条件によって消えるため、アクティブに開発中なリポジトリであれば、1度だけ default ブランチでビルドしておけばしばらくキャッシュが持つはずです
    - https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#usage-limits-and-eviction-policy
    - 年末年始や GW などで長期間だれも作業していないタイミングがあったり、他のジョブで大きなサイズのキャッシュを保存したりするとキャッシュが消えるかもしれませんが、デフォルトブランチへの push を1回行ってしまえば、キャッシュが動いていないことに気づいていなくてもいつかはキャッシュが復活するようにしています。
      - ちゃんとやるなら [schedule](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule) とか使って定期的にビルドが走ることを保証してもいいかもしれない。

## 備考
この記事のほぼ全ての内容は https://medium.com/@everton.spader/how-to-cache-package-dependencies-between-branches-with-github-actions-e6a19f33783a を参考にしたものです。
