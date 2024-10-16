---
title: "ChromebookのAndroidランタイムでゲームが遅かった理由を探ってみる"
type: "tech"
emoji: "📊"
topics: ["ChromeOS", "Android", "ベンチーマーク"]
published: true
---

[私の技術以外のことを書いているブログ](https://the.igreque.info/posts/2024/05-chromebook.html)では、[ASUS Chromebook CM30 Detachable](https://www.asus.com/jp/laptops/for-home/chromebook/asus-chromebook-cm30-detachable-cm3001/)というChromebookでAndroid向けのゲームを動かしてみたところ、遅くてまともにプレイできる状況ではなかった、という話を書きました。この記事ではその原因を推測するため、いくつかベンチマークを取ってみます。Chrome OS上のAndroidランタイム（ARCVM）と、比較用の安価なAndroid端末で計測したり、Chrome OSのネイティブなChromeや、Android上のブラウザーで計測したりしてみます。

:::message
結論から言うと、特にGPUが関わるベンチマークにおいて、あまり信頼性の高い結果は得られませんでした😞。ですが、記録のために記事に残しておきます。
:::

# 使用した端末の情報

- Chrome OSの端末:
    - ASUS Chromebook CM30 Detachable (CM3001)
        - CPU: MediaTek Kompanio 520 (8186)
            - 2GHz 2コア + 2GHz 6コア
            - 参考: <https://www.mediatek.jp/products/chromebooks/mediatek-kompanio-520>
        - GPU: Arm Mali G52 MC2 2EE
        - メモリー: 8 GB
        - ディスプレイ: 10.5 インチ
            - 解像度: 1920x1200px
        - Chrome OS: 129.0.6668.99 (Official Build)
        - ARC: 12363237 (SDK のバージョン: 33)
- Androidの端末:
    - SHARP AQUOS wish4
        - CPU: MediaTek Dimensity 700
            - 2.2GHz 2コア + 2GHz 6コア
            - 参考: <https://www.mediatek.jp/products/smartphones-2/dimensity-700>
        - GPU: Arm Mali-G57 MC2
        - メモリー: 4 GB
        - ディスプレイ: 6.6 インチ
            - 解像度: 1612x720px
        - Android: 14

冒頭で紹介した記事で触れた通り、上記のChromebookでは[グルミク](https://d4dj-groovymix.jp/)という音ゲーがとてもプレイしづらかった一方、より安価なAndroid端末であるAQUOS wish4では（演出を抑えるなどの設定はしたものの）問題なくプレイできました。改めてスペックを比較してみるに、数字の上ではCPU・GPU共にAndroidのほうがやや勝っているように見えます[^gpu]。とはいえ、それだけでプレイ体験の差を説明できるほどのものとは思えませんでした。やはりChrome OSのAndroidランタイムによるオーバーヘッドが大きいのでしょうか？

[^gpu]: GPUの性能については[Armが提供するGPUのデータシート](https://developer.arm.com/documentation/102849/0700/?lang=en)を参考にしました。

詳細なスコアは[リポジトリー](https://github.com/igrep/zenn/tree/main/others/2024-10-3-chromebook-benchmark)に載せておきます（スクショのものが大藩ですみません）。

# ベンチマーク1: Androidランタイム上でのベンチマーク

まずは、[Geekbench](https://www.geekbench.com/)というツールを使ってCPUとGPUの性能を比較してみましょう。

| OS                | CPU/Single-Core  | CPU/Multi-Core  | GPU                  |
| ---               | ---------------: | --------------: | -------------------: |
| Chrome OS (ARCVM) | 642              | 1659            | NA（データ取れず）   |
| Android           | 689              | 1822            | 1218                 |

Chrome OSでは直接Geekbenchを実行できないため、ここでいう「Chrome OS」はChrome OSのARCVM上で実行した結果を表しています。CPUの結果はAndroidの方が少し高いようです。これは前述の通りクロック周波数が少し高いCPUなので、予想通りの結果でしょう。

一方、GPUについては残念ながらChrome OSでベンチマークを実行すると、全く進行しないまま開始数分後にクラッシュ💥してしまい、結果が取れませんでした。今回気になっているのはゲームの性能なので、差が出やすそうなのはGPUの方でしょうし、残念です。

# ベンチマーク2: ブラウザー上でのベンチマーク

今回はChrome OSのネイティブなアプリケーション（すなわちChrome）と、同じマシンのARCVM上で動くアプリケーションを比較することで、ARCVMにかかるオーバーヘッドも調べてみます。御存知の通りChrome OSではChromeしかネイティブに動くアプリケーションがないので、ブラウザー上でベンチマークを取ることにしました。

利用したブラウザーは次のとおりです:

- Chrome OSの端末:
    - ネイティブのブラウザー
        - Chrome OS: 129.0.6668.99 (Official Build)
            - （上記と同じ）
    - ARCVM上のブラウザー
        - [WebView Test](https://play.google.com/store/apps/details?id=com.snc.test.webview2): 1.3.9
            - ARCVMでは紛らわしくなるのを避けるためか、Android版のChromeをインストールできなかったので、代わりにこちらのアプリケーションを使いました
        - WebViewのバージョン: 不明（もしかしてネイティブのChromeを使っている？）
- Androidの端末:
    - ネイティブのブラウザー
        - Chrome: 129.0.6668.100
            - 本当はAndroidもWebView Testで揃えたかったのですが、結果の取得がうまくできなかったので止むなくChromeにしました

## [Speedometer](https://browserbench.org/Speedometer3.0/)の結果

[Speedometer](https://browserbench.org/Speedometer3.0/)は、Webアプリケーションのパフォーマンスを測定するツールです。TODOアプリなどのような、実際に使われるであろうアプリケーションを自動で実行して、JavaScriptやDOM操作のパフォーマンスを測定します。

計測結果は次のとおりでした:

| OS (ブラウザー)        | スコア | 誤差[^diff] |
| ---                    | -----: | --------:   |
| Chrome OS (ネイティブ) | 3.32   | ±13.28%    |
| Chrome OS (ARCVM)      | 3.41   | ±4.05%     |
| Android (Chrome)       | 4.83   | ±7.42%     |

[^diff]: ここでいう「誤差」は、各種ベンチマーク項目のうち最も高いスコアと平均スコアとの差の、平均スコアに対するパーセンテージのようです。

意外や意外、ARCVM上のブラウザーの方がネイティブよりも若干高いスコアを出しました。Android端末の方はさらに高いスコアを出しています。ブラウザーについては、ARCVMによるオーバーヘッドは特に大きくないようです。

## [MotionMark](https://browserbench.org/MotionMark1.3.1/)の結果

Speedometerは、JavaScriptとDOM操作のパフォーマンスを測定する一方、ゲームのパフォーマンスに大きく関わるであろう、グラフィック処理の測定は行いません。そこで、[MotionMark](https://browserbench.org/MotionMark1.3.1/)というもう一つのブラウザーベンチマークを実行することで、グラフィック処理の性能を調べてみます。

結果は次の通りでした:

| OS (ブラウザー)        | スコア | fps  | 誤差      |
| ---                    | -----: | ---: | --------: |
| Chrome OS (ネイティブ) | 521.25 | 45   | ±19.28%  |
| Chrome OS (ARCVM)      | 5.01   | 60   | ±65.07%  |
| Android (Chrome)       | 14.00  | 90   | ±154.21% |

ご覧の通り、随分バラバラな結果になってしまいました。いくつもツッコミどころがありますね🧐:

- fpsが揃っていないせいで、公平な比較になっていないのではないかと懸念されます
    - 同じ端末のChrome OS (ネイティブ)とChrome OS (ARCVM)とでさえfpsが異なります。恐らく、ネイティブのChromeでは端末の処理性能に合わせて敢えて抑えているのでしょう
        - 先程のSpeedometerでの結果でAndroid版の方がスコアが高かったのも、同様の理由なんでしょうかね
- そもそもそれ以前に、画面のサイズや解像度が異なるので、純粋にGPUの性能という観点では比較になっていないかもしれません
    - 本当は同じディスプレイに繋いで比較すべきなのでしょうが、Android端末がDisplayPort Alt Modeにも対応していないので諦めました
- 一番大きな問題として、誤差が総じて大きな結果となってしまいました。これについては原因がよく分かりません...

# 総評

上記の通り、特にGPUについてのベンチマークにおいて、信頼性がとても低い結果しか得られませんでした。結局肝心の、Chromebookがゲームプレイ時に性能が低かった原因はよく分かりませんでした😩。とはいえ今回の結果から、Android端末のほうがゲームをしやすかったのは、単純にChromebookより（少なくともCPUにおいて）性能が高いという点と、描画するディスプレイの解像度が低いという点が影響したのではないかと推測しました。ざっとググってみた限り、Chrome OSのARCVMによるオーバーヘッドは特に大きくなかったのではないかと思います。
