---
title: "筒で理解する反変・共変"
type: "tech"
emoji: "🧪"
topics: ["TypeScript", "型システム"]
published: true
---

この記事では、Java、Scala、TypeScriptなど、サブタイピング（subtyping）をサポートする言語であれば間違いなくサポートしているであろう「反変（contravariant）」・「共変（covariant）」について、視覚的なアナロジーを用いつつ解説したいと思います。コード例を含め全てTypeScriptを前提とした説明ですが、同様の機能を持った言語であれば概ね同じことが言えるはずです。

## そもそもサブタイピングとは

サブタイピングとは、型と型との間にサブタイプ（subtype）・スーパータイプ（supertype）という関係を定めて、スーパータイプである型の代わりとして、サブタイプである型を利用できるようにする仕組みです。

例えば、TypeScriptでは`string`型は`Object`型のサブタイプであるので、次のように`Object`型の変数に`string`型の値を代入することができます:

```ts
const o1: Object = "string";
```

一方、次のように`Object`型の値を`string`型の変数に代入することはできません。

```ts
const o2: string = "string" as Object;
// 型エラー: Type 'Object' is not assignable to type 'string'.
```

実際の値としては見るからに文字列型の値であったとしても、`as`演算子で強引に`Object`型としてマークされた値は、`string`型の値として扱われないのです。

