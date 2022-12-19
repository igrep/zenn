---
title: "Axiosでmultipart/form-data形式の情報をPOSTするときにハマった"
type: "tech"
emoji: "📮"
topics: ["TypeScript", "axios", "Nodejs"]
published: true
---

# 試した環境

- Node.js: v18.12.1
- axios: v1.2.0
- form-data: v4.0.0
- OS: Windows 10 21H2

# 背景

とある、社内のAPIを公開していないRails製アプリケーションのformに自動で投稿したかった。

# ハマったこと1: `FormData`が紛らわしい

検索して出てくる「[axios v0.27からmultipart/form-dataが簡単に送れるようになった](https://qiita.com/the_red/items/b5668ee9d3bcc5ff0adf)」という記事を参考に、下記のようなコードを書いていました:

```typescript
const form = new FormData();
form.append('file', '');
form.append('other_params', 'other value');

await axios.post(url, form, config);
```

ここで出てくる`FormData`は、[前述の記事](https://qiita.com/the_red/items/b5668ee9d3bcc5ff0adf)では手前の行で`const FormData = require('form-data')`しているとおり、[`form-data`というパッケージ](https://www.npmjs.com/package/form-data)が提供する`FormData`であるべきなのですが、当初私は、[組み込みのAPIとして提供されている`FormData`](https://developer.mozilla.org/ja/docs/Web/API/FormData/FormData)と勘違いしてそのまま使っていました。`append`メソッドまで概ね同じ使用方法なので紛らわしい！

結果、いざ`axios.post`を実行すると次のようなエラーが発生していまいました:

```javascript
AxiosError: Data after transformation must be a string, an ArrayBuffer, a Buffer, or a Stream
    at dispatchHttpRequest (file:///node_modules/axios/lib/adapters/http.js:246:23)
    at new Promise (<anonymous>)
    ... 以下略 ...
```

こちらのエラーについて検索したところ、次のissueのとおり、`Axios`のインスタンスを作る際に「response transformer」というものを設定しなかった場合に発生すると報告されていました:

["Data after transformation must be a string, an ArrayBuffer, a Buffer, or a Stream" when making POST request with `Axios` instance · Issue #4710 · axios/axios](https://github.com/axios/axios/issues/4710)

しかし、今回は特にAxiosのインスタンスを作る際そのような設定はしてませんでしたし、どうやら該当するわけではなさそうでした。

どうしたものかとエラーが起きた周辺のソースを読んだりしているうちに、「どうやらこれは`post`メソッドに渡したリクエストボディーの型が違う」と判断し、やっとform-dataパッケージの存在に気づいたのでした😥

# ハマったこと2: 空のファイルを送るには？

`axios.post`を実行できた、と思いきや、今度はサーバーがエラーを返してきました:

```json
{"status":"error","message":"undefined method `read' for \"\":String"}
```

Rubyを普段から使っている方ならおなじみの、`NoMethodError`のようですね。冒頭で触れたとおり問題のアプリケーションはRails製なので、恐らくパラメーターとして間違った型の値を送ってしまったのが原因なのでしょう。

投稿したいformは`<input type="file"/>`、つまりファイルを送るための欄を含んでいるため、multipart/form-data形式に従って、ファイルを送る体でリクエストボディーを組み立てなければならないようです。先ほどの`form.append('file', '');`のように、空文字列を指定するだけじゃダメなんですね。

今回はひとまず空のファイルを送れれば十分だったので、試行錯誤した結果次のようなコードにたどり着きました:

```typescript
function createEmptyStream(): Readable {
  return new Readable({
    read(_size: number) {
      this.push(null);
    }
  })
}

form.append('file', createEmptyStream(), { filename: 'empty.txt' });
```

ファイルの中身として空の`Readable`なストリームと、ファイル名として`{ filename: 'empty.txt' }`と言うオプションを渡しました。今回参考にしたページでは`filename`は指定されていないことが多かったのですが、問題のアプリケーションでは指定しないとうまく処理してくれませんでした。

なお、上記の通り空の`Readable`を作る際は、`read`メソッドで直ちに`this.push(null);`と実行しましょう。`this.push('');`などと、間違って空文字列を渡すと無限ループに陥るのでご注意ください🔁。
