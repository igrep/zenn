---
title: "TypeScriptで、未初期化のインスタンス変数を意図せず参照してしまってもコンパイルエラーにはならない場合がある"
type: "tech"
emoji: "⚠️"
topics: ["TypeScript"]
published: true
---

例えば「[コンストラクタのメソッド利用で注意すること](https://irof.hateblo.jp/entry/2016/01/09/231631)」という記事でも解説されているとおり、Javaなど他の言語でもありふれた話だと思うのですが、TypeScriptでも油断してハマってしまったので、注意喚起のために共有します。

:::details こちら👇のとおりTypeScriptのIssueにも一応報告しました

...が、Duplicateだったようです。ミスった！ちゃんと検索したはずなのになぁ😞

[Uninitialized instance properties can be accessed via initializers · Issue #57984 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/57984)
:::

# 例

次のTypeScriptのコードは特にコンパイルエラーを起こしませんが、実行時に`this.a`が`undefined`になり、結果として`TypeError: Cannot read properties of undefined (reading 'toUpperCase')`が発生します:

```typescript
class Example {
  a: string;
  b = this.useA();
  constructor(a: string) {
    this.a = a;
  }

  useA(): string {
    return this.a.toUpperCase();
  }
}

console.log(new Example('example').b);
```

理由は簡単です。`b = this.useA();`という行で`this.useA()`が呼ばれるのですが、その時点では`this.a`は初期化されていないため`undefined`になるからです。

次のように、`b`の初期化時に`a`を明示的に参照すれば、コンパイルエラーが発生します。**インスタンスメソッドを挟むと、`a`を使っているかどうか分からなくなってしまう**ようですね。

```typescript
class Example {
  a: string;
  b = this.a.toUpperCase();
  constructor(a: string) {
    this.a = a;
  }
}
```

エラーメッセージ:

```
Property 'a' is used before its initialization.
```

# 対策

次のような、より簡潔な`class`の書き方をしていれば、この問題に出遭うことはないでしょう:

```typescript
class Example {
  b = this.useA();
  constructor(public a: string) {}

  useA(): string {
    return this.a.toUpperCase();
  }
}

console.log(new Example("hello"));
```

しかし、この方法はいわゆる「hard private」なインスタンス変数では使えません。`public a: string`のように、コンストラクターの引数でhard privateなインスタンス変数を宣言する方法がないからです:

```typescript
class Example {
  b = this.useA();

  constructor(#a: string) {}
  //          ^^ これはエラー！

  useA(): string {
    return this.#a.toUpperCase();
  }
}
```

というわけで、hard privateなインスタンス変数を使う場合、愚直にコンストラクター内で初期化しましょう:

```typescript
class Example {
  #a: string;
  b: string;
  constructor(a: string) {
    this.#a = a;
    this.b = this.useA();
  }

  useA(): string {
    return this.#a.toUpperCase();
  }
}

console.log(new Example('example').b);
```

私が最初にこの問題にハマったのもhard privateなインスタンス変数を使っていたからなので、書いておきました。

# 余談

😓そもそもこの問題にハマるような設計がまずいのでは？
