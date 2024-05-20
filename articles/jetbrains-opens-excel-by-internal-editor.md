---
title: "IntelliJ 等の Jetbrains 系 IDE で、エクセルファイルを内部エディタで開かないようにする"
emoji: "💺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["IntelliJIDEA", "JetBrains", "Excel"]
published: true
publication_name: "mixi"
---

解決方法が謎だったのでメモ

## エクセルファイルが 2024.1.1 系 IDEで内部エディタで開かれるようになった

2024年4月30日ごろ、 IntelliJ IDEA 等の Jetbrains 系 IDE のバージョン 2024.1.1 バージョンがリリースされました。
https://blog.jetbrains.com/idea/2024/04/intellij-idea-2024-1-1/

このアップデートで、エクセルファイル( .xlsx)が IDE 付属のテーブルデータエディタで開かれる機能が追加（？[^1]）されましたが、この機能はなんだか不具合が多く使いづらいです。

[^1]: 追加されたのか、以前から機能が存在したが2024.1.1 でデフォルト挙動が変わったのかは謎

↓エクセルを内部エディタで開いたものの、エクセル内部で使われている関数（簡単な VLOOKUP）が解釈できずに何も表示できない様子

![](/images/jetbrains-opens-excel-by-internal-editor/1.png)

このように中身を表示できないようなエクセルファイルでは、 "Open in -> Open in Associated Application" を選ぶことで以前と同様に Excel のアプリケーションでエクセルファイルを開くことができます。
![](/images/jetbrains-opens-excel-by-internal-editor/2.png)
しかし、毎回 "Open in Associated Application"を選択しないと Excel アプリが開かないのはめんどくさく、内部エディタが開くことで恩恵を受けることもほとんど無いので、できればデフォルトでファイル名のダブルクリック時等に Excel アプリを開くようにして、内部エディタはデフォルトでないようにしたいです。

## Excel ファイルのデフォルトオープン挙動を Excel アプリにする
Settings → Advanced Settings → Database にある "Open file as table if detected by scripted loader" のチェックを外すと、内部エディタが無効になり、 Excel ファイルをダブルクリックすると Excel アプリが開くようになります。 _なぜこのオプションがExcelファイルのオープン挙動に関係しているのかは謎です。_
![](/images/jetbrains-opens-excel-by-internal-editor/3.png)


## 参考

↓解決方法が書いてあったチケット
https://youtrack.jetbrains.com/issue/DBE-20207/Data-Editor-and-Viewer-does-not-respect-open-in-associated-application-file-type

↓同内容を質問していたコミュニティサポートの質問（回答しておいた）
https://intellij-support.jetbrains.com/hc/en-us/community/posts/18960222140050-How-to-turn-off-internal-editors-viewers?page=1#community_comment_19026495239826
