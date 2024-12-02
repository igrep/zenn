---
title: "Nushell向けgit-sh、git-aliases.nuを作った"
type: "tech"
emoji: "👾"
topics: ["Nushell", "Git"]
published: true
---

この記事は[Nushell Advent Calendar](https://qiita.com/advent-calendar/2024/nushell) 2日目の記事です。

[git-aliases.nu](https://github.com/igrep/git-aliases.nu)という、[Nushell](https://www.nushell.sh/ja/)でのGitの使用を少し便利にするスクリプトを書いたので、Nushellをお使いの方は試してみてください。Nushellをお使いでない方は[Nushell - 型付きシェルの基本とコマンド定義](https://zenn.dev/estra/articles/nu-typed-shell)などを参考に、Nushellと一緒に使ってみてください！

# 🌇背景

`git status`とか`git branch`とかいちいち`git`と打つのが面倒なので、`gst`とか`gb`とか、そんな感じのエイリアスを作っている人は多いかと思います。私はそうした問題に対応するために、これまで[git-sh](https://github.com/rtomayko/git-sh)というアプリケーションと次のようなGit組み込みの`alias`を併用していました:

```ini:~/.gitconfig
# git config --global alias.bra branch などと書くことで設定できる
# 以下は特によく使うもの
[alias]
  amend    = commit --verbose --amend
  bra      = branch
  ci       = commit
  co       = checkout
  d        = diff
  dc       = diff --cached
  disc     = diff --ignore-space-change
  discc    = diff --ignore-space-change --cached
  # ... 以下略 ...
```

git-shは、Gitの多くのサブコマンドを`git`抜きで入力できるよう`alias`を設定したり、プロンプトをGitリポジトリー向けにカスタマイズしたりしたbashです。もう何年も前からメンテされていませんが、私はこの単純かつ大胆なアプローチを気に入り、Nushellに出会って今回紹介するスクリプトを作るまでずっとgit-shを使い続けていました。WindowsでPowerShell中心で開発していたときも[似たようなスクリプト](https://gitlab.com/igrep/pwsh-git-aliases)を作っていたほどです。

そして、[Nushell - 型付きシェルの基本とコマンド定義](https://zenn.dev/estra/articles/nu-typed-shell)という記事をきっかけにNushellを使うようになったので、今回いわばNushell版git-shとも言うべきスクリプトを作りました。

[igrep/git-aliases.nu: Launch Nushell with aliases for all git commands, inspired by git-sh.](https://github.com/igrep/git-aliases.nu)

これと先程のaliasと併せれば、次のように実行することで`Git`が楽に使えます:

```nu
> git-aliases.nu

> st
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean

> bra
* main
```

使い方などは上記のリポジトリーのREADMEをご覧ください。単一の`.nu`ファイルのみで動作するので、`PATH`の通ったディレクトリーに置いて`chmod +x`する[^windows]だけで使えます。名前から察せられるかも知れませんが、元ネタのgit-shと異なりあくまで`git`のサブコマンドに対するエイリアスを提供するだけなので、プロンプトのカスタマイズは行いません。プロンプトをGit向けに書き換える機能は他が既に提供していますし、「一つのことをうまくやる」方が良いだろうと考えた結果です[^prompt]。

[^windows]: Windowsの人は.nuファイルの関連付けを変えるなど、運用上少しやりづらいやり方を採らないといけないかも知れません。かく言う私はLinux・Mac含め別の方法を採用しています。詳しくは明日投稿する「Nushellを導入したのでこんな風にプロンプトとかをカスタマイズした」をご覧ください。

[^prompt]: 結局、私は既存の方法があまり好きになれそうになかったので、git-shのコードを参考に自前で作りました。詳しくはこちらも明日の記事で。

# ⚡️少し苦労した点

最後に、このスクリプトを作るにあたって少し苦労した点を共有させてください。最初にこのスクリプトを作る際、私は先程触れたPowerShell向けgit-shみたいなスクリプト、[pwsh-git-aliases](https://gitlab.com/igrep/pwsh-git-aliases)と同様に、`alias`を設定するスクリプトを実行時に生成した上で`eval`（PowerShellなので実際には`Invoke-Expression`）することで実現しようと考えていました。ところが、設計思想によりNushellには`eval`がないのです！

[Nushell公式のドキュメントより](https://www.nushell.sh/book/thinking_in_nu.html#think-of-nushell-as-a-compiled-language):

> An important part of Nushell's design and specifically where it differs from many dynamic languages is that Nushell converts the source you give it into something to run, and then runs the result. It doesn't have an eval feature which allows you to continue pulling in new source during runtime. This means that tasks like including files to be part of your project need to be known paths, much like includes in compiled languages like C++ or Rust.
>
> For example, the following doesn't make sense in Nushell, and will fail to execute if run as a script:
>
> ```nu
> "def abc [] { 1 + 2 }" | save output.nu
> source "output.nu"
> abc
> ```
>
> The source command will grow the source that is compiled, but the save from the earlier line won't have had a chance to run. Nushell runs the whole block as if it were a single file, rather than running one line at a time. In the example, since the output.nu file is not created until after the 'compilation' step, the source command is unable to read definitions from it during parse time.

要するにNushellは、`use`や`source`するものも含め、実行前にソースコードをチェックした上でコンパイルし、実行するという仕様なので、実行中に組み立てたソースコードを`eval`したり、引用した例のように`save`で書き込んだソースコードを直後に`source`したりするような、**動的にソースコードを生成・評価して環境を書き換える術がない**のです。

これはNushellの安全性を高めたり、時には最適化したりするのに役に立つ特徴でしょう。もしpwsh-git-aliasesのようなスクリプトが悪さして、`alias cp = rm -rf`みたいなコードをこっそり生成して`eval`させようとしても、Nushellならばそもそもそれが不可能になっています。自分で明示的に`alias`や`source`, `use foo *`など[^def-env]と書かない限り`cp`はあくまで`cp`のままですし、仮にそうした悪意を持ってコマンドの定義を書き換えるスクリプトがあったとしても、`source`したり`use`したりするファイルの**コードを読めば事前に分かる**ようになっています。

[^def-env]: 後、`def --env`を使って定義された関数の実行や、`def`を使い`cp`という名前でコマンドを定義することも該当します。まだあるかな？

以上の通りNushellは、`eval`などによる動的に生成したソースコードの実行を設計上不可能にしています。ところが、git-aliases.nuのように`git`のサブコマンドを全て列挙[^common-aliases]してそこから`alias`するコードを生成し、実行するアプリケーションでは、その性質が障害となりました。何度か試行錯誤してもうまく行かず、悩んでいたその日の夜、床に就く直前に解決策が閃きました💡。

[^common-aliases]: 「わざわざ`git`のサブコマンド全部に`alias`を設定しなくても、よく使うものだけ設定すればよいのでは？」と思った方もいらっしゃるかも知れません。ぶっちゃけその通りです。git-shも実際そうしているし全くもってその通りなのですが、まあ私がやりたかったということでどうかお許しください😅。

その解決策とは、Nushellの本体、`nu`コマンドにある`--execute`というオプションを使った方法です。`--execute`（短縮形は`-e`）オプションは、シェルのセッションを始める前に指定したコードを実行します。例えば次のように`--interactive`オプションと組み合わせれば、`--execute`に渡したコードを`eval`のように実行した上でNushellを起動することが出来ます:

```nu
> nu --interactive --execute "alias hello = print 'hello'"
> hello
```

前作のpwsh-git-aliasesと異なり、必ず`nu`を子プロセスとして起動することになるので、終了したら`--execute`で定義したコードは失われてしまいますが、それで困る人はまずいないでしょう。

そんなわけで「これは、Nushellにマクロみたいな機能がないとダメなのでは？」などと悩んだりもしましたが、全くその必要なく期待したとおりの実装に出来ました。良かったらNushellと一緒に使ってみてください！👋
