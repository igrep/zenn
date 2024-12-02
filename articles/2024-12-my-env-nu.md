---
title: "Nushellを導入したのでこんな風にプロンプトとかをカスタマイズした"
type: "tech"
emoji: "▶️"
topics: ["Nushell", "Git"]
published: true
---

この記事は[Nushell Advent Calendar](https://qiita.com/advent-calendar/2024/nushell) 3日目の記事です。

[昨日の記事](./2024-12-git-aliases-nu)で触れたとおり、私は[Nushell - 型付きシェルの基本とコマンド定義](https://zenn.dev/estra/articles/nu-typed-shell)という記事をきっかけにNushellを導入しました。今までWindowsではPowerShell、LinuxやMacではbashと使い分けていたのを一本化[^msys2]できるだけでもとてもありがたいのに、それ以上の機能が山盛りでとても頼もしい💪ですね。昨日の記事で紹介した[git-aliases.nu](https://github.com/igrep/git-aliases.nu)のような、小さなCLIアプリケーションを書くのにも便利でした。

[^msys2]: PowerShellを使い始める前はMSYS2のbashを使っていたのですが、ある時からGitの実行が急激に遅くなり、解決策も見つけられなかったのでPowerShellに切り替えました。

# ▶️普段のプロンプト[^usual]

[^usual]: 実際のところ後述のGit使用時のプロンプトで作業している時間の方が長いので、どっちが「普段」なんだよって気はしますが...。

さて今回は、掲題の通り私がそんなNushellをどのように設定したのか共有しようと思います。そうはいっても元々bash+git-shで満足していたような者なので、実際にカスタマイズしたのはプロンプトくらいです。こんな風にしました👇️。

![Nushellのプロンプト](/images/2024-12-my-env-nu/prompt.png)

色使いはデフォルトのものを踏襲しつつ、右プロンプトは削除して右プロンプトにあった情報を左に集め、左プロンプトを複数行にしています。なんでこうしたのかというと、次の理由によります:

1. 右プロンプトがあると、Neovimのターミナルから入力したコマンドを`Y`でコピーするとき右プロンプトも一緒にコピーされてしまい、邪魔になる
    - ドキュメントに貼り付ける場合など、利用習慣上そうすることが多いので、右プロンプトは削除することにしました。先進的な機能で憧れてはいたんですけどね
1. プロンプトが複数行になっていた方が、長い出力をした後スクロールして遡るときにプロンプトを見つけやすいのでは？と考えたため
    - こちらについてはまだ検証中なので、考えを変えるかも知れません
1. 入力したコマンドの折り返しを極力避けるため
    - プロンプトの末尾が`:> `だけなのはそのためです。本当はbashでそうしていたように`>`だけにしたかったんですけど、何故かうまく行かなかったのでこのようになりました。ちょっと変わった記号の並びですし、Neovimの正規表現で検索しやすく、案外いいかも知れません。

で、こちらがそのための設定ファイルの抜粋です:

```nu:env.nu
def create_left_prompt [] {
  if ($env.GIT_ALIASES_ENABLED? == true) {
    return $"(exit_code_and_time)\n(left_prompt_when_git_aliases_enabled)\n:"
  }

  let dir = match (do --ignore-shell-errors { $env.PWD | path relative-to $nu.home-path }) {
    null => $env.PWD
    '' => '~'
        $relative_pwd => ([~ $relative_pwd] | path join)
    }
  let path_color = (if (is-admin) { ansi red_bold } else { ansi green_bold })
  let separator_color = (if (is-admin) { ansi light_red_bold } else { ansi light_green_bold })
  let path_segment = $"($path_color)($dir)(ansi reset)"
  let path_segment_colored = $path_segment
    | str replace --all (char path_sep) $"($separator_color)(char path_sep)($path_color)"

  return $"(exit_code_and_time)\n($path_segment_colored)(ansi reset)\n:"
}

def exit_code_and_time []: nothing -> string {
  let last_exit_code = if ($env.LAST_EXIT_CODE != 0) {
    $"(ansi red_bold)($env.LAST_EXIT_CODE) "
  } else {
    "0 "
  }

  # create a right prompt in magenta with green separators and am/pm underlined
  let time_segment = $"(ansi reset)(ansi magenta)(date now | format date "%Y-%m-%dT%H:%M:%S%z")"

  return $"($last_exit_code) ($time_segment)"
}

# Use nushell functions to define your right and left prompt
$env.PROMPT_COMMAND = {|| create_left_prompt }
$env.PROMPT_COMMAND_RIGHT = {|| "" }
```

`nu`コマンドでNushellを最初に起動すると作成される`env.nu`を元に、元々右プロンプトを組み立てるのに使っていた処理を左プロンプトを作る関数`create_left_prompt`に移し、それぞれのパーツを改行でくっつけただけです。実際にはそれだけではなく、冒頭にある`if ($env.GIT_ALIASES_ENABLED? == true) { ... }`という箇所で、`GIT_ALIASES_ENABLED`という環境変数によって左プロンプトを切り替えています。こちらは詳しくは次の節で解説しましょう。

# ▶️Git使用時のプロンプト

`GIT_ALIASES_ENABLED`という環境変数は、[昨日の記事](./2024-12-git-aliases-nu)で紹介した拙作の`git-aliases.nu`を起動していると`true`になるものです[^line]。まさしく今回のように`git-aliases.nu`を起動しているか否かでプロンプトを出し分けたかったので実装しました。

[^line]: 具体的には[この行](https://github.com/igrep/git-aliases.nu/blob/6a192010b4eac210595a8c22a0168e577e307e02/git-aliases.nu#L31)で設定するコードを生成し、`alias`を設定するコードと一緒に`--execute`オプションに渡しています。

この`GIT_ALIASES_ENABLED`が`true`の場合に実行する`left_prompt_when_git_aliases_enabled`という関数は、次のような実装となっています:

```nu:env.nu
def left_prompt_when_git_aliases_enabled []: nothing -> string {
  use std null-device
  const refsHeadsLength = ("refs/heads/" | str length)
  let head = (git symbolic-ref -q HEAD err> (null-device) | str substring $refsHeadsLength..-1)
  let br = if ($head == "") {
    git rev-parse --short HEAD err> (null-device)
  } else {
    $head
  }

  let repo_dir = (git rev-parse --show-toplevel)
  let state_marker = if ($"($repo_dir)/rebase-merge" | path exists) {
    "(rebase)"
  } else if ($"($repo_dir)/MERGE_HEAD" | path exists) {
    "(merge)"
  } else  {
    ""
  }

  let repo_name = ($repo_dir | path basename)
  let subdir = pwd | path split | skip ($repo_dir | path split | length) | str join "/"
  let dir = $"($repo_name)/($subdir)"

  let dirty = if (git status --porcelain | lines | is-empty) {
    ""
  } else {
    "*"
  }

  let r = ^git rev-parse --quiet --verify refs/stash | complete
  let stash_dirty = if (^git rev-parse --quiet --verify refs/stash | complete).exit_code == 0 {
    "$"
  } else {
    ""
  }

  return $"(ansi yr)($br)(ansi reset)!(ansi red)($state_marker)(ansi reset)(ansi blue_bold)($dir)(ansi reset) (ansi red)($dirty)(ansi reset)(ansi red)($stash_dirty)(ansi reset)"
}
```

詳しい解説は省きますが、やっていることはリポジトリーのルートディレクトリーからの相対パスや、現在のブランチ名などを表示しているだけです。この手の仕組みはやはりNushellでも既にあるようですが、見つかった方法がイマイチ好みでなかったので、git-aliases.nuの元ネタであるgit-shのコードを参考に自前で作りました。コピペして使えるのでご自由にどうぞ。

以下、実際にgit-aliases.nuを起動したときのプロンプトのスクリーンショットです。「普段のプロンプト」で表示している情報の一部を、Gitリポジトリーの情報に差し替えています:

![git-aliases.nu起動時のプロンプト](/images/2024-12-my-env-nu/prompt-git.png)

# 👽その他: `alias`の設定

残りは、git-aliases.nuでカバーできない`alias`の設定です。現状以下の3つだけです:

```nu:env.nu
alias del = rm
alias mov = mv
alias gsh = nu ~/dot-files/nushell/git-aliases/git-aliases.nu --except=[sh]
```

`rm`・`mv`はgit-aliases.nuが潰してしまうので代わりに`del`・`mov`を設定しました。git-aliases.nuはデフォルトで既存のコマンドに対しては`nongit-`というプレフィックスを付けてバックアップをとるのですが、やはり`nongit-`では長いし、PowerShell利用時の手癖で`del`と打っていたのでWindowsの`del`などを輸入しました。もちろん本当は`mov`ではなく`move`にしたかったんですけど、[`move`はNushellの組み込みコマンド](https://www.nushell.sh/commands/docs/move.html)なので...。

最後の`gsh`は、git-aliases.nuを起動するコマンドです。git-aliases.nuのためだけに`PATH`を増やすのも大袈裟だし、Windowsだと`PATH`の設定だけでは不十分なため[^windows]、直接インストールしたパスから起動する`alias`にしました。オプションを渡して設定を変えることも出来ますしね。

[^windows]: Windowsの場合、`.nu`ファイルの関連付けを変える必要があるようです。関連付けを変えるとなるとダブルクリックしたときに起動するアプリケーションもそれに合わせないと行けませんし、現行の開発環境をセットアップするスクリプトに関連付けを変える処理を追加するくらいなら、`alias`の方が楽だろうと思ってこのようにしました。

# ☑️以上！

Nushellは高機能なので、もっと日々の活動が楽になるカスタマイズがしたいものです。何かあれば是非教えてください！
