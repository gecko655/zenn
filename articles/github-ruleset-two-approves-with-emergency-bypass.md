---
title: "GitHub Pull Request で通常時は 2 approve、緊急時だけ 1 approve でマージする"
emoji: "🚦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "GitHub"
  - "PullRequest"
  - "Review"
published: true
publication_name: "mixi"
---

## モチベーション

ある運用中プロジェクトのチームでは、 Pull Request のマージに 2 approve を必須とするルールで運用しています。
しかし、これでは深夜アラート等の対応できる人数が非常に少ないときの緊急事態に対応できない場合があります。そのため、「緊急時は 2 approve 待たずに 1 approve でマージしても良い」ということにし、 GitHub リポジトリではこれら両方の要件を満たすため、低い方の approve 数 (=1) だけ approve されればマージできる設定にしていました。

この状態だと以下のような問題が発生します。

- **1 approve のままうっかりマージしてしまう。**
  - 平常時は、人間の目視によって 2 approve あるかどうかを確認してマージする必要がある
- PR が 1 approve の時点で `Approved` 表示になってしまい、 approve が足りていないことにレビュワーが気づきにくい。
  - ↓まだ 2 approve 集まってないプルリクエストが、プルリクエスト一覧画面ではこういう表示になるため、 1 approve なのか 2 approve なのか判別できない
  - ![](/images/github-ruleset-two-approves-with-emergency-bypass/1.png)
- プルリクエストのマージ条件がすべて整ったときに自動的にマージできる [auto-merge](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/automatically-merging-a-pull-request) 機能が使えない。
  - auto-merge では、 1 approve でマージ条件が揃ったことになりマージされてしまう

## Ruleset を2つ用意する

GitHub のプルリクエストのマージを制限する機能は Ruleset といい、これによって特定のブランチへの push, merge を制限するのが一般的かと思います[^1]。

[^1]: [Classic branch protection rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) は、現在では新規作成非推奨なので、もうみんな使ってないですよね…？

https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets

この Ruleset は、実は1つのブランチに複数設定することができます。複数設定すると、そのすべてのルールを満たしたときに限りブランチへマージすることができます。

> A ruleset does not have a priority. Instead, if multiple rulesets target the same branch or tag in a repository, the rules in each of these rulesets are aggregated. If the same rule is defined in different ways across the aggregated rulesets, the most restrictive version of the rule applies.
> https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets#about-rule-layering

また、 Ruleset は [bypass list](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#granting-bypass-permissions-for-your-branch-or-tag-ruleset) を設定することができ、必要なタイミングではルールを無視してプルリクエストをマージすることができます。

したがって、これらを組み合わせて次のように Ruleset を2つ作成し、同じブランチに設定することで、「平常時は 2 approve」「bypass したときだけ 1 approve でマージできる」が設定できます。

- Rule 1:
  - Bypass list を空欄にする
  - 1 approve でのマージを許可
  - （その他色々なルールもここに書く）
- Rule 2:
  - Bypass list を、深夜アラート対応を行なう可能性がある全員にする
    - ここではリポジトリの Admin や Maintainer などのロールとは関係なく、ユーザーグループを直接セットできるので、 bypass できるようにするユーザーに bypass 以外に強い権限を付与せずに済んで便利です。
  - 2 approve でのマージを許可

![](/images/github-ruleset-two-approves-with-emergency-bypass/2.png)
↑ bypass rules のチェックボックスが表示され、まだ approve が足りてないので普通はマージできないけど、 bypass しようと思えば誰でも bypass できる状態

## まとめ

bypass なし ＆ bypass ありの2つの Ruleset を設定することによって、緊急時の 1 approve でのマージを許しつつうっかりミスによる 1 approve でのマージを防ぐことができるようになり、レビューも円滑に進み平和になりました。おしまい。

## スクリーンショット集

今後 Ruleset を設定する方々のため、いろんなスクリーンショットを残しておきます。

（スクリーンショットは業務外の適当なリポジトリで撮影したもので、業務用リポジトリの設定とは異なる場合があります）

### 設定スクリーンショット

`https://github.com/[org]/[repo]/settings/rules` で設定できる。

#### Rule 1 （Bypass 不可、 1 approve 必須、その他必ず守らせたい設定色々）

![](/images/github-ruleset-two-approves-with-emergency-bypass/3.png)

#### Rule 2 （深夜アラート対応を行なう全員[^2] が Bypass 可、2 approve 必須）

[^2]: ここでは撮影の都合で、リポジトリの Write Role を持つ全員に bypass 権限を与えていますが、ユーザーグループに与えたほうが良いと思います。

![](/images/github-ruleset-two-approves-with-emergency-bypass/4.png)

### プルリクエスト画面スクリーンショット

#### 0 approve の プルリクエスト

`At least 1 approving review is required by reviewers with write access. At least 2 approving reviews are required by reviewers with write access.` と表示され、どうやってもマージできない。

![](/images/github-ruleset-two-approves-with-emergency-bypass/5.png)

#### 1 approve のプルリクエスト

赤文字で `Merge without waiting for requirements to be met (bypass rules)` と表示され、クリックするとマージができるようになる。

![](/images/github-ruleset-two-approves-with-emergency-bypass/6.png)

↓ bypass rules にチェックを付け、マージできるようになった時の様子

![](/images/github-ruleset-two-approves-with-emergency-bypass/7.png)

#### 2 approve のプルリクエスト

マージできる。

![](/images/github-ruleset-two-approves-with-emergency-bypass/8.png)
