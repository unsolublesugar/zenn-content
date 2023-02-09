---
title: git status がコード 128 で終了しました error bad signature fatal index corrupt
emoji: "🤯"
type: "tech"
topics: [SourceTree, git, windows]
published: true
---
# 備忘録
直近10年ほどの記憶内で初めて遭遇したエラーだったため書き残しておく。

ちょっとしたファイル変更を行おうとしたタイミングでPCが落ちてしまい、SourceTree上で以下のようなエラーが発生した。どうやらgitのindexファイルが破損してしまった模様。

![](https://storage.googleapis.com/zenn-user-upload/28e6c10c2e73-20230209.png)

> 'git status' がコード 128 で終了しました: error: bad signature
>  fatal: index file corrupt

他にもSourceTree上のHistoryを開こうとするとコミットログが死んでいたり、リモートリポジトリからのフェッチもできなくなる等、対象ディレクトリ配下のgit操作が色々と終わっていた。

![](https://storage.googleapis.com/zenn-user-upload/f8d1323a4b8f-20230209.png)



何もしてないのに壊れた／(^o^)＼

# 解決手段を探る

日本語でエラーメッセージをググるのも良いが、SourceTree等の海外製ツールは英文で情報を辿るのが定石である。

> 'git status' failed with code 128: error: bad signature
> fatal: index file corrupt

ピンポイントでAtlassianのコミュニティフォーラムへの投稿が見つかった。

https://community.atlassian.com/t5/Sourcetree-questions/git-status-failed-with-code-128-error-bad-signature-fatal-index/qaq-p/207697#M6023 


冒頭のエラーメッセージに記載のとおり、indexファイルに問題が起きているため、一旦indexファイルを削除してリセットすれば良さそうだ。

```
rm -f .git/index
git reset
```

いくつかの日本語情報を見ても大抵これで解決しているようだが…残念ながら手元の環境では症状が改善しなかった。

# リポジトリをクローンし直す
今回はリモートリポジトリとの大きな差分もなく、ローカルのコミットログを破棄しても特に問題なかったため、サクッとリポジトリをクローンし直すという判断をした。

SourceTreeを使用している場合は、対象リポジトリのブックマークを削除してから、ローカルリポジトリのディレクトリを削除する。

https://jira.atlassian.com/browse/SRCTREE-4095 

改めてリモートリポジトリをクローンすることで、何事もなかったかのようにあっさりエラーが解消された。

# 詳細原因を探る方法

今回は作業時間が惜しかったこともあり、エラー原因を詳しく深掘ることはしなかったが、あとから調べてみると`git fsck`で具体的な破損箇所の特定を行うことができると知った。

https://git-scm.com/docs/git-fsck

`git fsck`はデータベース内のオブジェクトの整合性を検証するコマンドで、問題のあるファイルを検出し、修正・除外する形でエラーを解消していく手法のようだ。Stack Overflowで類似エラーの検証に`git fsck`を使っているケースを見かけた。

https://stackoverflow.com/questions/20264032/git-status-shows-fatal-bad-object-head


Zennでも`git fsck`を使ってgitリポジトリの復旧作業をされていた方が居たため、参考までにリンクを貼らせていただきます。
https://zenn.dev/oshanqq/scraps/253ca171de4bdf

滅多に遭遇する事象ではないだろうが、また似たような状況に陥った際はじっくりと調査してみたい。こういう事例を見るとgitの奥深さに感嘆しますね。