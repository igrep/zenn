---
title: "配列からべき集合を求める割と綺麗な関数とそれに至るまで"
type: "tech"
emoji: "🏋️"
topics: ["Ruby", "アルゴリズム"]
published: true
---

# 💁‍♂本記事で掲載するコードについて

今回紹介するコードの他、RubyやHaskellによる、べき集合を求める関数の実装をいくつか含めたリポジトリーを[igrep / powersets](https://github.com/igrep/powersets)に公開したので興味があればご覧ください。使用した各種言語処理系のバージョンもそちらに記載しています。興が乗ったので、[すべての実装についてHaskell製の最も素朴なバージョンと同じ振る舞いをするか、QuickCheckでテスト](https://github.com/igrep/powersets/blob/87984c232f410811f0a14a0dd89f1323aa0ee9fb/test/Spec.hs#L24)までしています。

# きっかけ

半ば風が吹けば桶屋が儲かる的な話なのですが、2年半程前、Yokohama.rbというRubyのコミュニティーでこちら👇の問題を解こうとしたのがきっかけでした:

[マスクしても同じ数 Yokohama.rb #103](http://nabetani.sakura.ne.jp/yokohamarb/103mask/)

詳細は割愛しますが、この問題の解法の一つとして、入力された数における有効なビット番号の配列 --- 例えば2進数で`10100`となる数であれば下位ビットから数えて3ビット目と5ビット目が`1`なので`[3, 5]` --- の、べき集合を求めることがキーとなるものがあります。それに気づいた私は、べき集合を求める方法をあれこれ調べました。

最初に見つけた方法 --- といっても、ググって[Stack Overflow](https://stackoverflow.com/a/39539472/4299824)からRubyに翻訳したのとあまり変わりありませんが --- は、次のような素朴な再帰によるものでした:

```ruby
def powerset_rec xs
  return [[]] if xs.empty?
  x = xs.shift
  ps = powerset_rec(xs)
  ps.map {|p| [x] + p } + ps
end
```

しかし、上記のような関数を用いて件の問題を解こうとすると、大きな数（この場合有効なビット番号が多い数）を与えたところでスタックが溢れてエラーになってしまいました。問題の要件上、少しやり方を変えればこの関数を使って解くこともできたのですが、私はこの時**再帰呼び出しを使わないでべき集合を求める**方法が気になって気になって仕方がなくなってしまい、それ以降元の問題そっちのけでべき集合を求める方法を考えていました。

# 結論

そうして試行錯誤してたどり着いた、再帰呼び出しをしない、最も簡潔な（かつ恐らく効率もよい）、べき集合を求める関数はこちらです:

```ruby
def powerset_reduce xs
  xs.reduce([[]]) do |last_result, x|
    last_result.map {|p| [x] + p } + last_result
  end
end
```

そう、おなじみ`reduce`を使いました！中で使用している配列に対する`map`や`+`も含めて、似た機能の関数はRuby以外の多くの言語にありますし、よくある`for`ループに変換するのも比較的簡単なはずです。つまり、他のプログラミング言語に移植するのも、再帰でのバージョンと同じくらい簡単なところがこの方法のメリットです。

以下が上記二つの実行例です:

```ruby
irb(main):002:0> powerset_rec [9, 0, 8]
=> [[9, 0, 8], [9, 0], [9, 8], [9], [0, 8], [0], [8], []]

irb(main):003:0> powerset_reduce [9, 0, 8]
=> [[8, 0, 9], [8, 0], [8, 9], [8], [0, 9], [0], [9], []]
```

結果となる配列の順番が冒頭の`powerset_rec`と異なりますが、べき**集合**なので本稿では特に考慮しないことにします。ちゃんと列挙できているみたいですね！

日本語で軽く探した限り、少なくともRubyでこれに相当する実装を紹介した記事は見つかりませんでした。唯一エッセンスが似ていたのが、[こちらの記事](https://qiita.com/satzz/items/c1b1a5dced5ac8170a85)にあったPerlによる実装です:

```perl
my @a = (1,2,3);

my @p = ([]);
for my $a(@a) {
    @p = (@p, map {[@$_, $a]} @p);
}
```

`for`ループを`reduce`に置き換えれば、そのまま先ほどの`powerset_reduce`に翻訳できそうですね！

# たどり着くまで

べき集合の計算を、各要素が「存在する場合」と「存在しない場合」で分岐する2分木をたどる計算と解釈して走査してみたり、自前でスタックを作ってアセンブリーで関数呼び出しする要領でできないか、と考えたり、いろいろ紆余曲折ありましたが、最終的な答えに至ったきっかけは下記のように、元々の再帰呼び出しを利用したバージョンにおける、引数や途中の再帰呼び出しの結果をprintデバッグで確認したときでした:

```ruby
def powerset_rec xs
  return [[]] if xs.empty?
  x = xs.shift
  p x
  ps = powerset_rec(xs)
  p ps
  ps.map {|p| [x] + p } + ps
end
```

その実行例は次のとおり:

```haskell
irb(main):002:0> powerset_rec [3, 7, 0]
3
7
0
[[]]
[[0], []]
[[7, 0], [7], [0], []]
=> [[3, 7, 0], [3, 7], [3, 0], [3], [7, 0], [7], [0], []]
```

デバッグコードによって出力された値をご覧ください。最初の3行で引数の配列`[3, 7, 0]`にある各要素を出力した後、後半の3行でそれぞれの再帰呼び出しにおける結果に対して、配列の各要素を追加したりしなかったりしている、という動きが読み取れるでしょうか？

```ruby
def powerset_rec xs
  return [[]] if xs.empty?
  x = xs.shift
  p x                    # この行までで各要素を一つずつ処理して
  ps = powerset_rec(xs)  # この行以降で再帰呼び出しの結果を処理する
  p ps
  ps.map {|p| [x] + p } + ps
end
```

つまり、`powerset_rec`の処理は、配列が空になるまで要素を一つずつ取り出す段階と、取り出した要素と直前の再帰呼び出しの結果を結合する段階とで明確に二分されており、再帰呼び出しでないと書きづらいと言うほど複雑なことをしているわけではない、と読み取れます。よくある木構造に対する再帰処理のように、要素を一つ処理する毎に何か処理をしてはまた再帰呼び出しする、という動作ではないんですね。

ということは、下記のありふれたループでやるように、最後の要素を処理したときの結果を保存する変数`last_result`を用意しつつ、要素を一つずつ処理して`last_result`を更新していく... という定石が通用しそうです:

```ruby
last_result = 初期値

配列.each do |x|
  last_result = last_resultとxを使って何か計算
end
```

--- そしてそう、それは`reduce`メソッドで容易に書けるパターンでもあります:

```ruby
配列.reduce(初期値) do |last_result, x|
  last_resultとxを使って何か計算
end
```

`powerset_reduce`において`初期値`や`last_result`がどのような値になるかも、`powerset_rec`の動作を参考に導くことができます。`powerset_rec`は、前述のとおり配列が空になるまで要素を一つずつ取り出した後、再帰呼び出しします。このとき渡される引数`xs`は、前の行で`xs.shift`した後の`xs`、つまり先頭の要素を一つ取り除いた状態の`xs`です。このことから、`powerset_rec`が再帰呼び出しするのは、現在の要素`x`を取り除いた分だけ小さくなった配列に対する`powerset_rec`の結果であると分かります。そうして`x`を繰り返し抜き出した結果、最終的に`powerset_rec`に渡る引数`xs`は空配列になります。そこまで達すると、`powerset_rec`は`return [[]] if xs.empty?`の行で直ちに`[[]]`を返します。[べき集合の定義](https://ja.wikipedia.org/w/index.php?title=%E5%86%AA%E9%9B%86%E5%90%88&oldid=84166946#%E5%AE%9A%E7%BE%A9)のとおり、空の集合のべき集合は、空の集合一つだけの集合ですしね。

一方`powerset_reduce`では、これと逆の順番で処理すればいいのです。`reduce`メソッドの仕様上、`初期値`は引数として空の配列を受け取った時の戻り値と同じ値になるのですから`[[]]`を使います:

```ruby
配列.reduce([[]]) do |last_result, x|
  last_resultとxを使って何か計算
end
```

`last_resultとxを使って何か計算`については、`powerset_rec`が「先頭の要素を一つ取り除いた配列に対する`powerset_rec`の結果」、`ps`に対して繰り返し`ps.map {|p| [x] + p } + ps`して最後に`[[]]`を足し合わせるのですから、`powerset_reduce`では初期値`[[]]`から積み上げるようにして、前の結果`last_result`で`ps.map {|p| [x] + p } + ps`相当のことをすれば良いのです:

```ruby
配列.reduce([[]]) do |last_result, x|
  last_result.map {|p| [x] + p } + last_result
end
```

これであとは`配列`を`powerset_reduce`の引数`xs`に変えれば、`powerset_reduce`のできあがりです！

# 参考（本文中で言及していないもののみ）

- [Rubyで与えられた配列の部分集合を列挙（＋α） - Qiita](https://qiita.com/HMMNRST/items/b5ae96d3d3ad75d7c893)
- [二分木で再帰を使わない問題 - プログラマ専用SNS ミクプラ](https://dixq.net/forum/viewtopic.php?t=14893)
- [再帰呼び出しをループに置き換える - 空想犬猫記](https://xoinu.hatenablog.com/entry/2015/07/25/112906)
- [特定ディレクトリ以下の列挙 (再帰呼び出しとループ) - ぐるぐる～](https://bleis-tift.hatenablog.com/entry/20090616/1245134308)
