---
title: ミニPC（Intel NUC）へのProxmox VEのインストールから初期設定まで
tags:
  - Linux
  - サーバー
  - proxmox
  - 仮想化
  - ProxmoxVE
private: false
updated_at: '2024-04-02T22:03:18+09:00'
id: 1d576f7a25d6f33c6cb5
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

自宅サーバ用に以前ミニPC（Intel NUC）を中古で購入したのですが、買った後ほったらかしになってしまっていたので、今回改めて自宅サーバとして稼働してみました。
仮想化環境としては、VMware ESXiやHyper-Vなども有名ですが、今回はProxmox VEを採用しています。
本記事では、ミニPC上にProxmox VEを導入するところまでを説明しています。
それ以降のVM（Ubuntu）のインストール、初期設定などは出来次第、別記事にまとめる予定です。

（2024/04/02追記）
[続きの記事](https://qiita.com/yankee/items/495e80193070f6e70b65)を書きました。

https://qiita.com/yankee/items/495e80193070f6e70b65

Proxmoxの概要については、この辺のサイトが参考になるかと思います。

https://www.proxmox.com/en/proxmox-virtual-environment/overview

https://proxmox.classact.co.jp/proxmox_ve/#whats

## ◇開発環境等

- 使用デバイス
  - Intel NUC
    - CPU：Intel Core i7-5557U
    - RAM：16G
- 仮想化環境
  - Proxmox VE 8.1
- 使用ツール：
  - Raspberry Pi Imager

## ◇導入手順

進める上で、以下の2つのサイトを参考にさせていただきました。

https://pve.proxmox.com/pve-docs/chapter-pve-installation.html

https://internet.watch.impress.co.jp/docs/column/shimizu/1551702.html

### Proxmoxのダウンロード・インストール用USBの作成

公式サイトから、ISOファイルをダウンロードします。
今回は最新バージョンの8.1をダウンロードしています。

https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso

ダウンロードが完了したらisoファイルをインストール用USBに書きこみます。
書き込み用のソフトとしては、[参考サイト](https://internet.watch.impress.co.jp/docs/column/shimizu/1551702.html)同様、Raspberry Pi Imagerを使用しました。

https://www.raspberrypi.com/software/

今回は使用していませんが、Rufusなどのソフトでも問題ないと思います。

Raspberry Piデバイスは指定せず、OSをカスタム設定からダウンロードしたISOファイルを選択します。
書き込みが完了したら、USBメモリを抜いて完了です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/71e43f3f-b810-7f5a-1f2a-598f2bf1d877.png" width="500">

### Proxmoxのインストール

USBメモリをミニPCに挿した状態で、起動します。このとき、初期設定用にミニPCにはマウスとキーボードを接続しておく必要があります。
また、ミニPCにLANケーブルなどを接続してIPアドレスが取得できる状態にしておくと、Local設定などが一部自動検出されるので、可能であれば接続しておく方がインストールが楽になります。

なお、今回はスクリーンショットが撮れるように、USB-HDMI変換アダプタを使用しています。
`ミニPC <-> HDMIケーブル <-> USB-HDMI変換アダプタ <-> PC USB入力`という順に接続し、PCのカメラアプリを使ってミニPCの画面を表示しています。

#### インストールするタイプの選択

起動すると、インストール確認画面が表示されるので、インストールするタイプを選択します。
今回は、`Proxmox VE(Graphical)`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/eb65005e-969f-06e3-3497-4a989d7d3587.png" width="500">

#### 利用許諾の確認

利用許諾を確認し、問題なければ同意して次に進みます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/13dfa51d-7511-fb34-0235-437ebfb18b14.png" width="500">

#### インストール先の指定

インストール先のディスクを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8d5f7cea-e276-3e78-98b2-229cd601ce74.png" width="500">

#### 地域設定・タイムゾーン設定

国、タイムゾーン、キーボード配列の設定を行います。
前述したように、インターネットに接続できる状態であればこれらの設定は自動で入力されますが、
オフラインの場合、手動で入力する必要があります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e7a1a355-4d79-69ae-0836-db57b5a8314a.png" width="500">

#### パスワード・メールアドレス設定

パスワードとメールアドレスを設定します。
下の画像では、メールアドレスがBLANKとなっていますが、未入力だとエラーで次に進めません。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a2e1d412-2aa7-6ae3-9421-f9ff95344e0d.png" width="500">

#### ネットワーク設定

使用するネットワークインターフェース、IPアドレス等を設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/97d215b8-af9b-6261-f468-a1bfa2609d17.png" width="500">

#### インストール

インストール作業が完了するまでしばらく待ちます。
自分の環境では、10分掛からないくらいでした。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6047cde3-dcb6-d289-e45f-d3bef76875cf.png" width="500">

#### 再起動

インストールが完了したら、USBを抜いて、再起動します。
※USBを挿したまま再起動するとまたインストール作業用の画面が表示されるので注意
起動する環境を選択する画面が表示されたら、`Proxmox VE GNU/Linux`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5d3ed9e1-8999-30a3-532e-db378017e855.png" width="500">

起動すると、シェルログイン画面に加えて、Web管理画面に入るためのURLが以下のように記載されるため、ブラウザからアクセスします。
`https://xx.xx.xx.xx:8006`

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ea6b42b6-ef2f-d617-97d4-1d6c01942270.png" width="500">

### Proxmox初期設定

ブラウザでWeb管理画面に入るとログイン画面が表示されます。
`User name`を`root`とし、`Password`は先ほど入力したものを入力します。
`Realm`は`Linux PAM`を選択し、言語は必要に応じて日本語を選択します。
なお、言語設定を変更すると、他の入力項目がクリアされるため、言語設定を変える場合は一番最初に実施してください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/35c464eb-75d1-41f2-3877-d4e467bc9814.png" width="500">

#### リポジトリの変更

ログインが完了したら、まずリポジトリの変更を行います。
初期設定では、有償版リポジトリが選択されていますが、今回は有償版の契約をしないので、無償版のリポジトリに切り替えます。

ブラウザ上で、`proxmox - アップデート - リポジトリ`を選択し、APTリポジトリの追加ボタンを押します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/3d47fa38-c26c-1227-cc7d-a3253767ca12.png" width="500">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d4b9c7fb-0662-4ba8-a004-fc10046a8d90.png" width="500">

選択肢の中から、`No-Subscription`を選択して追加します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0a505d81-5797-46e5-5349-c3309ca3597c.png" width="500">

リポジトリに追加が完了したら、有償版のリポジトリ2つを`Disable`にします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/cbbf2f21-57b7-58fe-59c9-be39eab384cc.png" width="500">

これでリポジトリにアクセスできるようになったので、改めてアップデートを行います。
`アップデート`項目内で`再表示`ボタンをクリックして、アップデート可能なパッケージを確認します。
その後、`アップグレード`ボタンをクリックすると、別ウィンドウが表示されてコマンドライン上でアップグレードが始まります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5fed79c4-bce9-241d-2231-3d370970928c.png" width="500">

最初に、アップグレードの確認が表示されるので、`Y`を入力してEnterします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b756fe94-ac5c-5d17-a2ba-850d93c617af.png" width="500">

#### MFA（TOTP）の有効化

Proxmoxでは、MFAがGUI上から簡単に設定できるので、試してみました。
`アクセス権限 - 2要素 - 追加`を選択すると、認証の種類が選択できます。
今回はTOTPを選択します。

TOTPの概要は、以下のサイトが参考になりました。

https://www.istc.kobe-u.ac.jp/glossary/TOTP/

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2215f3e5-69b7-d6d0-06db-bf9ab977da40.png" width="500">

登録画面が表示されますので、表示されたQRコードを対応アプリで読み込みます。
今回、アプリはMicrosoft Authenticatorを使用しました。

https://apps.apple.com/jp/app/microsoft-authenticator/id983156458

QRコードが正しく読み込めると、アプリ上に数桁のコードが表示されるので、その数値をコードを検証の部分に入力します。
最後に「追加」ボタンをクリックすれば完了です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a43de68c-8d87-0638-866a-66b8e6882bf8.png" width="500">

MFAが正しく動作するかを確認するため、一旦ログアウトして再度ログインを試します。
パスワード認証後、以下の画面に遷移すればMFAが有効化されています。
先ほど同様Authenticatorアプリで表示されたコードを入力すればログインできます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7569f97f-5a78-d5e3-a0c8-d307627a553a.png" width="500">

## ◇おわりに

何番煎じか分かりませんが、仮想化環境として、Proxmox VEのインストールと初期設定までを行いました。
次のステップとして、VM（Virtual Machine）としてUbuntu Serverの導入と初期設定を試したいと思います。

（2024/04/02追記）
Ubuntu Serverの導入については、こちらで記事にしました。

https://qiita.com/yankee/items/495e80193070f6e70b65

# 🔚END
