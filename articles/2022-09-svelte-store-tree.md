---
title: "svelte-store-treeというライブラリーをリリースしました"
type: "tech"
emoji: "🚉"
topics: ["TypeScript", "Svelte", "状態管理"]
published: true
---

先日、Svelte用の状態管理ライブラリーをリリースしました:

[svelte-store-tree - npm](https://www.npmjs.com/package/svelte-store-tree)

名前の通り、[Svelteのstore](https://svelte.jp/tutorial/writable-stores)という機能について、tree、すなわち木のように入れ子の構造を扱いやすくするメソッドを加えたものです[^implementation]。現在お仕事のアプリケーションで扱っている、複雑な木構造をなるべく素直に扱えるようにするために開発しました。

[^implementation]: 「加えた」とは言っても、実装は本家のstoreをコピペしてから内部構造ごと書き換えたものなので、正確には「storeを参考に拡張した」という方が正しいですが。

# 使い方の例

以降では、[こちらで動かせるサンプルアプリ](https://codesandbox.io/p/github/igrep/svelte-store-tree/draft/floral-sound)における、以下のような型を扱います:

```typescript
export type Tree = string | undefined | KeyValue | Tree[];

export type KeyValue = {
  key: string;
  value: string;
};
```

いろいろな値を取り得る、再帰的な構造になっていますね。

なお、[このサンプルアプリのコードはGitHubにも](https://github.com/igrep/svelte-store-tree/tree/main/example)置いているので、今後具体的な箇所を例示する際は執筆時点のバージョンにおける該当の行へのリンクを張ります。

## `WritableTree`オブジェクトの作成

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

本家の`writable`関数と同様に、storeの初期値を引数で指定します。`Tree`型はunion型なので、型推論を助けるために型パラメーターを明示しておきましょう。

## 普通に`subscribe`したり`set`したりする

svelte-store-treeは本家Svelteのstoreのように[Store contract](https://svelte.jp/docs#component-format-script-4-prefix-stores-with-$-to-access-their-values-store-contract)を守っているので、おなじみのドルマーク `$` で自動で`subscribe`したり、two-way binding機能を通じて`set`することができます:


```svelte
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

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/01c95ab675f2d8f43f6b869e888ba7d4be3131da/example/Tree.svelte#L36-L42)

## `zoom`して、部分木に対する`WritableTree`を作る

svelte-store-treeは、部分木に対するstoreを作るためのAPIとして、以下の`zoom`で始まる名前のメソッドを提供します:

- `zoom<C>(accessor: Accessor<P, C>): ReadableTree<C>`
    - `Accessor`という、親`P`から子`C`を取得したり、親`P`における子`C`の値を書き換える関数のペアを渡すことで、読み取り専用のstore tree (`ReadableTree`)を作る
    - `ReadableTree`は、子のstore treeも含めて`set`できなくなっているstore tree
- `zoomIn<K extends keyof P>(k: K): ReadableTree<P[K]>`
    - `Accessor`を都度作るのは面倒なので、親のオブジェクトが持つフィールドの名前（または配列のindex）を指定することで、子の`ReadableTree`を作る
- `zoomWritable<C>(accessor: Accessor<P, C>): WritableTree<C>`
    - `zoom`と同様に`Accessor`を渡すことで、読み書きできるstore tree (`WritableTree`)を作る
    - `WritableTree`は`set`できるstore tree。子孫が`zoom`や`zoomIn`を使用して`ReadableTree`を作るまでは`set`できる
- `zoomInWritable<K extends keyof P>(k: K): WritableTree<P[K]>`
    - `zoomIn`の`WritableTree`を返すバージョン

サンプルアプリでは、次のように`zoomInWritable`を使うことで`KeyValue`型の値が持つ`key`と`value`を書き換えるための`WritableTree`を作っています。

```typescript
const key = tree.zoomInWritable("key");
const value = tree.zoomInWritable("value");
```

... が、こちらは正しくありません。svelte-checkで調べてみると、TypeScriptが次のような型エラーを出します:

```typescript
/svelte-store-tree/example/Tree.svelte:14:35
Error: Argument of type 'string' is not assignable to parameter of type 'never'. (ts)
  const keyValue = tree.chooseWritable<KeyValue>(chooseKeyValue);
  const key = tree.zoomInWritable("key");
  const value = tree.zoomInWritable("value");


/svelte-store-tree/example/Tree.svelte:15:37
Error: Argument of type 'string' is not assignable to parameter of type 'never'. (ts)
  const key = tree.zoomInWritable("key");
  const value = tree.zoomInWritable("value");
```

`parameter of type 'never'`とあるとおり、引数の型が`never`になっていて分かりづらいですが、これは`tree`の型が`WritableTree<Tree>`、つまり`Tree`型を含む`WritableTree`となっていて、その`Tree`型は先ほどの定義通り`undefined`も取り得るunion型なので、`keyof`の結果が`never`になってしまうためです。

これを修正するには、`WritableTree<Tree>`をなんとかして`WritableTree<KeyValue>`に変換しないといけません。それを実現するのが次の節で紹介する`chooseWritable`です。

## `choose`して、storeの値が特定の場合のみ値を取得する

`WritableTree`並びに`ReadableTree`は、`choose`と`chooseWritable`と言うメソッドで、storeの値が特定の条件にマッチする場合だけ`subscribe`している関数を呼ぶ、storeを作ることができます。

サンプルアプリから、`chooseWritable`を使っている箇所を抜き出しましょう:

```typescript
...
const keyValue = tree.chooseWritable<KeyValue>(chooseKeyValue);
...
```

`chooseWritable`は、引数として、「storeの中の値（上記の場合`Tree`型の値）を受け取って、別の値、あるいは`Refuse`を返す関数」を受け取ります。上の例における`chooseKeyValue`ですね。実装は次の通りです:

```typescript
export function chooseKeyValue(tree: Tree): KeyValue | Refuse {
  if (tree === undefined || typeof tree === "string" || tree instanceof Array) {
    return Refuse;
  }
  return tree;
}
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/01c95ab675f2d8f43f6b869e888ba7d4be3131da/example/tree.ts#L62-L67)

`chooseWritable`が返した`WritableTree`は、`chooseKeyValue`のような関数が`Refuse`という専用のunique symbol[^refuse]を返した場合は`subscribe`している関数を呼ばず、それ以外の値を返した場合はその値で`subscribe`している関数を呼びます。

[^refuse]: 「zoom」、「choose」、「refuse」の3つで韻を踏んでいます。

なぜ`boolean`を返す関数にしないのか、あるいは`undefined`を使わず`Refuse`という専用の値を用意したのか、という疑問が浮かぶ方もいらっしゃるかも知れません。

まず`boolean`を返す関数にしないのは、TypeScriptによる型の絞り込みで`WritableTree`の中の型を変換するため、変換結果を関数の型で明示しないとならないからです。ユーザー定義の型ガード（戻り値の型が`Tree is KeyValue`のようになっている関数）は、`choose`に渡す高階関数には使えませんでした。

それから、`Refuse`という専用の値を用意したのは、`{ property: T | undefined }`のような、`undefined`を含むプロパティーも柔軟に扱えるようにするためです。

話を戻しましょう。先ほど`tree.zoomInWritable("key")`などと書いて型エラーになっていた箇所は、次のように`chooseWritable`と組み合わせることで修正できます:

```typescript
const keyValue = tree.chooseWritable<KeyValue>(chooseKeyValue);
const key = keyValue.zoomInWritable("key");
const value = keyValue.zoomInWritable("value");
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/01c95ab675f2d8f43f6b869e888ba7d4be3131da/example/Tree.svelte#L13-L15)

あとは`zoomInWritable`が返した`key`・`value`を使うことで、`KeyValue`型の`key`と`value`だけをtwo-way bindingすることができます:

```html
<dl>
  <dt><input type="text" bind:value={$key} /></dt>
  <dd><input type="text" bind:value={$value} /></dd>
</dl>
```

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/01c95ab675f2d8f43f6b869e888ba7d4be3131da/example/Tree.svelte#L68-L71)

## 子による更新に親が応じる

これまでの例は、実際のところあまりstore treeの良さを生かせていません。わざわざ一つのデータ構造が入った一つのstoreとして管理しなくても、個別のコンポーネントの状態としてそれぞれの値を管理すれば事足りるからです。

svelte-store-treeはもちろんそれより一歩踏み込んでいます。それは、**子における更新を、直接の親だけに伝播できる**、という点です。

例えば、先ほどの`key`と`value`をtwo-way bindingした二つの`<input>`のうち、`key`の`<input>`が更新された場合をイメージしましょう:

```html
<dl>
  <dt><input type="text" bind:value={$key} /></dt>
  <dd><input type="text" bind:value={$value} /></dd>
</dl>
```

この場合、`key`に`set`した値は、それぞれ`key`自身を`subscribe`している関数と、その親である`keyValue`をはじめとする、直接の先祖を`subscribe`している関数に伝わります。兄弟に当たる`value`を`subscribe`している関数や、親の兄弟などを`subscribe`している関数には伝わりません。

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

[該当箇所](https://github.com/igrep/svelte-store-tree/blob/01c95ab675f2d8f43f6b869e888ba7d4be3131da/example/Root.svelte)

子における更新が確実に伝播するので、親において木全体に対する処理を書くことも容易にできます！

# 今後

私のTypeScript力が不足していたこともあり、他のライブラリーを調べているうちにいろいろ改善点が見えてきた[^competitors]ので、使い方を紹介した後で恐縮ですが、APIをいろいろ修正したいと思います。乞うご期待。

[^competitors]: 作る前に他のライブラリーを調べろよ、という批判は甘んじて受けます😥
