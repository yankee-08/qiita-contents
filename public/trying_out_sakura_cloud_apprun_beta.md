---
title: さくらのクラウド『AppRun β版』をサクッと試してみる
tags:
  - さくらのクラウド
  - AppRun
  - Docker
  - grafana
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

先日さくらインターネットさんから、アプリケーション実行基盤「AppRun β版」のトライアルを開始する旨のアナウンスがありました。

https://www.sakura.ad.jp/corporate/information/newsreleases/2025/02/04/1968218382/

コンテナ実行基盤サービスの1種のようですが、他のクラウドも含めて触ったことがなかったので、試してみました。
なお、現在提供されているβ版は**無料**で利用可能です。

## ◇構築環境

公式のマニュアルにあるクイックスタートをそのまま試してもいいのですが、今回は以前ローカルで作成したコンテナを使って試してみました。

https://manual.sakura.ad.jp/cloud/apprun/getting_started.html#

今回は前に作った、Grafana＋Infinityプラグインを動かすコンテナを使います。

当時書いた記事はこちらです。

https://qiita.com/yankee/items/0918c7c8001831008b09

## ◇導入手順

基本的には、公式のチュートリアルに沿って進んでいきます。

https://manual.sakura.ad.jp/cloud/apprun/getting_started.html

### 事前準備（アカウント、クレジットカード登録）

アカウントを持っていない場合、まずはアカウントの登録が必要になります。
自分は以前にさくらのクラウドシェルを試す際に会員登録をしていたため、今回はそちらのアカウントを使用しています。

https://zenn.dev/yankee/articles/try_sakura_cloud_shell

アカウント登録の手順は公式サイトに手順が載っています。

https://manual.sakura.ad.jp/cloud/payment/signup.html#id13

また、今回使用するAppRun β版は無料で使えますが、クレジットカードの登録は必要となります。
※クレジットカード登録していないと、コンテナレジストリのサービスアクセス時にエラーとなる

