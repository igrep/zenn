---
title: "SvelteKitでパッチを当てた外部のコンポーネントを読むまで"
type: "tech"
emoji: "🔗"
topics: ["SvelteKit"]
published: true
---

# 背景

[こちらの、svelte-material-uiというパッケージに送ったPull request](https://github.com/hperrin/svelte-material-ui/pull/655)を作るに当たって[^merge]、修正したバージョンを手元のSvelteKitのプロジェクトで試そうとしたところ、思いのほか苦労したのでやったことを共有します。記憶とコマンドのログなどを元に書くので、ちょっと間違っている箇所があるかも知れませんが、参考になれば幸いです。

[^merge]: （余談）執筆時点で、svelte-material-uiの作者はSvelte 5に対応したsvelte-material-ui v8への対応に集中しているようなので、多分マージしていただくのは時間がかかるでしょう。

# 利用した主なソフトウェア

- Node.js: 20.11.0
- SvelteKit: 1.27.2
- Svelte: 4.2.2
- Vite: 4.5.2
- OS: WSL 2上の Debian 11.9

# うまくいった方法

結論から言うと、[こちらの記事](https://zenn.dev/negi/scraps/ff8cc6bf68521a)を参考に対象のコンポーネントを含むパッケージを`npm link`し、その上で対象のコンポーネントをビルドすれば🆗です。

例として、今回実際に手を加えたsvelte-material-uiにおける[Select](https://sveltematerialui.com/demo/select/)というコンポーネントを修正したとしましょう。

## 1. パッチを当てるパッケージをサブモジュールに

まずは修正したコンポーネントを利用するプロジェクトのディレクトリー（ここでは`svelte-kit-project`という名前にしておきます）に行き、パッチを当てたパッケージを含むリポジトリー、即ち今回の例ではsvelte-material-uiのリポジトリーのフォークを`git submodule add`します。

```bash
$ cd ~/path/to/your/svelte-kit-project
$ git submodule add https://github.com/igrep/svelte-material-ui
```

`npm link`の仕様上、実際のところ`git submodule`を使う必要は愚か、`svelte-kit-project`ディレクトリー以下に置く必要すらないのですが、Gitリポジトリーとしてでしか取得できないパッケージを利用する際は、次の理由により`git submodule add`で追加するのをおすすめします:

- 使用しているバージョンをGitのコミット単位で固定できるので、特に変更したバージョンの利用が長期化した場合において安全
- プロジェクトで試した結果に基づき、パッケージに更に変更を加えた場合、そのまま`git commit`・`git push`でフォークにフィードバックできる

必要なら、ここで`git submodule add`したリポジトリーのブランチを、当てたいパッチを含むものに切り替えたりしておきましょう:

```bash
$ cd svelte-material-ui
$ git checkout -t origin/your-patched-branch
```

## 2. パッチを当てたパッケージをビルド

svelte-material-uiに含まれる各パッケージは、ビルドしないと使えません。今回修正したいのは`Select`コンポーネントを含む`@smui/select`パッケージなので、`packages/select`ディレクトリーに移動して`npm run build`します:

```bash
# svelte-material-ui ディレクトリーから
$ cd packages/select
$ npm run build
```

:::message alert
私が試した限り、この手順は`Select`コンポーネントを**修正するたびに必要**でした。ご注意ください。
:::

## 3. `npm link`で利用するプロジェクトから参照する

続いて、`npm link`コマンドを使って、`svelte-kit-project`の`node_modules`にビルドしたパッケージへのシンボリックリンクを張ることで、`svelte-kit-project`の依存関係として使えるようにします。冒頭でも触れた[こちらの記事](https://zenn.dev/negi/scraps/ff8cc6bf68521a)を参考に、まずは`@smui/select`パッケージをグローバルなパッケージとして`npm link`します:

```bash
# svelte-material-ui/packages/select ディレクトリーから
$ sudo npm link
```

ℹ️手元の環境では`npm`のグローバルなパッケージはroot権限がないとアクセスできない場所にインストールされていたので、`sudo`をつけました。

それができたら、いよいよ修正した`@smui/select`を利用するプロジェクト、本稿で言うところの`svelte-kit-project`で`npm link`しましょう:

```bash
# svelte-material-ui/packages/select ディレクトリーから
# svelte-kit-project ディレクトリーに戻る
$ cd ../../..

# 念のためバージョンも指定しておく
$ npm link @smui/select@7.0.0-beta.18
```

ℹ️[`npm link`のドキュメント](https://docs.npmjs.com/cli/v10/commands/npm-link)を改めて読んだところ、実際のところ、`svelte-kit-project`側で直接`npm link svelte-material-ui/packages/select`と書くだけでよいみたいですが、今回私が参考にして試した手順では上記の通り一旦グローバルなパッケージとしてインストールしたのでその通りに書きました。

## 4. `npm install`する

「ここまでの手順で`svelte-kit-project`の`node_modules`には`npm link`した`@smui/select`が見えているんだから、これで十分だろう」と思ってそのまま`vite`を実行してみましたが、これだけではどうも変更が反映されないようでした。あれこれ触ってみたところ、どうやら修正した`@smui/select`を使う`svelte-kit-project`において、再度`npm install`しなければらなかったようです:

```bash
# svelte-kit-project ディレクトリーから
$ npm install
```

## 5. vite.config.ts を編集する

前の手順を終えて再度`vite`を起動すれば、`svelte-material-ui`ディレクトリーにチェックアウトしたバージョンの`Select`コンポーネントが利用されるようになっている... はずが、viteが次のエラーを出すようになってしまいました:

```
Failed to load url /svelte-material-ui/packages/select/dist/index.js (resolved id: /home/user/path/to/your/svelte-kit-project/svelte-material-ui/packages/select/dist/index.js) in /home/user/path/to/your/svelte-kit-project/src/lib/Program.svelte. Does the file exist?
Failed to load url /svelte-material-ui/packages/select/dist/index.js (resolved id: /home/user/path/to/your/svelte-kit-project/svelte-material-ui/packages/select/dist/index.js) in /home/user/path/to/your/svelte-kit-project/src/lib/Program.svelte. Does the file exist? (x2)
Error: Not found: /svelte-material-ui/packages/select/dist/index.js
    at resolve (/home/user/path/to/your/svelte-kit-project/node_modules/@sveltejs/kit/src/runtime/server/respond.js:483:13)
    at resolve (/home/user/path/to/your/svelte-kit-project/node_modules/@sveltejs/kit/src/runtime/server/respond.js:277:5)
    at #options.hooks.handle (/home/user/path/to/your/svelte-kit-project/node_modules/@sveltejs/kit/src/runtime/server/index.js:49:56)
    at Module.respond (/home/user/path/to/your/svelte-kit-project/node_modules/@sveltejs/kit/src/runtime/server/respond.js:274:40)
    at process.processTicksAndRejections (node:internal/process/task_queues:95:5)
The request url "/home/user/path/to/your/svelte-kit-project/svelte-material-ui/packages/select/dist/index.js" is outside of Vite serving allow list.

- /home/user/path/to/your/svelte-kit-project/src/lib

... 省略 ...

Refer to docs https://vitejs.dev/config/server-options.html#server-fs-allow for configurations and more details.
```

最後に書かれている`Refer to docs https://vitejs.dev/config/server-options.html#server-fs-allow for configurations and more details.`に従い、次のように`vite.config.ts`に`server.fs.allow`という設定を加えれば解決できました:

```ts:vite.config.ts
export default defineConfig({
  // ... 省略 ...
  server: {
    fs: {
      allow: ['./svelte-material-ui/'],
    },
  },
});
```

多分、セキュリティーのために意図しないファイルを開発用サーバーから参照できないようにしているんでしょう。その際シンボリックリンクの参照先で判断するようになっているみたいですね。

# うまくいかなかった方法

参考までに、**うまくいかなかった**方法とその際発生したエラーについても、覚えている範囲で共有いたします。

## package.json の `file:` プロトコルを使う

従来、私はnpmにリリースされていないバージョンのパッケージを用いるときは、`npm link`ではなく[この`file:`プロトコルを使った方法](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#local-paths)を使っていました。

やり方は簡単です。修正したバージョンを使用するプロジェクトの`package.json`を編集して、`dependencies`あるいは`devDependencies`における、該当のパッケージのバージョンが書かれた箇所を、`file:<該当のパッケージへのパス>`に置き換えるだけです（「うまくいった方法」と同様、予め該当のパスに`git submodule add`している前提です）。今回編集した`@smui/select`はSvelteのライブラリーなので、`devDependencies`を編集しましょう:

```js
{
  "name": "svelte-kit-project",
  "devDependencies": {
    "@smui/select": "file:./svelte-material-ui/packages/select",
    // ... 省略 ...
  },
  // ... 省略 ...
}
```

この状態で`@smui/select`をビルドした上で`npm install`し、`vite`を起動したところ、次のようなエラーに遭遇してしまいました:

```
[plugin:vite:import-analysis] Missing "./src/Select.svelte" specifier in "@smui/select" package
```

エラーメッセージで検索してもハッキリわからず、状況から考えて「sveltekit local dependencies」などといった単語で検索したところ、[こちらのissue](https://github.com/sveltejs/kit/issues/4261)が見つかりました。いくつかワークアラウンドが提案されていたので試してみましたがいずれもうまくいかず（あるいは本当に正しく適用できているかもわからず）、[Svelte JapanのDiscord](https://disboard.org/ja/server/777141291800723468)で質問し、頂いた答えをきっかけにあれこれ調べているうちに`npm link`の存在を思い出し、解決に至りました😌。