[ここまでのコードをTypeScript Playgroundで試す](https://www.typescriptlang.org/play/?#code/MYewdgzgLgBCCMAuGB5ARgKwKbFgXhgCJoAnASzAHNCBuAWAChRJYQAmZUiymA4qclUIwAhhFSYcUGkA)

## 反変・共変とは

反変と共変は、関数やリストのような、型引数を持った複雑な型におけるサブタイプ・スーパータイプの関係を説明するために使う用語です。型引数を受け取る型が具体的な型を受け取ったとき、他の型とどのようにサブタイプ・スーパータイプの関係を成すかを表します。

### 共変1: 単純なプロパティーとして備えている場合

ひとまず比較的単純な、共変の例を示しましょう。

```ts
interface Covariant1<T> {
  value: T;
}
```

上記の`Covariant1`という型は、型引数`T`に対して**共変**です。共変であるということは、`Covariant1`に適当な型を渡して作った型同士 --- 例えば`Covariant1<string>`や`Covariant1<Object>` --- のサブタイプ・スーパータイプの関係を、渡した型同士の関係から**変えない**ことを表します。そのため、例に挙げた`Covariant1<string>`と`Covariant1<Object>`では、`string`が`Object`のサブタイプなので`Covariant1<string>`は`Covariant1<Object>`のサブタイプ、ということになります。

したがって、次のように`Covariant1<Object>`型の変数に`Covariant1<string>`型の値を代入することができます:

```ts
const cov1: Covariant1<Object> = { value: "string" };
```

そしてその逆、`Covariant1<string>`型の変数に`Covariant1<Object>`型の値を代入することはできません:

```ts
const cov1Bad: Covariant1<string> = { value: "string" } as Covariant1<Object>;
// 型エラー: Type 'Covariant1<Object>' is not assignable to type 'Covariant1<string>'.
//             Type 'Object' is not assignable to type 'string'.
```

[ここまでのコードをTypeScript Playgroundで試す](https://www.typescriptlang.org/play/?#code/JYOwLgpgTgZghgYwgAgMIHsBucrDuARgB4AVAPmQG8AoZZbAGwFcIAuZEgbmoF9rqE6EAGcwyQZgLsM2XPjDEA8gCMAVhARgKAXir04zNsgBEo3CADmx5D24Cho8VgIAhOABNpWHHkJEzoBY6eows7KZg5lY2yHDCaN5yfirqmmScQA)

このことを、サブタイプ・スーパータイプの関係より単純な、集合の大小（取り得る値の数）の関係で例えてみます。例えば`Object`は`string`のスーパータイプなので、`Object`は`string`より大きい、と考えます。`Object`には当然`string`以外の型の値が含まれますから、取り得る値の数は大きいと言えますね:

![Object > string](/images/2024-09-variance/object-string.png)

サブタイプ・スーパータイプの関係は、実際のところ全順序な関係ではない、つまりサブタイプでもスーパータイプでもどちらとも言えない場合があるので、単純な「大小」による例えは正確ではありませんが、この記事ではどちらかに該当するケースのみを取り上げるので問題ありません。

この例えにしたがって`Covariant1<Object>`と`Covariant1<string>`型の関係を示すと、次のようなイメージになります:

![Covariant1\<Object\> > Covariant1\<string\>](/images/2024-09-variance/covariant1-object-string.png)

型引数として渡した`Object`や`string`は`Covariant1`の「付属品」として付いているイメージで、その「付属品」の大きさによってサブタイプ・スーパータイプの関係を定めています。

変数は、自身に割り当てられた型[^inference]の「大きさ」にフィットする、器のような役割を果たします。代入しようとしている値の型が、変数に割り当てられた型より「小さい（あるいは同じ大きさの）」場合のみ代入できるわけです。ただし、`Covariant1<Object>`のような「付属品」が付いた型の場合は、「付属品」と「付属品」を付けている型**両方**の型が「小さく（あるいは同じ大きさで）」なければなりません。

[^inference]: TypeScriptを含め、昨今のプログラミング言語では推論された式の型に合わせて変数の型が決まる場合の方が多いですが、この記事では説明しやすさのために、全ての変数に予め型を設定しておきます。

### 共変2: 関数の戻り値として返す場合

`Covariant1`のように、型引数の型を単純にプロパティーとして備えている場合の他、型引数の型を関数の**戻り値**として備えている場合も、共変となります。例えば、下記の`Covariant2`は、型引数`T`をプロパティー`returnValue`関数の戻り値としているので、`Covariant2`は型引数`T`に対して共変です:

```ts
interface Covariant2<T> {
  returnValue: (value: any) => T;
}
```

`Covariant1`と同様、`Covariant2<Object>`型の変数に`Covariant2<string>`型の値を代入することができますし、その逆はできません:

```ts
const cov2: Covariant2<Object> =
  { returnValue: (_value: any) => "string" };
const cov2Bad: Covariant2<string> =
  { returnValue: (_value: any) => "string" } as Covariant2<Object>;
// 型エラー: Type 'Covariant2<Object>' is not assignable to type 'Covariant2<string>'.
//             Type 'Object' is not assignable to type 'string'.
```

[ここまでのコードをTypeScript Playgroundで試す](https://www.typescriptlang.org/play/?#code/JYOwLgpgTgZghgYwgAgMIHsBucrDuARgB4AVAPmQG8AoZZbAGwFcIAuZEgbmoF9rqE6EAGcwyQZgLsM2XPjDEA8gCMAVhARgKAXir04zNsgBEo3CADmx5D24Cho8VgIAhOABNpWHHkJEzoBY6eows7KZg5lY2yHDCaN5yfirqmmScQA)

これも大小関係で例えてみましょう。関数は値を受け取って値を返すものですから、値を入れると値が出てくる、筒のようなもので例えましょう。引数や戻り値における型の違いは、筒の入り口と出口の大きさの違いに相当するわけです。すると`Covariant2<Object>`と`Covariant2<string>`は次のようにイメージされます:

![Covariant2\<Object\> > Covariant2\<string\>](/images/2024-09-variance/covariant2-object-string.png)

※ここでは引数の型は関係がないので、引数に当たる、筒の左側の部分は省略しています。

なぜこのような大小関係になるのでしょうか？実際に`Covariant2<string>`のような、関数を備えた型のオブジェクトを使う側になって考えてみてください。`Covariant2<string>`、つまり`string`型の値を返す関数を持ったオブジェクトのつもりで、`Covariant2<Object>`、つまり`string`型よりも「大きい」値を返す関数を持ったオブジェクトを使うと、次のような問題が起こり得るからです:

![Covariant2\<string\>のつもりでCovariant2\<Object\>の関数を呼び出すと...](/images/2024-09-variance/covariant2-wrong-string1.png)

![Objectのうち、string以外の値が返されてしまうことが！](/images/2024-09-variance/covariant2-wrong-string2.png)

図のとおり、`Covariant2<string>`を使うつもりで`Covariant2<Object>`に含まれる関数を呼び出すと、`string`には含まれないけど`Object`には含まれる型 --- 例えば`number` --- の値が出てきてしまう恐れがあります。`Object`は`string`よりも集合として大きい、すなわち取り得る値の数が多いので、`string`を返す関数のつもりで`Object`を返す関数を扱ってしまうと、想定外の結果が返ってしまうことがあるのです。

以上の理由により、`Covariant2<Object>`は`Covariant2<string>`のスーパータイプである、言い換えると、`Covariant2<Object>`の代わりとして`Covariant2<string>`は利用できるが、その逆、`Covariant2<string>`の代わりとして`Covariant2<Object>`を利用することはできない、と言えます。

### 反変: 関数の引数として受け取る場合

「反変」は、「反」という字が含まれていることから察せられる通り、「共変」とは反対の方向に振る舞います。例を見てみましょう:

```ts
interface Contravariant<T> {
  receiveValue: (value: T) => void;
}
```

`Contravariant`型は、型引数`T`に対して**反変**です。`receiveValue`プロパティーのように、型引数の型を関数の**引数**とする関数を含んでいる場合、対象の型引数に対して反変になります。反変であるということは、`Contravariant`に適当な型を渡して作った型同士 --- 例えば`Contravariant<string>`や`Contravariant<Object>` --- のサブタイプ・スーパータイプの関係を、渡した型同士の関係から**反対にする**ことを表します。

例えば`Object`は`string`のスーパータイプなので、それぞれを`Contravariant`に渡した`Contravariant<string>`と`Contravariant<Object>`におけるサブタイプ・スーパータイプの関係はその逆、すなわち`Contravariant<string>`が`Contravariant<Object>`のスーパータイプ、ということになります。

コードを書いてtscに聞いてみても、確かにそうなるようです:

```ts
const contra: Contravariant<string> =
  { receiveValue: (_value: Object) => {} };
const contraBad: Contravariant<Object> =
  { receiveValue: (_value: string) => {} };
// 型エラー:  Type '(_value: string) => void' is not assignable to type '(value: Object) => void'.
//              Types of parameters '_value' and 'value' are incompatible.
//                Type 'Object' is not assignable to type 'string'. [2322]
```

[ここまでのコードをTypeScript Playgroundで試す](https://www.typescriptlang.org/play/?#code/JYOwLgpgTgZghgYwgAgMIHtxTgNzlYOcAHgBUA+ZAbwChlkoIlgcIA1OAGwFcIAuZAAo8PfslIBKZAF5KOdMAAmAbhoBfGjQSYAzmGTascARiN4CRMMT0EQAc0rS61Bkwgt2XXgMEB9Ed7IAPIARgBWTGBSstRqyGqqhnoGmGDYAEJwiiap2OaEJKERCGCOzlSuzKwcoj7+XmI2oHbRlFRxCUA)

なぜ関係が逆になるという、一見直感に反する結果となるのでしょうか？ここでも、関数を筒のようなもので例えて説明します。

![Contravariant\<string\> > Contravariant\<Object\>](/images/2024-09-variance/contravariant-string-object.png)

今度は関数の引数が型引数の型で置き換えられるので、筒の左側の部分に型を置きました。

こちらも実際に`Contravariant<Object>`のような、`Object`型の値を受け取る関数を持ったオブジェクトを使う立場に立って考えます。`Contravariant<Object>`、つまり`Object`型の値を受け取る関数を含んだオブジェクトのつもりで、`Contravariant<string>`、つまり`Object`型よりも「小さい」値のみを受け取る関数を持ったオブジェクトを扱うと、次のような問題が起こり得るからです:

![Contravariant\<Object\>のつもりでContravariant\<string\>の関数を呼び出すと...](/images/2024-09-variance/contravariant-wrong-object1.png)

![Objectのうち、string以外の値を渡してしまうことが！](/images/2024-09-variance/contravariant-wrong-object2.png)

図のように、`Contravariant<Object>`のつもりで`Contravariant<string>`に含まれている関数を使用すると、`string`には含まれないけど`Object`には含まれる型 --- 例えば`number` --- の値をうっかり渡してしまう恐れがあります。`Contravariant<string>`の関数は`string`のことしか想定していないので、それ以外の値を渡すと当然問題があります。したがって、`Contravariant<string>`を`Contravariant<Object>`の代わりに用いるのは型エラーとなります。

### まとめ

反変・共変とは:

- ある型が型引数の型に対して**共変**である場合は、渡した型引数間のサブタイプ・スーパータイプの関係を**変えない**
- ある型が型引数の型に対して**反変**である場合は、渡した型引数間のサブタイプ・スーパータイプの関係を**反対に**する

どういう型が、ある型引数に対して反変・共変になるか:

- 型引数の型が、関数の**戻り値**（あるいは関数でない、単純なプロパティー）として使われていれば**共変**となり、
- 型引数の型が、関数の**引数**として使われていれば**反変**となる

この違いは、関数を使う側と関数自身、どちらが変数（引数）に代入する値を決める立場にあるか、という点にあります。関数の戻り値は、**関数自身が**変数に代入する値を決める一方、関数の引数は、**関数を使う側が**引数に代入する値を決めるからです。変数に代入する値の型は、変数の型のサブタイプでなければならないので、代入する値の型（引数として渡す型）がサブタイプとなるように定められているのです。

## 参考文献（本文中で言及しなかったもののみ）

- [プロを目指す人のためのTypeScript入門 安全なコードの書き方から高度な型の使い方まで](https://gihyo.jp/book/2022/978-4-297-12747-3)
- [サブタイピング (計算機科学) - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%B5%E3%83%96%E3%82%BF%E3%82%A4%E3%83%94%E3%83%B3%E3%82%B0\_(%E8%A8%88%E7%AE%97%E6%A9%9F%E7%A7%91%E5%AD%A6))
- [Covariance and contravariance (computer science) - Wikipedia](https://en.wikipedia.org/wiki/Covariance\_and\_contravariance\_(computer\_science))