こちらも[公式のマニュアル](https://manual.sakura.ad.jp/cloud/payment/signup.html#creditcard-registration)に従って登録します。

### コンテナレジストリのリポジトリ作成

まず、AppRunで実行するためのコンテナイメージを格納するための準備をします。
AppRunで実行するコンテナイメージは、さくらのクラウド内のコンテナレジストリに配置する必要があります。

こちらも基本的に公式の手順に従って進めていきます。

https://manual.sakura.ad.jp/cloud/appliance/container-registry/#

こちらの記事も参考になったので、共有します。

https://qiita.com/zembutsu/items/4c5e58918249c1eae8c8

まず、コンテナレジストリの画面から追加ボタンを押します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d3b6bf59-2275-462c-a77f-27c65f4c4896.png" width="700">

名前などの必要事項を入力していきます。
なお、コンテナレジストリ名は他の人が使っている名称は使えません。

「公開設定」は、今回は認証なしで`Push & Pull`ができる設定にしています。
ほかの方にアクセスされたくない場合は、`非公開`設定を選択してください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f35e7993-53cf-4c8e-989e-13d70790a192.png" width="700">

確認画面が出るので、「作成」ボタンを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d217e5b6-4783-494f-b878-9bb7af0536f3.png" width="300">

コンテナレジストリ一覧表示画面に戻り、作成したコンテナレジストリのステータスが「成功」になっていればOKです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c06542ca-90d2-4ca1-adce-aabb02671383.png" width="800">

次に公式のチュートリアルでは、ユーザの作成を行っていますが、今回は誰でも`Push & Pull`ができる公開設定にしているため、ユーザ作成はスキップします。

https://manual.sakura.ad.jp/cloud/appliance/container-registry/#id8

:::note warn
今回作成したコンテナレジストリは既に削除しています。
認証なしの設定にした場合、誰でもイメージをPushできる状態になるので、取扱は注意してください。
:::

### コンテナイメージの準備＆レジストリへの格納

今回は以前に別記事で作成したコンテナを使ってコンテナイメージを作成します。
コンテナからイメージを作成する際のDockerコマンドは、`docker commit`コマンドを使います。

まず、イメージ化したいコンテナIDを調べるため、`docker ps`コマンドを実行します。

```bash
docker ps
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

調べたIDに対して、`docker commit`コマンドを実行します。

なお、その際に[公式](https://manual.sakura.ad.jp/cloud/appliance/container-registry/#push)にも記載されているように、イメージ名の命名方法にルールがありますので、ご注意ください。

> コンテナレジストリにpushするイメージ名は [レジストリ名].sakruacr.jp/[任意のイメージ名]:[タグ] である必要があります。pushするイメージは docker tag コマンドでイメージ名を変更します。

```bash
docker commit {{CONTAINER_ID}} yankee-grafana-infinity.sakuracr.jp/my-grafana:latest
```

コマンド実行が完了したら、イメージ一覧表示コマンドを実行してイメージが追加されていることを確認します。

```bash
$ docker image ls
REPOSITORY                                       TAG       IMAGE ID       CREATED        SIZE
yankee-grafana-infinity.sakuracr.jp/my-grafana   latest    {{IMAGE_ID}}   2 days ago     485MB
```

`docker commit`コマンドの詳細は公式ドキュメントをご覧ください。

https://docs.docker.jp/engine/reference/commandline/commit.html

コンテナイメージができたら、`docker push`コマンドでコンテナレジストリにpushします。

```bash
$ docker push yankee-grafana-infinity.sakuracr.jp/my-grafana
Using default tag: latest
The push refers to repository [yankee-grafana-infinity.sakuracr.jp/my-grafana]
・・・
latest: digest: sha256:xxxx size: 2622
```

:::note info
上記コマンドでも動作しましたが、本来は`イメージ名:latest`という形でタグ名も記載するようでした。
:::

今回は、公開設定にしているため、上記コマンドだけでOKですが、非公開設定にした場合は[公式ドキュメント](https://manual.sakura.ad.jp/cloud/appliance/container-registry/#id10)に沿って、`docker login`コマンドを先に実行する必要があります。

ここまでで、コンテナレジストリへのイメージの格納は完了です。
なお、コンテナレジストリのメニュー上からはイメージが格納されたか判断がつかないため、CLI上のログで判断してます。

### AppRun β上にアプリケーション作成

いよいよAppRunでアプリケーションを作成していきます。
この部分も公式ドキュメントにしたがって作業していきます。

https://manual.sakura.ad.jp/cloud/apprun/getting_started.html#id9

まず、「アプリケーションの作成」をクリックし、各種設定を入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a2bb148f-77c8-478a-b76a-6df0f73756fb.png" width="600">

アプリケーション名以外は公式と同じ設定としました。

:::note alert
[後述](#起動したがアクセスできない)しますが、ポート設定を間違っており、このままではアクセスできませんでした。
:::

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/bccb10fa-395e-4c18-bce7-c14016d83344.png" width="800">

コンテナの構成情報の設定画面では、先ほどコンテナレジストリに格納したコンテナイメージを指定します。
コンテナレジストリアクセス設定は**非公開**設定にしている場合は入力が必要です。

リソース設定はメモリーのみデフォルトの512MiBから1GiBに変更しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8836e106-278c-4c7d-8493-84eda907d025.png" width="800">

パケットフィルター設定はデフォルトはオフですが、今回は念のため有効にして、送信元IPアドレスを制限しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2f337d91-78ce-4bcc-a867-6155a577726b.png" width="800">

ここまできたら、「作成する」をクリックすればアプリが構成されて起動するはずです。

### 起動したが、アクセスできない

しばらくすると、公開URLが表示されたため、そちらにアクセスしましたがGrafanaのログイン画面に入れませんでした。

改めて自分が登録したコンテナイメージの構成ファイル（compose.yaml）を確認すると、開放ポート番号が3000となっており、AppRun側で設定していた8080と食い違っていたことが原因でした。

```yaml:compose.yaml
services:
  grafana:
    image: grafana/grafana-oss:11.3.1
    container_name: grafana-11
    restart: unless-stopped # 自動再起動ポリシー
    environment:
      - GF_INSTALL_PLUGINS=yesoreyeram-infinity-datasource # Infinityプラグインをインストール
    ports:
      - "3000:3000" # ホスト:コンテナ
    volumes:
      - grafana_storage:/var/lib/grafana # 永続化データの保存
volumes:
  grafana_storage: {}
```

そこで、アプリケーション構成の変更画面から、ポート番号を3000に変更しました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/51762637-c7f4-4f29-9ced-63c949e0513f.png" width="800">

改めてアプリケーション作成を行い、しばらくするとステータスが正常になるため、公開URLにアクセスします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9694a28d-c696-4b5a-98c2-43c9696a2a6c.png" width="600">

Grafanaのログイン画面が出れば成功です。

### Grafanaへのログイン確認＆Infinityプラグインのインストール確認

最後に念のため、GrafanaにログインしてInfinityプラグインがインストールされているかまで確認します。

Grafanaログイン画面でUsername、Passwordともに`admin`でログインしてみます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c83fe763-40b3-4fde-9911-9d117cd963df.png" width="400">

Grafanaにログイン出来たら、Infinityプラグインがインストールされているかを確認します。
トップ画面の左側メニューから`Connections`を選択し、検索窓から`Infinity`で検索します。
Infinityプラグインが表示されたら選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/03d605a8-1975-40ec-b18a-77d2016967f1.png" width="300">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e9fc1139-ac88-4c39-b2e1-00b7b22ea6c4.png" width="400">

Infinityプラグインの画面を開いて、右上に`Uninstall`ボタンが表示されていれば、Infinityプラグインが正しくインストールされています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0f3d3bd4-9cb2-423a-ac76-79d125da7059.png" width="800">

## ◇おわりに

コンテナ（アプリケーション）実行基盤サービスを初めて使ってみましたが、思ったり簡単に使うことができました。
どちらかというと、コンテナレジストリの方（dockerコマンドの使い方含む）が慣れるのに少し勉強が必要かなという感じです。
特にHTTPS対応の設定を自分で行う必要がないのが非常に助かります！
他のクラウドサービスも含めて、この辺のサービス（PaaS,CaaS,FaaS）はあまり触れていない、そもそも各社が提供しているサービスの違いがあまりわかっていないため、今後もう少し触ってみたいと思います。

最後に、今回使用した、さくらのクラウド AppRun β版ではフィードバックを募集しているので、自分なりに気になった点をフィードバックしたいと思います。

# 🔚END
