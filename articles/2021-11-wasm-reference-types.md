---
title: "WebAssembly Reference Typesで、WasmでDOMを操作する壁がここまで下がった"
type: "tech"
emoji: "🤝"
topics: ["WebAssembly", "wasmbindgen"]
published: true
---

# きっかけ（となったtweetの訂正）

もう1ヶ月以上も経ってしまったが、こちらのtweetの公約どおり、WebAssembly (Wasm)におけるDOMの操作について知っている限りのことを書こう。

https://twitter.com/igrep/status/1443396752841666564

まずこの節の見出しのとおり、上記の発言は大きく間違えている。私はReference Typesがもたらすパフォーマンス的なメリットや、JavaScriptのオブジェクトを直接Wasmで渡すことが（一応）可能になったということを根拠に上記のtweetをした。しかし下記のtweetでも否定されているとおり、この観点は穴だらけなので、実際のところ多くの人が「直接操作できる」と実感できる状態ではないだろう。

https://twitter.com/kamenoko_dev/status/1443766820515639296

詳細は後述するとして、我ながらひどい凡ミスを犯してしまった。JavaScriptのことを十分に知っているはずなのに、情けない。謹んでお詫びし、ここで訂正する。

# 大前提: ある意味で永遠にそんな日は来ない

まず、「WasmでDOM直接操作する」というのが「WasmそのものにDOMを操作するAPIができる」という意味でないことは強調しておきたい。後述の通り「WasmでDOM直接操作する」のを達成するためにはいくつかの段階があり、ある日ブラウザーの新しいバージョンによって突然WebAssemblyがDOMのAPIを何もしなくても呼べる、なんてことは起き得ない。というのも、Wasmにはそもそも**組み込みで呼べる関数がない**のだ。Wasmはセキュリティーと可搬性を強く重んじる設計思想故に、ホスト（ウェブアプリケーションをはじめとする、Wasmモジュールを利用するアプリケーション）が明示的に許可した関数を通じてしか、Wasmの外の世界に影響を与えることができない。これはつまりWasmモジュール自身が保持しているメモリーやグローバル変数を書き換えることしかできないという意味であり、DOMを書き換えたりHTTPリクエストを送ったりするには、逐一必要な関数をホストがインポートさせなければならない。なので、Wasmモジュール自身が自由にDOMの要素を追加したり削除したりできるようになるということはあり得ない。それはもはやWasmの思想に反する別の何かだろう。

# Reference Typesとは？

冒頭のtweetで触れた「[Reference Types](https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md)」とは、JavaScriptの参照オブジェクトをそのままWasmの関数に渡したり、Wasmの関数から返したりできるようにする、という仕様だ。従来、Wasm製の関数は引数や戻り値として、整数や浮動小数点数のようなごく一部のプリミティブな型の値しか扱えなかった。これに、「参照型」として`externref`と`funcref`という型を加えたのが「Reference Types」だ。

具体例は以下の通り。

externrefを使用するWasmモジュールをWATで書いたもの example1.wat:

```wasm
(module
  (import "imports" "append"
    (func $append (param externref i32 i32)))

  (memory (export "memory") 1)

  ;; 線形メモリーにおける 0x42 の箇所に
  ;; 出力する文字列を配置
  (data (i32.const 0x42) "Hello, Reference Types!")

  ;; 引数として externref を受け取る関数
  (func (export "hello") (param externref)
    (call $append
      ;; どの要素に追記するかを externref の値で指定
      (local.get 0)
      ;; 次の二つの引数で、Wasmの線形メモリーに
      ;; 書かれた文字列（先頭のアドレスと長さ）を指定
      (i32.const 0x42)
      (i32.const 24))))
```

上記のWasmモジュールを使用するJavaScript example1.js:

```js
async function main() {
  const imports = {
    "imports": {
      append(domNode, address, length) {
        // 指定した文字列のアドレスと長さを使って Wasm モジュール
        // の線形メモリーから文字列をデコードし、テキストノードとして
        // domNode に追記する関数。詳細は省略。
      }
    }
  };

  // Wasm モジュールの読み込み、コンパイル
  const response = await fetch("./example1.wasm");
  const wasmBytes = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(
    wasmBytes,
    imports
  );

  // DOM への参照を直接 Wasm モジュールの関数に渡す
  const output = document.getElementById("output");
  instance.exports.hello(output);
}

// ... 以下略 ...
```

