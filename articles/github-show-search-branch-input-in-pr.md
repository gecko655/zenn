---
title: "2024-06-21 GitHub の Pull Request 作成画面で branch の検索窓が表示されなくなった問題の、一時回避策"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github"]
published: true
---

この記事を執筆している時点で、 github.com では Pull Request の対象 branch の検索窓が表示されない不具合が発生しています。

![](/images/github-show-search-branch-input-in-pr/1.png)

https://twitter.com/search?q=github%20%E3%83%96%E3%83%A9%E3%83%B3%E3%83%81&src=typed_query&f=live

とりいそぎ、 ↓のスクリプトをブラウザのコンソール上に打ち込むと、検索窓が復活します。

```js
$("#base-ref-selector .SelectMenu-modal input-demux").prepend($("#base-ref-selector .SelectMenu-filter"));
$("#head-ref-selector .SelectMenu-modal input-demux").prepend($("#head-ref-selector .SelectMenu-filter"));
```

![](/images/github-show-search-branch-input-in-pr/2.png)


## 追記 2024-06-22

もう直ったらしい。
