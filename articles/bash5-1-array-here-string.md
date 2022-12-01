---
title: "Bash5.1の仕様変更に巻き込まれてCIが落ちた話"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Bash"]
published: true
publication_name: "mixi"
---

# はじめに
この投稿はZennへの投稿テストを兼ねています。
https://qiita.com/gecko655/items/5cae0c8f48854c6bca64 と全く同じ内容です。

# 発生したこと
bashでは以下のようにarrayを定義することができます。
```bash
$ arr=("a" "b")
$ echo ${arr[0]}
a
$ echo ${arr[1]}
b
$ echo ${arr[@]}
a b
```
https://www.gnu.org/software/bash/manual/html_node/Arrays.html

arrayに空文字が含まれていると、想定したとおりに動きます。
```bash
$ arr=("" "a")

$ echo ${arr[0]}

$ echo ${arr[1]}
a
$ echo ${arr[@]}
a
```

しかし、ここでarray変数に対して[here stringリダイレクト](https://www.gnu.org/software/bash/manual/html_node/Redirections.html#Here-Strings)をしたときに、bash5.0以前と5.1以降で異なる挙動をするようです。
例えば↓を実行してみます。
```bash
arr=("" "a")
tee -a <<< ${arr[@]} | hexdump -C # どのようなバイト列が得られたのかを表示する
```

（ここから先の検証ではdockerを使います。便利な世の中ですね）

```bash
# Bash 5.0
$ docker run --rm bash:5.0 bash -c 'arr=("" "a"); tee -a <<< ${arr[@]} | hexdump -C'
00000000  61 0a                                             |a.|
00000002

# Bash 5.1
$ docker run --rm bash:5.1 bash -c 'arr=("" "a"); tee -a <<< ${arr[@]} | hexdump -C'
00000000  20 61 0a                                          | a.|
00000003
```

Bash 5.1 では、 **[スペース](https://en.wikipedia.org/wiki/Space_(punctuation)#Encoding)を表す `20` が先頭に出力されるようになりました**。
空文字がarrayに含まれているときのhere stringリダイレクト周りの挙動が、Bash 5.0〜5.1の間で変更されたようです。

- Bash 5.1のリリースノートを読んだところ、↓この部分がこの仕様変更と対応しているっぽい
    - https://lists.gnu.org/archive/html/info-gnu/2020-12/msg00003.html

> c. Here documents and here strings now use pipes for the expanded document if
it's smaller than the pipe buffer size, reverting to temporary files if it's
larger.

- bashのコミットログは5.0〜5.1の差分を1コミットにsquashされているため、どの差分がこの仕様変更に関わっているのかよくわからない
  - https://git.savannah.gnu.org/cgit/bash.git/commit/?id=8868edaf2250e09c4e9a1c75ffe3274f28f38581

- array変数をhere stringでリダイレクトするのではなくarray変数をファイル出力すると、Bash 5.1のhere stringリダイレクトの挙動と同様に先頭にスペース( `20` )が入る。
```
$ docker run --rm bash:5.0 bash -c 'arr=("" "a"); echo "${arr[@]}" > tmpfile; cat tmpfile | hexdump -C '
00000000  20 61 0a                                          | a.|
00000003

$ docker run --rm bash:5.1 bash -c 'arr=("" "a"); echo "${arr[@]}" > tmpfile; cat tmpfile | hexdump -C '
00000000  20 61 0a                                          | a.|
00000003
```
つまり……
| x |  bash 5.0 | bash 5.1 |
| -- | -- | -- |
| 空白つきarrayをhere stringする | **スペースが入らない** |  スペースが入る |
| 空白つきarrayをファイル出力する | スペースが入る |  スペースが入る |

Bash 5.0以前でhere stringリダイレクト時にarrayの空文字要素が出力されなかったのは不具合っぽい？


Bashのバージョン差（しかもマイナーバージョン）の影響を受けることなんてあるんだな〜と思いました（小並感）。

# Q&A
## なんでBashのバージョンをアップデートしたの？

能動的にBashのバージョンをアップデートしようとしたわけではなく、CI(GCP cloud build)で使っているBashのバージョンが勝手にアップデートされてしまい、今回の問題が発生しました。

durianのCIでは、↓にある「CloudBuildに初めから存在するDockerImage」を使うことで、ビルド用Docker Imageをpullしないで済むようにしています。
- 具体的には `gcr.io/google.com/cloudsdktool/cloud-sdk:cloudbuild_cache` を使っています。

https://qiita.com/Gin/items/7a21f5ed7dee339ea131

このイメージはなんらかのDebianっぽいLinuxになんらかのBash及び[cloudsdk](https://cloud.google.com/sdk/docs)がインストールされているイメージで、`cloudbuild_cache` タグはcloud buildのジョブ内以外では見えないタグです。そのため、このタグは任意のタイミングで更新される可能性があります。また、最新のバージョンが使われている保証もありません。
- cloudbuild_cache タグのイメージについては特にGCPのドキュメントに書いてあるような仕様ではないのですが、Googleの方に直接聞いたところ、「最新のイメージが使われること等は保証できないが、使っても大丈夫」との返答をいただいています。

このイメージが2022-10-29〜2022-10-30の間に更新され、Bashのバージョンが5.1に上がったため、 **なにもしてないのにCIが落ちるようになった** のでした。

## この仕様変更でCIが落ちるってどんなCI実装だったの？

空arrayを初期化する際に、
```bash
arr=()
```
とすべきところが、なぜか
```bash
arr=("")
```
になっていました。

`arr=("")` では長さ0の配列ではなく、空文字の要素が入っている長さ1の配列が定義され、Bash 5.0以前ではこれらの2つに挙動の差がなく気が付かなかったのですが、Bash 5.1にアップデートされたことで問題が顕在化しました。

