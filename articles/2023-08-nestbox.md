---
title: "Nestboxでスマホを開発環境にするまで"
type: "tech"
emoji: "📱"
topics: ["Nestbox", "Android"]
published: true
---

# 🪺Nestboxとは

Nestboxとは、Android 13以降で利用できるようになった「[Android仮想化フレームワーク](https://source.android.com/docs/core/virtualization?hl=ja)」を応用して開発された、Android端末上で仮想マシン（主にUbuntuなど普通のLinuxディストリビューション）を動かすためのアプリケーションです。[前回の「1人以上で勉強会」](2023-06-03-one-plus)で触れたとおり、AndroidがOSとして提供する仮想化支援機構の上で動作するので、高速に動作します。

今回私は、root化したPixel 6にNestboxをインストールし、iPadからsshで操作できるようにしたことで、それなりに快適な外出先での開発環境を整備できたので、それまでにやったことや注意事項を共有します。

# ⚠️注意事項

執筆時点で私が知る限り、利用できるのは次の端末だけです。作者による最初の紹介記事、「[Exclusive: Nestbox - Easy Linux VMs for Pixel 6+](https://www.patreon.com/posts/74333551)」ではタイトルに「Pixel 6+」とあるのでPixel 6以降のAndroid 13が搭載された端末であればどれでも使えるように見えるのですが、[最新版についての記事](https://www.patreon.com/posts/nestbox-beta-7-76632132)のコメントでも確認できるとおり、**Pixel 7aやPixel Tabletでは動きません**。かくいう私自身もPixel Tabletで試して敢えなく失敗しました😭。

- root化しなくても利用できる端末
    - Pixel 7 Pro
    - Pixel 7
- root化すれば利用できる端末
    - Pixel 6a
    - Pixel 6

後述の通り、今回私はPixel 6をroot化してNestboxを利用できるようにしたので、大雑把な手順（といってもroot化の手順自体は割愛しますが）と、Debianをインストールしてから行ったことを共有します。

# ⏬ダウンロード

執筆時点における最新版は、[先ほども触れたこちらのPatreonの投稿](https://www.patreon.com/posts/nestbox-beta-7-76632132)にあります。作者である[kdrag0n](https://www.patreon.com/kdrag0n/membership)さんが提供するメンバーシップのうち、「Exclusive」というランク以上のものに登録すればダウンロードできます。最低月額5USDです[^once]。

[^once]: 最後の投稿が1月28日で、最近更新されていないようなので、とりあえず試したい！という場合は一度だけ支払ってダウンロードしてからすぐ解約、というのもありかもしれません。私はもうちょっと応援したい気持ちがあるので続けますが...

# ⚙️インストール（の前にroot化など）

インストールそのものは、前述のPatreonの投稿からapkファイルをダウンロードしてインストールするだけです。Pixel 7のようなroot化不要の端末なら実に簡単に使えるでしょう。しかし、（繰り返すようで恐縮ですが）今回はPixel 6へのインストールですので、下記👇の記事を参考にroot化するところから始めました。

[Pixel6･Pixel6 ProでRootを取る - Qiita](https://qiita.com/hntk/items/4929f97a1fc78217e354)

こちらの記事は、細かいところに省略や間違いと思しき箇所があるものの、概ね手順通りにやればroot化できました。

その上で、Pixel 6をfastbootモードにし、root化の際使用したPCから次のコマンドを実行し、Android仮想化フレームワークに必要な、pKVMを有効にしましょう（Pixel 7やPixel 7 Proなら、この手順も不要です）。

```
fastboot oem pkvm enable
```

後は、Patreonからダウンロードしたapkファイルをインストールすれば、出来上がりです✅。

# 🚀インストールしてから実行したコマンドなど

Nestboxを起動できたら、右下の➕ボタンをタップして、インストールしたいLinuxディストリビューションを選択しましょう。選んだディストリビューションがインストールされます。今回は個人的な好みでDebianにしました。お馴染みUbuntuやNixOSも利用できるので、お好みで。

## ユーザーの追加

ディストリビューションのインストールが完了したら、直ちにターミナルエミュレーターがrootユーザーで立ち上がります。デフォルトでは普通のユーザーが用意されていないみたいなので、普段使いのユーザーを追加しておきましょう:

hoge

```bash
# もちろんユーザー名はお好みで！
adduser igrep

# 何かと便利ですし、 sudo を使えるようにもしておきましょう
adduser igrep sudo

# パスワードの設定も大事ですね！
sudo -u igrep passwd
```

## SSHの設定

最小限の設定ができたら、スマホでコマンドを入力し続けるのも辛いですし、SSHでお好きなマシンからログインできるようにしましょう。`apt install openssh-server`してSSHサーバーを有効にしたら、公開鍵の設定や、sshd\_configの設定（`PasswordAuthentication no`とか！）をしておきましょう（ありふれたものなので詳細は割愛します）。

Nestboxで起動しているLinuxに外部のマシンからアクセスする場合、ポートフォワーディングの設定を行う必要があります。ホストOSであるAndroidが、ルーターになって外部からのアクセスをNATするわけですね。SSHでアクセスする場合ももちろんポートフォワーディングの設定が必要です。公開鍵などの設定が済んだらNestboxの設定⚙️を開いて「Port fowarding」という項目をタップし、右上の➕ボタンを押して、次のように入力します:

- Container:
    - ポートフォワーディングを設定したいVMを選びます。今回のように一つしか作っていないなら特に迷うことはないでしょう
- Container port:
    - VMがリッスンしているポート番号。SSHのデフォルトから変えていないなら22番ですね
- Android port:
    - Pixel 6としてリッスンするポート番号。ポートフォワーディングを設定すると、ここで設定したポート番号へのリクエストを、上記の「Container port」へ転送するようになるわけです
    - 残念ながら、いわゆるwellknown portは使えないようなので22番ポートは指定できません[^root-wellknown]
- Open to other devices:
    - パソコンなどPixel 6の外部からアクセスできるようにする場合はチェック☑️する必要があるみたいです

[^root-wellknown]: root化してroot権限をとっていてもダメです！

入力できたら右上のチェックマーク✔️を押して設定完了です。これでパソコンから次のようなコマンドで、Nestbox上のLinuxにログインできます:

```bash
ssh -p <「Android port」で設定したポート番号> <ユーザー名>@<Pixel 6のIPアドレス>
```

### テザリングで接続しているマシンからログインするときのコツ

最後に紹介したsshコマンドにおける`<Pixel 6のIPアドレス>`の箇所を求める際のちょっとしたコツも共有しましょう。お互い同じネットワーク（ルーター）に所属している場合、Pixel 6側のIPアドレスを「設定」の「ネットワークとインターネット」➡️「インターネット」から自分が接続しているネットワーク（Wi-Fiで接続しているなら、該当するSSID）をタップしてスクロールすれば分かります。

一方、テザリングで、Pixel 6にログインするマシンに直接接続している場合、Pixel 6の画面から、Pixel 6自身のIPアドレスを調べる方法はありません（私が見落としてなければ！）。その場合、Pixel 6にログインするマシンから調べることができます。テザリングというのは、要するにスマホをルーターにすることでスマホを通じてインターネットに接続することなので、接続したマシンからルーターのIPアドレス、すなわちデフォルトゲートウェイのIPアドレスを調べれば、それがそのまま`ssh`コマンドに渡すIPアドレスとして使えます。

私の場合Windowsから接続したので、PowerShellの機能を活用して、次のコマンドで一発でログインできるようにしました:

```powershell
ssh -p 10022 "igrep@$(Get-NetIPConfiguration | Where { $_.InterfaceDescription -eq "UsbNcm Host Device" } | Foreach { $_.IPv4DefaultGateway.NextHop })"
```

`Get-NetIPConfiguration`で始まって`{ $_.IPv4DefaultGateway.NextHop }`のところまでが、デフォルトゲートウェイのIPアドレスを取得する処理です。

USBテザリングでもBluetoothテザリングでも、手法を問わずこの方法は利用できます。

# その他個人的にやったこと・所感

以上の手順で、Pixel 6を小さなLinuxとして遊べるようにするための最低限の準備ができました。後はお好みで、好きなエディタや好きなプログラミング言語の開発環境をインストールするなりすればよいでしょう。私の場合、NeovimやNode.jsをインストールしたりしました（Haskellもやろうと思えばできそうですが、最近Haskellの開発は限られたタイミングにしかしてないので...😥）。特段難しい所もなかったので手順は省略します。

それから、個人的にお馴染み、[Unison](https://github.com/bcpierce00/unison)も設定することで、メインのPCと簡単に同期できるようにしました。これも普通のLinux環境で行った場合と特にやることは変わりません。詳しくは次回の[「1人以上で勉強会」](https://www.youtube.com/playlist?list=PLRVf2pXOpAzJ-OTJhS1VayvRLm2K7iB2v)で詳しくお話しします（いつになるかな...）。

VMへのソフトウェアのインストール以外に、iPadから接続できるよう設定もしました。といっても[Termius](https://termius.com/)をインストールしてUSBテザリングやBluetoothテザリングなどと合わせて設定しただけですが。

ここまで諸々設定して以来、ちょくちょくカフェに行って集中して開発してみてます。転地効果なのか、結構集中できますね。これまでに試したUserLAndと比べてこれといったトラブルもなく、速度も満足できる水準で、快適です。やろうと思えばiPadなどからログインする間もなく、Pixel 6だけで作業できるのも素晴らしいです（当然タイプしにくいですし、滅多に使うことはないと思いますが...）。

唯一不満なのは、Raspberry Piを外出先での開発環境にしていたときのように、mDNSでの接続（ドメインでの接続）ができない点です。残念ながら、調べた限りAndroidではできないようです。VM側にavahiをインストールしても、VM自身には必ずNestboxのポートフォワーディングを経由しての接続になるので、接続元のPCまで届きません。これができればIPアドレスを調べる手間も省けるのに😢。

ともあれ、これで1年以上は発症していた「開発機を異様に小さくしたい病」も満足できる形で解消できました。みなさんも、特にroot化が要らないPixel 7 (Pro)をお持ちの方は是非試してみてください！

# 📚参考にしたページ（本文で言及したものを除く）

- [Pixel 7シリーズ、非rootでLinux VM起動可能に！NestboxアプリでpKVMを利用 - AndroPlus](https://androplus.org/entry/pixel-7-no-root-linux/)
- [【PowerShell】Default Gatewayの取得方法 - buralog](https://buralog.jp/powershell-get-the-default-gateway/)
