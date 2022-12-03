---
title: "GitHubのSlack通知がいきなりスレッドに投稿されるようになり不便なので苦情を送ったら、スレッド機能を無効化できるようになった話"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "Slack", "GitHub Issue"]
published: false
publication_name: "mixi"
---

※この記事は、MIXI社内で2022-10-18から公開し、更新していた社内向け記事を、一般公開向けに修正したものです。

※追記が多くて読みづらいかもしれません。お急ぎの方は[#追記 2022-12-03 もとのスレッドを使わない機能に戻せるようになった](#追記-2022-12-03-もとのスレッドを使わない機能に戻せるようになった)まで飛ばしていただければと思います。

## 経緯
2022-10-18以降、GitHubのSlack連携に以下の変更が行われました。

- Create issues as you collaborate (会話の流れでIssueを作成できる)

![](/images/github-slack-01.png)

- Issue card updates and threading (Issue通知のSlack投稿からIssueを更新(=コメント, 編集, クローズ)できる & **Issueコメントがスレッドに流れる**)

![](/images/github-slack-02.png)

https://github.blog/changelog/2022-10-19-github-app-in-slack-issue-create-and-manage-experience/
https://github.com/integrations/slack#threading

この「Issueコメントがスレッドに流れる」機能が非常に厄介で、僕の知る限り以下のような問題があります。
- Issueコメントがチャンネルに流れないので、Issueにアサインされている人以外はIssue上でどのような会話がされているのかわからなくなった。
- 逆に、Issueにアサインされている人はIssueにコメントが来るたびに通知が飛ぶので、通知がうるさくなった。
- もともとSlackの"threads"欄は人間との会話に使っていたのに、GitHubの通知で"threads"欄が流されるようになってしまい、人間との会話が追えなくなった。
  - ![](/images/github-slack-03.png)

- （今回のアップデート直後のみ、）アップデート以前に作られたIssueに対してコメントがあったときは、"Issue Created"のSlack投稿を再度流した上でその投稿のスレッドにコメントされるようになった（とてもうるさい）

## 苦情

あまりに耐えられない仕様だったため、GitHub slack連携のGitHub repositoryに、Feature RequestとしてIssueを立てました。
- 当時、Twitterを見た限りでは日本人しか騒いでおらず、誰かがIssueを書かないと進展しなそうだなと思い、あんまり英語に自信はなかったんですが頑張って英作文しました……

![](/images/github-slack-04.png)

https://github.com/integrations/slack/issues/1500

今日(10月27日)時点で :+1: が328個、Issueへの参加者（発言者）が26人ついており、多くの方々が同じ思いであることがわかります。

その後メンテナーの方から返信があり、最初は「チャンネル上に投稿されるノイズを減らして整理するためにこの変更をしたんだけど、Issueにアサインされていれば引き続きSlackの機能で通知が飛ぶはずだよ〜〜 :wave: :pray: 」[1]のような、さっぱり問題を理解してない感じの反応だったのですが、

[1] ![](/images/github-slack-05.png)

https://github.com/integrations/slack/issues/1500#issuecomment-1281990344

このコメントに対してリアクションと反論が飛び交った結果、 **「持ち帰って検討する」** [2]、 **「2,3週間のうちにスレッド機能をオフにできるようにする」** [3]との回答が返ってきて、

[2] ![](/images/github-slack-06.png)

https://github.com/integrations/slack/issues/1500#issuecomment-1282109259

[3] ![](/images/github-slack-07.png)

https://github.com/integrations/slack/issues/1500#issuecomment-1283966644

さらにこの日程も早まって **「今週末にはスレッド機能をオフにできるようにしたいと思っている」** と、スレッド機能のリリースから数えて1週間半ほどでスレッド機能がオフにできるよう実装される旨が告知されました。

![](/images/github-slack-08.png)
https://github.com/integrations/slack/issues/1500#issuecomment-1291155069

その後、10/27 23時ごろに、「スレッド機能をオフにする」機能 **ではなく、「Issueコメントは引き続きスレッドに投稿されるが、Issueコメントがチャンネルにも投稿される」機能が追加されました**。
違う、そうじゃない……

![](/images/github-slack-09.png)

https://github.com/integrations/slack/issues/1500#issuecomment-1293587491

この機能を有効にした時の様子：

![](/images/github-slack-10.png)

![](/images/github-slack-11.png)

## Issueコメントがチャンネルにも投稿されるようにする

Issueコメントがチャンネルにも投稿されるようにするには、以下のようなコマンドを実行してください。

```
/github subscribe org/repo comments:"channel"
```

`org/repo` は各々リポジトリ名に置き換えてください。
なお、デフォルトでは（ `/github subscribe org/repo comments` を実行しただけでは）チャンネルには投稿されず、明示的に `comments:"channel"` を指定する必要があります。

---

この機能によって、冒頭で書いたスレッド機能の4つの問題のうちの1つが解決します。

- ~~Issueコメントがチャンネルに流れないので、Issueにアサインされている人以外はIssue上でどのような会話がされているのかわからなくなった。~~ →解決
- 逆に、Issueにアサインされている人はIssueにコメントが来るたびに通知が飛ぶので、通知がうるさくなった。 →解決していない。アサインされているIssueにコメントが来ると通知が飛ぶ
- もともとSlackの"threads"欄は人間との会話に使っていたのに、GitHubの通知で"threads"欄が流されるようになってしまい、人間との会話が追えなくなった。 → 引き続きthread欄は荒らされます。
- （今回のアップデート直後のみ、）アップデート以前に作られたIssueに対してコメントがあったときは、"Issue Created"のSlack投稿を再度流した上でその投稿のスレッドにコメントされるようになった（とてもうるさい） → 引き続きチャンネル上で"Issue Created"のコメントが流れるようです。

threads欄が荒らされる問題は解決していないので、引き続きIssueで苦情をお伝えしないといけないなと思っています。
To be continued....

## 追記 2022-11-24
メンテナーの方から返信があり、 **スレッドを介さずにGitHubのSlack通知が設定できるようにする機能**を11月末を目処に実装するとのことでした。
これでようやくスレッド欄が荒れなくなりそう :tada: :tada:

> Based on the popular request, we are proving a self-service way to turn off threading in the channels where you feel it more of a noise. We have been working on this feature for the past few weeks. We will ship this feature by end of this month (Nov-30).

https://github.com/integrations/slack/issues/1500#issuecomment-1326149981

ちなみに、現時点でこのIssueには400近い :+1: と36人のコメント投稿者が居る。
![](/images/github-slack-12.png)

## 追記 2022-12-03 もとのスレッドを使わない機能に戻せるようになった

スレッド機能を完全にオフにし、以前のような挙動でGitHubのSlack通知を投稿できるようになりました！ :tada: :rocket:

`/github settings` コマンドを打つと、↓のような設定画面が表示され、"Disable"をクリックするとスレッド機能がオフになります。
![](/images/github-slack-13.png)

- 設定はチャンネルごとのため、この設定を行いたいすべてのチャンネルで `/github settings` -> "Disable" をクリックする作業が必要。

https://github.com/integrations/slack/issues/1500#issuecomment-1335564029
