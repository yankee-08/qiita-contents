---
title: Proxmox VE環境へのUbuntu Server 22.04のインストールからssh接続まで
tags:
  - Linux
  - Ubuntu
  - proxmox
  - 仮想化
  - ProxmoxVE
private: false
updated_at: '2024-04-02T21:54:29+09:00'
id: 495e80193070f6e70b65
organization_url_name: null
slide: false
ignorePublish: false
---

## ◇はじめに

[前回](https://qiita.com/yankee/items/1d576f7a25d6f33c6cb5)からの続きになります。

前回の記事では、ミニPCにProxmox VEのインストールから初期設定までを行いました。

https://qiita.com/yankee/items/1d576f7a25d6f33c6cb5

本記事では、次のステップとして、Proxmox VE上にVM（Ubuntu Server）のインストール、ssh接続確認までを行っています。

Proxmoxでは仮想化の方法として、仮想マシン（VM）とコンテナ（LXC）の2種類が選択できます。
両者の違いについては、公式サイトの説明や参考サイトをみてもらえればと思いますが、今回は仮想マシン（VM）を使用しています。

https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines

https://pve.proxmox.com/wiki/Linux_Container

https://knowledge.sakura.ad.jp/2108/

## ◇開発環境等

- 使用デバイス
  - Intel NUC
    - CPU：Intel Core i7-5557U
    - RAM：16G
- 使用ソフト
  - TeraTerm 5.2
  - Visual Studio Code 1.87.2
- 仮想化環境
  - Proxmox VE 8.1
- 導入VM：
  - Ubuntu Server 22.04.4

仮想マシンについては、Ubuntu ServerのLTS版であるUbuntu Server 22.04.4を選択しました。
ちなみに、Ubuntu22.04とUbuntu22.04.4のisoイメージどちらもダウンロード可能なため、どちらを使えばいいかで少し悩みましたが、以下のサイトを参考にして、今回はUbuntu22.04.4を使うことにしました。

https://pc.watch.impress.co.jp/docs/column/ubuntu/1572699.html

## ◇導入手順

進める上で、以下の2つのサイトを参考にさせていただきました。

https://pve.proxmox.com/pve-docs/chapter-qm.html

https://internet.watch.impress.co.jp/docs/column/shimizu/1551702.html

### ISOイメージのダウンロード

まず、今回インストールしたいUbuntu Server 22.04.4のISOイメージを準備します。
方法としては、

1. ISOイメージをローカルに一旦ダウンロード後にProxmoxにアップロードする
2. Proxmox上でISOイメージのダウンロードURLを指定し、直接ダウンロードする

という2種類の方法がありますが、今回は2の方法で進めていきます。

#### ダウンロードURLの確認

Ubuntuの公式サイトにいき、Ubuntu Serverをダウンロードするページに進みます。

https://ubuntu.com/download/server

このページでDOWNLOADボタンをクリックします。
なお、もし自動的にダウンロードが始まってしまった場合は一旦キャンセルします。
`download now`のリンクにマウスを合わせた状態で右クリックして、「リンクのコピー」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a7e771be-1da8-593d-27c6-4c67491126f5.png" width="600">

#### ダウンロードURL、チェックサムの入力

次に、Proxmox VEの管理メニュー上から、「local(proxmox)」-「ISOイメージ」-「URLからダウンロード」の順にクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/87f81fd3-45ff-e17d-e8a1-4b42ceb11e80.png" width="600">

URLを入力する画面が表示されるため、先ほどコピーしたURLを貼り付けし、「クエリURL」をクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/68ac94ad-88d6-77a4-d73d-7d0374984617.png" width="600">

しばらくすると、ファイル名やファイルサイズなどが表示されるため、問題ないかを確認します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/67825614-5538-c65b-9f40-59ab2a79b50f.png" width="600">

今回はダウンロード時のチェックサム検証を行っています。
先ほどのURL入力画面の右下にある「詳細設定」チェックボックスをオンにすると、入力項目が増えますので、ハッシュアルゴリズムに`SHA-256`、「証明書を検証」チェックボックスを有効にします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ff8dc3c9-6058-68f0-eb39-378a20c35da8.png" width="600">

チェックサムについては、先ほどのサイト上にある`verify your download`リンクをクリックすると、チェックサムが表示されますので、この値を入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/325a46f1-2615-7d9b-d286-8a0020d584c6.png" width="600">

:::note info
ダウンロードするISOイメージによって、チェックサムは異なります。
:::

ここまで入力が完了したら、「ダウンロード」ボタンをクリックして、しばらく（数分程度）待ちます。
ダウンロード及びチェックサムの検証が完了すると、`TASK OK`という文が出力されますので、右上の❌ボタンで画面を閉じます。
ISOイメージの一覧にダウンロードしたISOファイル名が表示されていれば問題なく、ダウンロードできています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4164dc76-39bc-6804-e2ba-e24d1b20b64e.png" width="600">

なお、チェックサムの検証が合っていない場合、`TASK ERROR: checksum mismatch`といったエラーが表示されます（試しにやってみました）。

### Ubuntu Server仮想マシンの作成

管理メニュー右上にある「VMを作成」ボタンをクリックします。
すると、仮想マシンを作成する際に必要な項目を設定するポップアップ画面が表示されるため、順次入力していきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4b728558-f9e3-5de0-248c-e170612fd566.png" width="600">

#### ［全般］

仮想マシンの「名前」等を入力します。
「名前」以外は初期値のままでも問題ありません。
自分で後で見てわかりやすい名前を設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/26303b3c-9b6d-c33e-6e88-23904d5af734.png" width="600">

#### ［OS］

インストールするOSのイメージを選択します。
今回は、すでにISOイメージをダウンロード済みのため、そのイメージを選択します。
ストレージは「local」のまま「ISOイメージ」から先ほどダウンロードしたイメージを選択します。
ゲストOSの項は初期設定のまま進めます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/169e829d-2265-a522-0569-509c8b056001.png" width="600">

#### ［システム］

システム構成に関する各種設定をする箇所になりますが、今回はすべて初期設定のままにしています。
なお、Windowsマシンを作成する場合やPCIeパススルーを行いたい場合は設定をいじる必要がありそうですので、必要に応じて変更してください。

https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_system_settings

https://qiita.com/disksystem/items/0879f379e2bbc7a08675

#### ［ディスク］

VMのディスクサイズ等を設定します。
今回は、初期設定のまま`32G`としています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/19ee9aab-aa9f-137a-8026-d65f08d8a53f.png" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4d06f235-e912-b385-6ff4-853ff32697f0.png" width="600">

#### ［CPU］

ソケット数やコア数を設定します。デフォルトは1ソケット1コアとなっていますが、以下の公式ページを参考にして合計コア数を2に変更しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/198e8f90-77ac-58fa-9bf2-f18e78db6c76.png" width="600">

https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_cpu

#### ［メモリ］

VMの使用メモリを設定します。初期値は`2048MB`ですが、今回は`4096MB`に設定しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f05c61d9-91f9-e7ab-0f94-8bd7d4ee1129.png" width="600">

#### ［ネットワーク］

ネットワークの設定についても、今回は初期設定のままにしています。
公式のドキュメントでは、「モデル」の項目は`Intel E1000`がデフォルトとなっていましたが、自分の設定では`VirtIO`が初期設定となっていたため、そちらを採用しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/18fe70e0-5510-380d-2017-18162a4b3798.png" width="600">

#### ［確認］

最後に、設定した内容を確認し、問題なければ「完了」ボタンをクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/eaec3353-836a-b0f3-b722-d9be407d258f.png" width="600">

### Ubuntu Serverのインストール

ここまで作業が終わると、Proxmox VEの管理メニュー上に先ほど作成したVMが表示されるはずです。
次に、右上の「開始」ボタンを押して、VMのインストールを行っていきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/aa09b387-611a-73e9-b286-32c61da1433a.png" width="800">

「コンソール」をクリックします。このとき、右上の「コンソール」をクリックした場合は**新規ウィンドウ**でコンソール表示、左側の「コンソール」をクリックした場合は**同一ウィンドウ内**でコンソール表示となります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/af59328b-576b-3d82-2bda-4a91fb5ad83c.png" width="800">

#### 言語・地域設定

コンソール画面を開いてしばらくすると、各種設定画面に移ります。
最初に、言語を選択する画面が表示されますが、日本語の設定が見当たらなかったため、とりあえず`English`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7899ae28-8efc-1a58-8940-b497d7063a8d.png" width="600">

#### キーボードレイアウト設定

キーボードレイアウトは、日本語キーボードの設定にします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ddbc4659-aaea-d313-a601-44cd2c8c15ac.png" width="600">

#### インストールタイプ選択

インストールするUbuntu Serverのタイプを選択します。
今回は、minimized版ではなく、通常版を選択しています。
third-party driversの項目も初期設定のままオフとしています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/938c2fa6-2533-beec-4585-62318db5daed.png" width="600">

#### ネットワーク設定（固定IP）

IPアドレスの設定に関しては、今回は固定IPを設定しています。
DHCPでの自動割り当てで問題ない場合は以下の設定は行わず、次に進んで問題ありません。
`Edit IPv4`を選択後、`Manual`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e4853b91-a3b6-2904-554d-ccffca5f3dea.png" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d45f0a6c-37cd-cfcf-81bc-d81c3d893186.png" width="600">

SubnetやAddressなど必要事項を入力していきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b1425470-2e41-e59f-de70-5377a41ed269.png" width="600">

:::note warn
`Subnet`の設定は、**サブネット**を入力する必要があります。
Windowsのネットワーク設定と同じようにサブネットマスクを設定すると以下のようなエラーが出ますので注意してください。
（自分はここでハマりました・・・）
:::

※下の画面では、仮のIPアドレスを設定しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/160afad8-ebe4-1f9f-a238-5afe930d448f.png" width="600">

正しく入力すると、エラー表示が消えます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/041f37b2-059f-968b-6315-65ea923cb13a.png" width="600">

#### プロキシ設定

プロキシの設定をしたい場合はここで設定します。
必要なければそのまま次に進みます。

#### ミラーサーバアドレスの設定

こちらも特に希望なければ初期設定のまま次に進みます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ef6116a1-30a0-2e3f-7034-06fcd45395a2.png" width="600">

:::note warn
自分の環境では、初期設定のミラーサーバだとインストール途中にエラーが発生してしまいました。
まずは初期設定で試してみて同様の問題が発生する場合は[以下の項](#インストール時にエラー発生)を試してみてください。
:::

#### ストレージ設定

ストレージの設定についても初期設定のままとしています。
ディスクの選択では、先ほど割り当てた`32GB`のディスクが選択可能なので、それを選びます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9da0880c-3d45-cf72-98c1-71e31ebbca4f.png" width="600">

ストレージ設定の確認画面が表示されるので、問題なければそのまま`Done`を選択します。
なお、初期設定では、ディスクサイズのすべてを使用せず、一定量を`free space`として確保しています。
最初からすべてのディスクをルートディレクトリに割り当てたい場合、`ubuntu-lv`のSIZEを最大までに変更します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0d42c331-bd59-834a-5600-fd55342e92e7.png" width="600">

`Done`選択後、確認画面が表示されるので、`Continue`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6b74a43d-1f13-3e61-9fde-30ed1f35580d.png" width="600">

#### ユーザー名・パスワード設定

作成したVMにログインするためのユーザー名とパスワードを設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/fb54529c-a31e-e356-13bc-57b9f822514b.png" width="600">

#### Ubnutu Proへのアップグレード設定

特に希望が無ければ、初期設定のまま`Skip for now`を選択。

#### OpenSSHのインストール

SSH接続を行うためのOpenSSHのインストールを行うかを選択します。
今回は、後程SSH接続を行うため、チェックをオンにします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7d7f6fe9-cec6-8ae8-046b-8d4c6aae41f5.png" width="600">

#### インストールしたいパッケージの選択

最初に一緒にインストールしたいパッケージがある場合は`Space`キーを押してチェックボックスをオンにします。
今回は、とりあえずチェックはしません。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c7abb03b-f067-58fd-c652-43dc8844a64b.png" width="600">

#### インストール

ここまでくると、Ubuntuのインストールが始まりますので、しばらく待ちます。
インストールが完了すると、「Reboot Now」を選んで、再起動します。
再起動すると、インストールメディアを抜いてくださいといった旨がCLI上に出力されるますので、Proxmox VEの管理メニューを操作します。

#### CD/DVDドライブの削除

物理マシン上でのインストールの場合は、インストールメディアを抜き取ればOKですが、今回はProxmox VE上の仮想マシンとなるため、仮想マシン上での対応が必要になります。
まず、今回起動したVMの「ハードウェア」を選択し、ISOイメージを割り当てている「CD/DVDドライブ」を選択します。
その状態で「削除」ボタンをクリックすれば完了です。
※最初はデタッチ的な操作項目を探しましたが、見当たらなかったため削除で対応しました

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ff4fa707-2cf8-db8e-2945-d1c1b26854b6.png" width="800">

#### 起動・ログイン

改めて、コンソール画面を立ち上げてしばらくするとログインユーザー名・パスワードを入力できる状態になる（場合によってはEnterキーを押す必要があるかも）ため、先ほど設定したユーザー名とパスワードを入力します。

```bash
SERVER_NAME login:
password:
```

ログイン成功すればコマンド入力が可能になります。

ここまででくれば、Ubuntuのインストールは完了です。

## ◇SSH接続（パスワード認証）の確認

今回はインストール時にOpenSSHをインストールしているため、別端末からのSSH接続が可能になっています。
今後の作業でSSH接続することが多いため、問題なく接続できるかを確認します。
なお、とりあえずはパスワード認証での接続を試します。

### TeraTermから

まずはTeraTermを使ってSSH接続してみます。
使ったバージョンは`TeraTerm 5.2`です。

https://github.com/TeraTermProject/teraterm

`ttermpro.exe`を起動して、接続先を以下のように指定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6e16270d-59d0-1966-b4ac-2a3a57fc48d1.png" width="600">

「Host」には今回作成した仮想マシンのIPアドレスを設定します。
ポート番号も現時点ではまだ変更していないため、初期設定の22番ポートで大丈夫です。

「OK」ボタンを押すと以下の警告画面が表示されます。
この警告画面は、今回接続するホストがTeraTerm側で持っている既知の接続先リストに含まれていないため、確認を促すものになります。
接続先側がなりすましている可能性もあるため、フィンガープリントを確認して問題なければ「Continue」で接続していきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e66681b2-c4cb-7ad7-272e-e842c3ee7d69.png" width="600">

いつもであれば、なにも考えずに「Continue」をクリックして次に進んでいるところですが、せっかくなのでちゃんとフィンガープリントを確認してみたいと思います。

#### フィンガープリントの確認

<details><summary>Skipしても可</summary>

先ほどの画像内に、`ECDSA 256`と`SHA256`という記載があるのがわかります。
そのため、今回は接続先ホストのECDSA方式の公開鍵をSHA256アルゴリズムでハッシュ化したフィンガープリントとの整合性を確認すればよいと考えられます。

なお、フィンガープリントを出力するには、`ssh-keygen -lf FILE_NAME`というコマンドを使用します。
（こちらのサイトを参考にさせていただきました）

https://maku.blog/p/m2k4j2h/

次に、Proxmox VEの管理メニューから起動したコンソール上から上記のコマンドを入力します。
なお、公開鍵は`/etc/ssh/`内に置いてあったため、そちらに合わせて以下のコマンドを実行します。

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ecdsa_key.pub
256 SHA256:4Nt*******************************+ntGsw
```

一部しか表示していませんが、実際には先ほどのTeraTerm上に表示されたフィンガープリントと一致していることが確認できました。

</details>

#### ログイン

TeraTermに戻って先ほどの警告画面で「Continue」をクリックすると、ログイン画面が表示されます。
Ubuntuのユーザー名とパスワードを入力して「OK」をクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a5a5b46e-4d46-fe46-d206-06f031e1bee6.png" width="600">

`Welcome to Ubuntu 22.04.4 LTS ~~~`といった文章が出力されればSSH接続完了です。

### Visual Studio Codeから

次に、Visual Studio Code（通称VS Code）からもSSH接続を試してみました。
ここでは、VS Codeに加えて、「Remote - SSH」という拡張機能を使用します。

https://code.visualstudio.com/

https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh

「Remote - SSH」をインストールした状態でVS Code左側アイコン内の「リモートエクスプローラー」を選択します。
次に、SSHの構成ファイルを開きます（`⚙️`ボタンではなく、`＋`ボタンで直接接続してもOKです）。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/33bbb3d2-8ba2-61f3-f857-a7150963a5ba.png" width="600">

ホストを追加するSSH構成ファイルを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9c99a714-ee69-0e91-4fd3-11335f33c22b.png" width="500">

構成ファイル上に追加したいホストの情報を追加します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/dbabb4cd-79a7-f71d-1616-975f034699e9.png" width="700">

構成ファイルを保存して、再度「リモートエクスプローラー」を表示すると、追加したリモート接続先が見えるようになるので、接続ボタンを押します（現在のウィンドウと新しいウィンドウどちらでも可）。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0b780561-51ea-a308-8db9-482c5c3a8d28.png" width="600">

OSのタイプの選択画面が出るため、`Linux`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1fde31d4-9515-1bd4-eb15-cdb75f119393.png" width="600">

次に、TeraTermと同様フィンガープリントの確認画面が表示されます（タイムアウト設定があり、画面キャプチャは取れず）。
ここで注意点ですが、先ほどとフィンガープリントが異なっています。
ホストの各公開鍵を確認したところ、ECDSAではなく、**ED25519**の公開鍵のフィンガープリントを表示しているようでした（なぜこの公開鍵が使われるかについての参考情報は見つけられませんでした）。

先ほどと同様の方法で指定するファイル名のみ変更してコマンドを実行します。

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:*******************************
```

フィンガープリントの確認でOKを選択すると、パスワード入力画面が表示されるので、先ほど同様Ubuntuのパスワードを入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/178d87fd-d6de-77d1-be11-dd41049db5a2.png" width="700">

これで一通り完了ですが、初めてのSSH接続の場合、複数回パスワード入力を求められることがあるので、その都度入力します。

ターミナル画面を表示し、各種コマンドが入力できる状態になればSSH接続完了です。

## ◇ハマったポイント

### インストール時にエラー発生

今回自分の環境では、Ubuntuの[インストール](#インストール)のタイミングでエラーが発生し、中止される問題が複数回発生しました。
エラー発生時の画面キャプチャは取り忘れたのですが、ログを追ってみたかぎりでは、インストール時にアクセスするミラーサーバに対して、

- ミラーサーバの名前解決に失敗
- ミラーサーバからのレスポンスがタイムアウト

と思われるエラーログが出力されていたため、以下の対策を試しました。

#### ＜対策①DNS設定の変更➡️改善せず❌＞

名前解決が失敗している部分があったため、最初はDNSの設定を疑いました。
そこで、[ネットワーク設定](#ネットワーク設定固定ip)の画面でDNSの設定（Name serversの項目）を変えてみました（`8.8.8.8`や`1.1.1.1`なども試してみた）が、改善しませんでした。

#### ＜対策②ミラーサーバの変更➡️改善した⭕＞

DNSサーバの変更で改善しなかったため、次にミラーサーバの変更を試しました。
初期設定では`jp.archive.ubuntu.com`のURLが設定されていたため、まずはjpを抜いた`archive.ubuntu.com`をミラーサーバとしてインストール作業を行いましたが、同様のエラーが発生しました。

次に、以下のミラーサイト一覧内のJapanのミラーサイトを試しました。

https://launchpad.net/ubuntu/+archivemirrors

とりあえず一番上の「ICSCoE(Industrial Cyber Security Center of Excellence)」の`ftp.udx.icscoe.jp`をミラーサーバに設定して再度インストール作業を行ったところ、今回は無事インストール作業が完了しました。

初期設定のミラーサーバでインストールが失敗する原因は未解決のままですが、同様のエラーが発生した場合は参考にしてもらえればと思います。

## ◇おわりに

当初の予定では、SSH接続の公開鍵認証の設定まで記事にする予定でしたが、ボリュームが多くなったため、パスワード認証でのSSH接続確認までの記事としてまとめました。
他の設定などは余裕のあるタイミングで別記事で載せたいと思います。

とりあえず、ここまででUbuntuの仮想マシンまでインストールできたので、この後はふつうのUbuntuとほぼ同じように使用できると思います。
自分もこの後、気になっているソフトのインストールを試していきたいと思います。

# 🔚END
