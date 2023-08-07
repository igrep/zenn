---
title: "Vimの検索で、都合がいいときだけ大文字小文字を区別する"
type: "tech"
emoji: "🔡"
topics: ["vim"]
published: true
---

この記事は、[Vim駅伝](https://vim-jp.org/ekiden/) 2023年8月7日の記事です。

一言で言うと「[`:help \c`](https://vim-jp.org/vimdoc-ja/pattern.html#/\c)」をご覧ください、で済む話なのですが、検索にもヒットしにくいお話なのでこの場を借りて共有します。

# 問題: `:set smartcase`（あるいは`ignorecase`）じゃ困る時もある

`/`などで検索する際、「検索文字列に大文字が含まれている場合のみ大文字小文字を区別する（それ以外は区別しない）」という設定、`smartcase`は便利ですが、「小文字だけの文字列にヒットさせたい！」という場合も時にはあるでしょう。

# 解決策: 検索文字列に`\C`（バックスラッシュ、大文字のC）を含める

見出しの通り、`\C`という文字列を検索する文字列に含めると、`smartcase`や`ignorecase`をONにしていたとしても大文字小文字を区別してマッチするようになります。次のスクリーンショットは[hlsearch](https://vim-jp.org/vimdoc-ja/options.html#'hlsearch')を有効にした上で、手元のNeovimにおける[`:help \c`](https://vim-jp.org/vimdoc-ja/pattern.html#/\c)を`/foo\C`で検索した場合のスクリーンショットです:

![](/images/2023-08-vim-backslash-c/example.png)

ちゃんと小文字🔡の`foo`だけにマッチしていますね！

# おまけ

バックスラッシュの後にShiftキーが必要な大文字のC、というのはちょっと入力しづらいので、ショートカットを設定しておくのがお勧めです。例えばこんな感じのキーバインドを設定しておけば、簡単に`\C`を用いた検索ができます。個人的な好みでプレフィックスを`<Space>`にしています。

```vim
" \C を付けた状態で検索開始
nnoremap <Space>/ /\C
nnoremap <Space>? ?\C

" 最後の検索に \C を付け加えた状態で検索し直す
"" 本来の n, N と挙動が一貫しないところがあるが、面倒なのでこのままで
nnoremap <Space>n /<Up>\C<CR>
nnoremap <Space>N ?<Up>\C<CR>
```
