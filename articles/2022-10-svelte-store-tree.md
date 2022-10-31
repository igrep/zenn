---
title: "なぜSvelte風Solid.jsのstoreは作れないか、およびsvelte-store-treeの新バージョンの紹介"
type: "tech"
emoji: "🚉"
topics: ["TypeScript", "Svelte", "状態管理"]
published: true
---

先ほど、[前回](https://zenn.dev/igrep/articles/2022-09-svelte-store-tree)紹介した、Svelte用の状態管理ライブラリー、svelte-store-treeの新しいバージョン（v0.3.1）をリリースしました:

[svelte-store-tree - npm](https://www.npmjs.com/package/svelte-store-tree)

このバージョンでは、APIを[Solid.jsのstoreという機能](https://www.solidjs.com/tutorial/stores_createstore)を参考に、ガラッと変更してより使いやすくしました。そこでこの記事では、新しくなったAPIを紹介するとともに、類似のライブラリーとして参考にした、Solid.jsのstoreと比較したいと思います。それを通じて、Svelteのstoreについて一つ問題点を紹介しますので、今後の仕様を検討する上で参考にしていただければ幸いです（この後この記事の英語版を作ります）。

# あらまし

- svelte-store-treeはv0.3.1以降、`WritableTree`と`ReadableTree`自身のメソッドを単純化して、代わりに`Accessor`を作成したり合成したりしやすくした
- 類似のライブラリーであるSolid.jsのstoreを参考にしたが、Solid.jsのstoreと異なり、svelte-store-treeは`choose`というAPIを使ってstoreの中の値を選択しなければ、union型をうまく扱えない。それは、Solid.jsのstoreと異なり、Svelteのstoreが「storeの中の値を読むAPI」と「storeの値を更新するAPI」を別々のオブジェクトに属させていることに起因している。

# 使い方の例

まずは前回と同様のサンプルに沿って、新しい使い方を紹介します。以降では、[こちらで動かせるサンプルアプリ](https://codesandbox.io/p/github/igrep/svelte-store-tree/draft/floral-sound)における、以下のような型を扱います:

```typescript
export type Tree = string | undefined | KeyValue | Tree[];

export type KeyValue = {
  key: string;
  value: string;
};
```

いろいろな値を取り得る、再帰的な構造になっていますね。

なお、[このサンプルアプリのコードはGitHubにも](https://github.com/igrep/svelte-store-tree/tree/main/example)置いているので、今後具体的な箇所を例示する際は執筆時点のバージョンにおける街頭の行へのリンクを張ります。

## `WritableTree`オブジェクトの作成

（ここは前のバージョンから変わってません）

svelte-store-treeが提供する`WritableTree`オブジェクトを作るには、その名の通り`writableTree`という関数を使います:

```typescript
export const tree = writableTree<Tree>([
  "foo",
  "bar",
  ["baz1", "baz2", "baz3"],
  undefined,
  { key: "some key", value: "some value" },
]);
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/30740be58afe4413f0d4e2d454bfe2b340a929e4/example/tree.ts#L11-L17)

本家の`writable`関数と同様に、storeの初期値を引数で指定します。`Tree`型はunion型なので、型推論を助けるために型パラメーターを明示しておきましょう。

## 普通に`subscribe`したり`set`したりする

（ここも前のバージョンから変わってません）

svelte-store-treeは本家Svelteのstoreのように[Store contract](https://svelte.jp/docs#component-format-script-4-prefix-stores-with-$-to-access-their-values-store-contract)を守っているので、おなじみのドルマーク `$` で自動で`subscribe`したり、two-way binding機能を通じて`set`することができます:

```html
<script lang="ts">
  export let tree: WritableTree<Tree>;
  ...
</script>
...
{#if typeof $tree === "string"}
  <li>
    <NodeTypeSelector label="Switch" bind:selected {onSelected} /><input
      type="text"
      bind:value={$tree}
    />
  </li>
...
{/if}
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/30740be58afe4413f0d4e2d454bfe2b340a929e4/example/Tree.svelte#L36-L42)

## `zoom`して、部分木に対する`WritableTree`を作る

svelte-store-treeは、部分木に対するstoreを作るためのAPIとして、以下の`zoom`で始まる名前のメソッドを提供します:

- `zoom<C>(accessor: Accessor<P, C>): WritableTree<C>`
    - `Accessor`という、親`P`から子`C`を取得したり、親`P`における子`C`の値を書き換える関数を備えたオブジェクトを渡すことで、新しい`WritableTree`型のstore treeを作る
- `zoomNoSet<C>(readChild: (parent: P) => C | Refuse): ReadableTree<C>`
    - 親`P`から子`C`を取得する関数`readChild`を渡すことで、新しい`ReadableTree`型のstore treeを作る
        - `Refuse`型については後述
    - `ReadableTree`は、`set`できなくなっているstore tree（子を`zoom`メソッドで作成した場合、その子については`set`できるので、完全にread-onlyというわけではない）

以前はオブジェクトのプロパティーに対するstore treeを作る、`zoomInWritable`などのメソッドもありましたが、新しいバージョンでは`Accessor`を簡単に作るためのAPIを複数用意することで、`zoom`と`zoomNoSet`だけ使えるようにしました。ちょっとシンプルになりましたね！

サンプルアプリでは、次のように`into`という`Accessor`を作る関数を使うことで、`KeyValue`型の値が持つ`key`と`value`を書き換えるための`WritableTree`を作っています:

```typescript
const key = tree.zoom(into("key"));
const value = tree.zoom(into("value"));
```

... が、こちらは正しくありません。svelte-checkで調べてみると、TypeScriptが次のような型エラーを出します:

```typescript
/svelte-store-tree/example/Tree.svelte:14:35
Error: Argument of type 'string' is not assignable to parameter of type 'never'. (ts)
...


/svelte-store-tree/example/Tree.svelte:15:37
Error: Argument of type 'string' is not assignable to parameter of type 'never'. (ts)
...
```

`parameter of type 'never'`とあるとおり、引数の型が`never`になっていて分かりづらいですが、これは`tree`の型が`WritableTree<Tree>`、つまり`Tree`型を含む`WritableTree`となっていて、その`Tree`型は先ほどの定義通り`undefined`も取り得るunion型なので、`keyof`の結果が`never`になってしまうためです。

これを修正するには、`WritableTree<Tree>`をなんとかして`WritableTree<KeyValue>`に変換しないといけません。それを実現するのが、次の節で紹介する`choose`関数で作る`Accessor`です。

## `choose`して、storeの値が特定の場合のみ値を取得する

`choose`関数が作った`Accessor`を`zoom`に渡すことで、「storeの値が特定の条件にマッチする場合だけ、`subscribe`している関数を呼ぶ」store treeを作ることができます。

サンプルアプリから、`choose`を使っている箇所を抜き出しましょう:

```typescript
...
const keyValue = tree.zoom(choose(chooseKeyValue));
...
```

`choose`は、引数として、「storeの中の値（上記の場合`Tree`型の値）を受け取って、別の値、あるいは`Refuse`を返す関数」を受け取ります。上の例における`chooseKeyValue`ですね。実装は次の通りです:

```typescript
export function chooseKeyValue(tree: Tree): KeyValue | Refuse {
  if (tree === undefined || typeof tree === "string" || tree instanceof Array) {
    return Refuse;
  }
  return tree;
}
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/30740be58afe4413f0d4e2d454bfe2b340a929e4/example/tree.ts#L62-L67)

`choose`が返した`WritableTree`は、引数として受け取った関数が、`Refuse`という専用のunique symbol[^refuse]を返した場合は`subscribe`している関数を呼ばず、それ以外の値を返した場合のみその値で`subscribe`している関数を呼びます。

[^refuse]: 「zoom」、「choose」、「refuse」の3つで韻を踏んでいます。

なぜ`boolean`を返す関数にしないのか、あるいは`undefined`を使わず`Refuse`という専用の値を用意したのか、という疑問が思い浮かぶ方もいらっしゃるかも知れません。

まず`boolean`を返す関数にしないのは、TypeScriptによる型の絞り込みで`WritableTree`の中の型を変換するため、変換結果を関数の型で明示しないといけないからです。ユーザー定義の型ガード（戻り値の型が`Tree is KeyValue`のようになっている関数）は、`choose`のような高階関数には使えませんでした。

それから、`Refuse`という専用の値を用意したのは、`{ property: T | undefined }`のような、`undefined`を含むプロパティーも柔軟に扱えるようにするためです。

話を戻しましょう。先ほど`tree.zoom(into("key"))`などと書いて型エラーになっていた箇所は、次のように`choose`と組み合わせることで修正できます:

```typescript
const keyValue = tree.zoom(choose(chooseKeyValue));
const key = keyValue.zoom(into("key"));
const value = keyValue.zoom(into("value"));
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/30740be58afe4413f0d4e2d454bfe2b340a929e4/example/Tree.svelte#L13-L15)

あとは`zoom`が返した`key`・`value`を使うことで、`KeyValue`型の`key`と`value`だけをtwo-way bindingすることができます:

```html
<dl>
  <dt><input type="text" bind:value={$key} /></dt>
  <dd><input type="text" bind:value={$value} /></dd>
</dl>
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/30740be58afe4413f0d4e2d454bfe2b340a929e4/example/Tree.svelte#L68-L71)

## 子による更新に親が応じる

（ここは前のバージョンから変わってません）

これまでの例は、実際のところあまりstore treeの良さを生かせていません。わざわざ一つのデータ構造が入った一つのstoreとして管理しなくても、個別のコンポーネントの状態としてそれぞれの値を管理すれば事足りるからです。

svelte-store-treeはもちろんそれより一歩踏み込んでいます。それは、**子における更新を、直接の親だけに伝播できる**、という点です。

例えば、先ほどの`key`と`value`をtwo-way bindingした二つの`<input>`のうち、`key`の`<input>`が更新された場合をイメージしましょう:

```html
<dl>
  <dt><input type="text" bind:value={$key} /></dt>
  <dd><input type="text" bind:value={$value} /></dd>
</dl>
```

この場合、`key`に`set`した値は、`key`自身を`subscribe`している関数と、その親である`keyValue`をはじめとする、直接の先祖を`subscribe`している関数に伝わります。兄弟に当たる`value`を`subscribe`している関数や、親の兄弟などを`subscribe`している関数には伝わりません。

図で表すと、例えば木が次のような形になっていたとして...

![List 1を根としてList 2, string, KeyValueと言う子を含む木](/images/2022-09-svelte-store-tree/example0.png "子を更新する木の例")

「key」を更新したとき、更新が伝播する、すなわち`subscribe`している関数を呼び出すのは「List 1」・「KeyValue」・「key」の3つのstoreです:

![List 1を根としてList 2, string, KeyValueと言う子を含む木を更新したとき](/images/2022-09-svelte-store-tree/example1.png "keyを更新したとき、どこに更新が伝播するか")

「value」、「string」や「List 2」、「List 2」の子にあたるstoreには更新が伝わりません。そうすることで、余分な再レンダリングを避けることができます。

以上の特徴を活かすためにサンプルアプリでは、木のルートにおいて、木に含まれるノードの数を種類毎に集計することにしました:

```html
<script lang="ts">
  import TreeComponent from "./Tree.svelte";
  import { tree, type NodeType, type Tree } from "./tree";

  $: stats = countByNode($tree);
  function countByNode(node: Tree): Map<NodeType, number> {
    ...
  }
</script>

<table>
  <tr>
    <th>Node Type</th>
    <th>Count</th>
  </tr>
    {#each Array.from(stats) as [nodeType, count]}
  <tr>
      <td>{nodeType}</td>
      <td>{count}</td>
  </tr>
    {/each}
</table>
...
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/30740be58afe4413f0d4e2d454bfe2b340a929e4/example/Root.svelte)

# 変更点まとめ

まとめるとsvelte-store-treeのv0.3.1では、次の変更を加えました:

- `WritableTree`と`ReadableTree`のメソッドを`zoom`と`zoomNoSet`のみに絞り、代わりに`Accessor`を組み立てるAPIを充実させました
    - `Accessor`をクラスにして、`and`というメソッドで合成できるようにしました（後述します）
- `ReadableTree`の仕様を変えて、`ReadableTree`から`zoom`したstore treeは、`WritableTree`を返すようにしました
    - `ReadableTree`を完全に`set`できないようにすると、`derived`で実現できるものと変わらない程度の機能になることに気づいたので、このように変更しました

# 対Solid.jsのstore

冒頭で触れたとおり、svelte-store-treeのv0.3.1は、目的がよく似ている[Solid.jsのstoreという機能](https://www.solidjs.com/tutorial/stores_createstore)を参考にしています。なのでここからは、svelte-store-treeがどのような点でSolid.jsのstoreに近づいたか、また、逆に、Svelteの仕様上Solid.jsのstoreと同等に提供するのが難しい、と気づいた部分も共有することで、今後Svelteの仕様を考える上で参考になる情報が提供できれば幸いです。

## Solid.jsのstoreを簡単に紹介

Solid.jsのstoreは、先ほどリンクを張ったページで「ネストしたリアクティビティに対する Solid の答えです」と謳われているとおり、入れ子の構造を部分的に更新したり、部分的にトラックしたりする仕組みです。下記のように`createStore`関数を実行すると、（Reactの`useState`などとうっすら似たように）storeの現在の値と、値を更新するための関数を含むペアを返します:

ℹ️この節の例はすべて[公式サイトにおける例](https://www.solidjs.com/docs/latest/api#%E3%82%B9%E3%83%88%E3%82%A2%E3%81%AE%E6%9B%B4%E6%96%B0)から引用しました。

```javascript
const [state, setState] = createStore({
  todos: [
    { task: 'Finish work', completed: false },
    { task: 'Go grocery shopping', completed: false },
    { task: 'Make dinner', completed: false },
  ]
});
```

ペアの一つ目に含まれる値（上記の例で言うところの`store`）は、[Proxy](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Proxy)でラップされています。そうすることで、`store`の値や、その一部のプロパティーに対するアクセスを追跡してくれます。

それから、ペアの二つ目に含まれる関数（上記の例で言うところの`setStore`）は、次のように入れ子構造の各段階におけるプロパティーの名前や、（該当する値が配列であれば）、どの要素を更新するか指定する関数などを、「パス」として各引数で指定できます:

```javascript
// `todos` プロパティーにおける0番目と2番目の要素の`completed`プロパティーを`true`に
setState('todos', [0, 2], 'completed', true);

// `todos`プロパティーにおける要素のうち、
// `completed`が`true`の要素の`task`プロパティーの末尾に`'!'`をつける
setState('todos', todo => todo.completed, 'task', t => t + '!');
```

`setState`関数一つだけで入れ子構造を何段階でも深掘りして更新できる、というパワフルな機能を備えています。

また、上記の通りSolid.jsのstoreは、`createStore`関数が返す`Proxy`に包まれた値と、更新するための関数を分けて扱うように設計されています。Svelteのstoreは、two-way bindingとの兼ね合いもあり、基本的に`set`と`subscribe`を実装したオブジェクトを一つのオブジェクトとして取り回すように作られていますので、その点が大きく異なりますよね。

## どこを参考にしたのか

そんなSolid.jsのstore機能から（少し）影響を受け、svelte-store-treeは`Accessor`、すなわち入れ子構造にどうアクセスするか指定するためのAPIを改善しました。具体的には、Solid.jsのstoreでは、前の節で言ったところの`setStore`関数の引数において、プロパティー名や要素を選択する関数を複数指定することで、それらを組み合わせることができると紹介しました。一方、svelte-store-treeでは、`Accessor`クラスの`and`というメソッドを呼ぶことで、入れ子構造へのアクセス方法を組み合わせることができます。

[^array]: ⚠️Solid.jsのstoreにおいて、storeの値を関数でフィルタリングしたりする機能は、storeの値が配列である場合のみ使えます。svelte-store-treeにおける`choose`のように、型の絞り込みに使うのとは趣旨が異なるのでご注意ください。

例えば「`store`の値における`foo`というプロパティーの値が、`undefined`でない場合のみ`subscribe`した関数を呼ぶ」storeは、次のように`into`と`isPresent`という`Accessor`を組み合わせることで作ることができます:

```typescript
store.zoom(into('foo').and(isPresent()));
```

これを使えば、当記事前半で紹介したサンプルコードにおける、`tree`からプロパティー`key`を取り出す処理は、次のようにも書き換えることができます:

```typescript
const key = tree.zoom(choose(chooseKeyValue).and(into("key")));
```

なぜ、Solid.jsのstoreを見習って`zoom`メソッドの引数に`Accessor`を複数渡せる形式にしなかったのかというと、`zoom`の実装をシンプルにするのと、一度に何段階も`Accessor`を`and`で積み重ねて入れ子構造の深い位置にアクセスするのは、あまり望ましい使用方法でないだろうと考えたためです。

まず前者について詳述すると、Solid.jsの場合と異なり`zoom`メソッドは引数を一つしか受け取らないので、`zoom`メソッドは複数の`Accessor`を組み合わせる必要がなくなり、その分処理が簡潔になります。`Accessor`を組み合わせるのはあくまで`and`メソッドの仕事なのです。

そして後者は、使用した場合における設計の問題です。そもそも入れ子構造に対して一度に何段階も深く潜る処理を書くのは、内部構造を切り開くようなものであり、変更に弱くなるリスクを孕んでいます。またそもそも、svelte-store-treeを開発した当初私が想定していたような、再帰的な構造は入れ子の深さが動的に変わるので、関数の引数を複数列挙することで組み合わせる方式ではうまく合成できません。

複数の引数として渡す代わりに`and`メソッドで組み合わせるのは少々冗長な書き方となってしまいますが、私が期待した利用頻度と使い勝手に基づき、このような設計にしました。

## どこを参考にできなかったのか

ある意味、「Solid.jsのstoreのSvelte向けバージョン」とも言えるsvelte-store-treeですが、どうしても取り入れられなかった特徴があります。それは、`choose`関数による`Accessor`が必要な点です。利用方法の紹介で1セクション割いて説明した`choose`... ですが、Solid.jsのstoreに、これに相当するものがありません[^array]。不要だからです。

[^array]: ⚠️Solid.jsのstoreにおいて、storeの値を関数でフィルタリングしたりする機能は、一見`choose`と似ているように見えますが、storeの値が配列である場合のみ使えます。`choose`のように型の絞り込みに使うのとは趣旨が異なるのでご注意ください。

なぜSolid.jsのstoreでは`choose`のような機能が要らないのでしょうか？それは、`createStore`関数が返す、storeの値を読み出すためのオブジェクト（`Proxy`で包まれたstoreの現在の値）と、更新するための関数を、別々に扱うよう設計されているところにあります。

`choose`が担うような、storeの値が特定の条件にマッチするよう選ぶ処理は、Solid.jsのstoreでは、単にstoreの値のコンポーネントの中で[`Show`](https://www.solidjs.com/tutorial/flow_show)や[`Match`](https://www.solidjs.com/tutorial/flow_switch)（Solid.jsのコンポーネントにおける`if`文に相当するもの）を使って分岐すれば良いのです。

これは同じことがsvelte-store-treeにも当てはまりそうにも聞こえますが、うまくいきません。`choose`を最初に紹介したとき用いた例を思い出してください:

```html
<script lang="ts">
  // ...

  const keyValue = tree.zoom(choose(chooseKeyValue));
  const key = keyValue.zoom(into("key"));
  const value = keyValue.zoom(into("value"));

  // ...
</script>

{#if typeof $tree === "string"}
  ...
{:else if $tree === undefined}
  ...
{:else if $tree instanceof Array}
  ...
{:else}
  ...
  <dt><input type="text" bind:value={$key} /></dt>
  <dd><input type="text" bind:value={$value} /></dd>
  ...
{/if}
```

上記のコード例における`key`と`value`という二つのstoreは、`{#if} ... {/if}`で選択しているとおり、`tree`の値が`string`でも`undefined`でも`Array`でもない、つまり`KeyValue`型の値である場合だけ使用されます。`key`と`value`を`choose(chooseKeyValue)`を経由して取得するのは、実は`{#if} ... {/if}`で行う絞り込みを、`chooseKeyValue`関数でも行うことに他なりません。余分な絞り込みなのです。

そこで、どうせ`key`と`value`は上記の`{:else}`節の中でしか使われないのだから、と、次のように`{@const ...}`で書き換えてみたとします:

```html
{:else}
  ...
  {@const key = tree.zoom(into("key"))}
  {@const value = tree.zoom(into("value"))}
  <dt><input type="text" bind:value={$key} /></dt>
  <dd><input type="text" bind:value={$value} /></dd>
  ...
{/if}
```

しかしこれでは、svelte-checkが次のようなエラーを起こします:

```
/svelte-store-tree/example/Tree.svelte:69:42
Error: Stores must be declared at the top level of the component (this may change in a future version of Svelte) (svelte)
```

storeはトップレベル[^script]で宣言しなければならず、`{@const ...}`のなかでstoreを新しく定義して`subscribe`することはできない、というエラーです。「特定の場合にのみ作成して`subscribe`するstore」というのは作れなくなっているんですね。

[^script]: `<script>`ブロックにおけるトップレベルのことのようです。

強引にこのエラーを回避するなら、`<input type="text" bind:value={$key} />`などの部分を独立したコンポーネントに切り出して、そのコンポーネントが`WritableTree<KeyValue>`を受け取るようにする、という手があります。しかしそうしたとしても、型エラーが残ります。上記の`{#if typeof $tree === "string"}`で始まる分岐において型を絞り込めたのは、あくまでも`$tree`、すなわち`tree`が`subscribe`している関数に渡した現在の値であり、`tree`を`WritableTree<Tree>`から`WritableTree<KeyValue>`に絞り込めたわけではないのです。TypeScriptはそこまでこちらの事情をくみ取ってはくれません。

以上のような問題が発生するため、少なくともSvelteにおける**従来のstoreに倣うような形で、ネストした構造を扱う機能を加えるのは、難しい**と判断しました。

なぜこのような、storeの値だけでない、storeそのものに対する型の絞り込みが必要なのでしょうか？それは、Svelteのstoreが値を読み出すAPI（`subscribe`）と、更新に用いるAPI（`set`など）の両方を、一つのオブジェクトに担わせているからです。詳細について、`WritableTree`を次のように簡略化して解説します:

```typescript
WritableTree<T> = {
  // storeの現在の値を取得する。
  // 実際のstoreにはgetはなくsubscribeを使う必要があるが、
  // 説明を単純化するために変更した
  get: () => T;

  // storeの値を更新する。これはオリジナルのstoreと同じ
  set: (newValue: T) => void;
};
```

この`WritableTree`に、例えば`number | undefined`という型を渡して、値に`undefined`を含みうるstoreを作ってみます:

```typescript
WritableTree<number | undefined> = {
  get: () => number | undefined;
  set: (newValue: number | undefined) => void;
};
```

そしてこれから`undefined`を取り除き、`WritableTree<number>`に変換したいとしましょう。`get`メソッドを使って`WritableTree<number>`から値を取得するコードは、`get`が`undefined`を返すことを想定していないので、`get`メソッドが`number`のみを返すよう変換しなければなりません。一方、`WritableTree<number>`に対して`number`の値を書き込むコードは、`set`メソッドに`number`の値しか渡さないので、`(newValue: number | undefined) => void`という型の関数がそのまま使えるのです[^covariant]。

[^covariant]: この違いは「反変・共変」として知られる関係で、かの[プロを目指す人のためのTypeScript入門](https://gihyo.jp/book/2022/978-4-297-12747-3)でも解説されています。

したがって、型の絞り込みによって変換する必要があるのは、実はstoreの値を読み込むAPIだけなのです。Solid.jsのstoreは、storeの現在の値のみを分岐によって絞り込めば事足りる一方、svelte-store-treeは、現在の値を取得するAPIと値を更新するAPIが一つのオブジェクトに含まれている関係上、両方をひっくるめて型の絞り込みを行うAPIが必要なのです。