- ※本稿で作成した例はこれ以前もこれ以降も含めて、[こちらのリポジトリー](https://github.com/igrep/wasm-reference-types-examples)にて全体を公開する。ビルドした.wasmファイルなども敢えてリポジトリーに含めているので、`git clone`して適当なウェブサーバー[^web-server]で配信すれば試せるはず
- ※[Bytecode AllianceがReference Typesを紹介した記事](https://bytecodealliance.org/articles/reference-types-in-wasmtime)の例を元に、単純化した

[^web-server]: 例えば `python -m http.server`。Haskellerとしてはwai-app-staticパッケージのwarpコマンドがお手軽でおすすめ。

ご注目いただきたいのは、実質的な最後の行`instance.exports.hello(output);`だ。`document.getElementById`で取得したDOM要素への参照を、直接Wasmモジュールの関数に渡しているところにご注目いただきたい。これは件の`hello`関数がReference Typesを利用して、引数の型に`externref`を用いているためだ。このように、Wasmモジュールは`externref`型を使用することで、ホストにおける「参照型」を直接Wasmの関数で扱うことができる。ただし、Wasmの関数側でこの参照の中身をいじられるとセキュリティー上の問題が生じるので、`externref`型の値はWasmから見ると「不透明な値」としてしか使用することができず、importした他のホストの関数に渡すことしかできない。そのため、下記のようなコードは型エラーになる:

invalid.wat:

```wasm
(module
  (func (export "hello") (param externref)
  ;; externref を i43.add に渡すことはできない！
  (i32.add
    (local.get 0)
    (i32.const 0x42))))
```

エラーの例:

```
> wat2wasm invalid.wat
invalid.wat:4:4: error: type mismatch in i32.add, expected [i32, i32] but got [externref, i32]
  (i32.add
   ^^^^^^^
invalid.wat:4:4: error: type mismatch in function, expected [] but got [i32]
  (i32.add
   ^^^^^^^
```

それから、ここではDOMオブジェクトへの参照を例に挙げたが、実際のところ`externref`型の値はJavaScriptで取り扱う値であれば何でもよい。そしてもちろん、JavaScript以外の言語で書かれたホストアプリケーションであれば、そのプログラミング言語における任意の値を渡すことができる（もちろん、「参照型」と一口に言っても多様な型があるので、正しい型の値を渡さなければうまく動かないが）。

[WebAssemblyのRoadmap](https://webassembly.org/roadmap/)曰く、すでに標準化されていて、FirefoxやSafariで使える。[Chrome Platform Status](https://www.chromestatus.com/feature/5166497248837632)によるとChrome 96（2021年11月14日時点でのBeta版）であれば、デフォルトで使えるらしい。

# JavaScriptのコードはどこまで減る？

この`externref`型の参照は、前述のとおりWasm側では数値のように直接演算できず、結局のところ`import`したホストの関数に渡すしかないため、一見して大きな進歩には見えないかも知れない。しかし従来はそれすらかなわず、とても煩雑な方法を執らざるを得なかった。

Rustとwasm-bindgen、web-sysを利用した例で紹介しよう。次のような、ただNodeの`firstChild`を取得するだけの関数をWasmにコンパイルした後、wasm-bindgenコマンドでJavaScriptによるラッパーを生成すると、次のような関数が生成される:

Wasmにコンパイルする前のRustのソース src/lib.rs （抜粋）:

```rust
#[wasm_bindgen]
pub fn get_first_child(elem: web_sys::Node) -> web_sys::Node {
    elem.first_child().unwrap()
}
```

コンパイルしたWasmの関数をラップするJavaScriptの関数 pkg-no-reference-types/reference\_types\_examples.js （抜粋）:

```javascript
const heap = new Array(32).fill(undefined);

/* ... 省略 ... */

function dropObject(idx) {
    if (idx < 36) return;
    heap[idx] = heap_next;
    heap_next = idx;
}

function takeObject(idx) {
    const ret = getObject(idx);
    dropObject(idx);
    return ret;
}

function addHeapObject(obj) {
    if (heap_next === heap.length) heap.push(heap.length + 1);
    const idx = heap_next;
    heap_next = heap[idx];

    heap[idx] = obj;
    return idx;
}

/* ... 省略 ... */

export function get_first_child(elem) {
    var ret = wasm.get_first_child(addHeapObject(elem));
    return takeObject(ret);
}
```

このうち、実際にWasmモジュールの`get_first_child`関数を呼んでいるのがこの箇所:

```javascript
var ret = wasm.get_first_child(addHeapObject(elem));
return takeObject(ret);
```

純粋に`wasm.get_first_child`を呼ぶほかに、`addHeapObject`と`takeObject`という関数が引数と戻り値の間に挟まっている点にご注目いただきたい。詳細は割愛するがこれらは上記のコードにあるとおり、少々煩雑な処理をしている。`externref`を使わないWasmでは、このような変換処理が都度必要なのだ。

一方、wasm-bindgenコマンドを実行する際に`--reference-type`というオプションを渡してReference Typesを使用するよう設定すると、前述の`addHeapObject`などはきれいさっぱりなくなり、直接`wasm.get_first_child`を呼んでくれるようになる[^wasm-pack]:

コンパイルしたWasmの関数をラップするJavaScriptの関数 pkg-with-reference-types/reference\_types\_examples.js （抜粋）:

```javascript
export function get_first_child(elem) {
    var ret = wasm.get_first_child(elem);
    return ret;
}
```

[^wasm-pack]: 残念ながらwasm-packではまだサポートされていない。 [Pull requestは送られている](https://github.com/rustwasm/wasm-pack/pull/888)ものの、依然としてマージされていない。

ここでは省略したボイラープレートなコードがまだ残るものの、不要なJavaScriptの関数をいくつも減らすことができた。

# まだ足りないのは？

私が冒頭のツイートは、ここまで説明した仕様を指して「WasmでDOMを操作するのはreference typesを使えば最早JS経由とすら言えない」などとのたまわっていた。しかしながら、これでは不十分なことは賢明な読者ならばすぐに気づくだろう。確かに関数の引数や戻り値はシームレスにやりとりできるようになったものの、次のような問題がある。

**JavaScriptは0じゃない**

上記の例では省略したが、依然としてJavaScriptからWasmの関数を呼ぶためには、事前にJavaScriptのAPIである、`WebAssembly.instantiateStreaming`関数などを利用してWasmのコンパイルをしなければならない。Wasmがエクスポートする関数そのものは直接利用できるが、その前の準備はJavaScriptが必要なままだ。

**関数の引数として渡す、戻り値として返す、以外の操作ができない**

DOMオブジェクトを含むJavaScriptの参照型の値を扱う上で重要な操作といえば、プロパティへのアクセスだろう。Reference Typesだけでは、このような、関数の引数として渡す、戻り値として返す、以外の操作を賄うことができない。

これらの問題を解決するためには、次のような機能が必要であろう:

- JavaScriptを一切介さず、（`<script>`タグのように）HTMLのみでWasmモジュールをimportする機能
    - Wasmにimportさせる関数などをどうやって決めるか、といった点が問題になりそう。[WebAssembly/ES Module Integration](https://github.com/WebAssembly/esm-integration/tree/main/proposals/esm-integration)が解決のヒントになるか？
- JavaScriptのオブジェクトを操作する機能
    - [WebAssembly Interface Types](https://github.com/WebAssembly/interface-types/blob/main/proposals/interface-types/Explainer.md)を実現できれば、オブジェクトをRecord型として扱い、個々のプロパティーを自動で分解して関数の引数として渡せるようになると思われる
        - ただし、プロパティーの書き換えはできないので、これでもまだ完全にJavaScriptをゼロにはできない模様

# パフォーマンスは改善される？

ここまでで、Reference Typesによって、確かにWasmでDOMを操作するのに必要な、JavaScriptのコードが少し減ったことがわかった。次に気になるのは、JavaScriptのコードが減ったことによるパフォーマンスの改善だろう。私にとってこの記事を書く重大なモチベーションの一つでもある。[Mozillaによるこの辺りの記事](https://hacks.mozilla.org/2018/10/calls-between-javascript-and-webassembly-are-finally-fast-%F0%9F%8E%89/)を読むと、JavaScriptからWasmの関数を呼ぶ際のコストは充分に下げられたらしい。であれば、Reference Typesを利用したDOMに対する操作も充分に速くなっていてほしいものだが、実際にはどうだろうか。

そこで、あまり実践的なコードではないものの、以下のRust製コードを`rustc 1.55.0`でWasmにコンパイルした上で、それぞれを100万回ずつ実行する簡単なベンチマークをとってみた:

src/lib.rs （抜粋）:

```rust
#[wasm_bindgen]
pub fn append_and_remove(elem: web_sys::Element) {
    let doc = web_sys::window().unwrap().document().unwrap();
    let child = &doc.create_element("br").unwrap();
    elem.append_with_node_1(child).unwrap();
    let _ = elem.remove_child(child).unwrap();
}
```

ご覧のとおり、web-sysクレートを使って`br`要素を作成し、指定した要素に追加して直ぐに削除する、ただそれだけのコードだ。これを次のコードで100万回実行して、簡単に計測した:

common.js （抜粋）:

```js
const target = document.querySelector('#app .target');

// ... 中略 ...

console.time(label);
for (let i = 0; i < 1_000_000; ++i) {
  append_and_remove(target);
}
console.timeEnd(label);
```

結果は以下の通り:

| ブラウザー | label                | 実行時間 (ms) | 対NO reference types比 (%) |
| ---------- | -------------------- | -------------:| --------------------------:|
| Firefox    | NO reference types   |          2167 |                     100.00 |
| （同）     | With reference types |          2687 |                     124.00 |
| （同）     | Only JavaScript      |           637 |                      29.40 |
| Chrome     | NO reference types   |          3432 |                     100.00 |
| （同）     | With reference types |          4129 |                     120.31 |
| （同）     | Only JavaScript      |          1039 |                      30.27 |
| Edge       | NO reference types   |          3416 |                     100.00 |
| （同）     | With reference types |          3858 |                     112.94 |
| （同）     | Only JavaScript      |          1187 |                      34.75 |

使用したブラウザーの詳細:

- Firefox: Firefox 94.0.1
- Chrome: Google Chrome 95.0.4638.69
- Edge: Microsoft Edge 95.0.1020.44
    - Edgeに関しては、なぜか最初の実行が2回目以降の実行と比べて10倍以上遅く、3回目以降の実行でようやく安定した結果が出てきたので、数回繰り返して最後に記録された値を記録した。この辺りは固有の事情がありそうだ。もしかして初回アクセスの時だけ[Super Duper Secure Mode](https://microsoftedge.github.io/edgevr/posts/Super-Duper-Secure-Mode/)が発動してる？

※いずれも64bit版で、Windows 10で実行。Chrome・Edgeでの結果は、1ms未満四捨五入

ご覧のとおり、Reference Typesを使った場合 (With Reference Types)の実行時間は、Reference Typesを使わず、wasm-bindgenによる従来のラッパーを使用した場合 (NO reference types) よりも20%前後長くなってしまった。そしてWasmを全く使わず、JavaScriptのみで同等の処理を書いた場合の時間 (Only JavaScript) よりもさらに遅い。JavaScriptのみのバージョンが一番速いのはいいとして、なぜJavaScriptによるラッパーが薄いにも関わらず、Reference Typesを使ったバージョンの方がより遅くなってしまったのだろうか？一体どんなオーバーヘッドがReference Typesによって生まれてしまったのだろうか。残念ながら私はブラウザーの実装には詳しくないため、見当がつかなかった。英語だが、下記のとおりStack Overflowで質問したのでもしご存知の方がいれば回答願いたい:

[rust - Why is passing DOM objects as `exnternref`s *slower* than passing through the JS value table? - Stack Overflow](https://stackoverflow.com/questions/69943544/why-is-passing-dom-objects-as-exnternrefs-slower-than-passing-through-the-js)

# 結論

以上のとおり、WebAssembly Reference Typesは、DOMをはじめJavaScriptのオブジェクトをWasmの関数で直接やりとりできるようになる便利な機能だ。しかしながら、「WasmからDOMを直接触れる」と自信を持って言うには依然として機能的にもパフォーマンス的にも不足している。[Wasm公式のFAQ](https://webassembly.org/docs/faq/#is-webassembly-trying-to-replace-javascript)のとおり、Wasmは完全にJavaScriptを置き換える為のものではないとされているものの、多様なプログラミング言語でブラウザーベースのアプリケーションが少しでも書きやすくなる未来は今後も期待したい。

# 参考文献（本文中で言及のないもののみ）

- [WebAssembly Specification Release 1.1 + bulk instructions + reference types (Draft, Mar 11, 2021)](https://webassembly.github.io/reference-types/core/index.html)
- [Command Line Interface - The `wasm-bindgen` Guide](https://rustwasm.github.io/docs/wasm-bindgen/reference/cli.html)
- [かめのこにょこにょこさんのこちらのtweetを含むスレッド](https://twitter.com/kamenoko_dev/status/1443770229893468163)
