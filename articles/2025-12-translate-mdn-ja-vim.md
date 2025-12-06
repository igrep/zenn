---
title: "MDNの翻訳を行う際の定型作業を自動化するVim scriptを適当に書いた"
type: "tech"
emoji: "🚀"
topics: ["Vim"]
published: true
---

この記事は[MDNとかをみんなで編集！翻訳！ Advent Calendar 2025](https://qiita.com/advent-calendar/2025/mdn-translation-community-japan) 12日目の記事です。

タイトルのとおりMDNの翻訳を行う際、とりわけまだ日本語版がないページの翻訳を始める際の作業を自動化するVim scriptを書きました。適当なディレクトリーにダウンロード[^plugin]した上で`:source translate-mdn-ja.vim`などと実行すれば利用できます。実行すると、現在開いている英語版のmarkdownファイルを日本語版のパスにコピーした上でfront-matterを書き換えたりしてくれる、`BeginTranslation`関数が使えるようになります。他にも、英語版のファイルを最後に編集した時のコミットハッシュを取得する`GetLatestSourceCommit`関数なども提供しています。

[^plugin]: 十分に小さくニッチだし、適当に済ませたかったのでVim pluginとしてではなく単一のスクリプトファイルとして提供しています。機能追加などはご勝手にしてください。

詳しくはこちらのソースコード冒頭にあるドキュメントをご覧ください:

@[gist](https://gist.github.com/igrep/3618bcdf0294c81bda3481464be96bc8)
