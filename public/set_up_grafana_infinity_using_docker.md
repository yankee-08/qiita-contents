---
title: Dockerを使ったGrafana＋Infinityのセットアップ手順
tags:
  - Linux
  - Ubuntu
  - Docker
  - proxmox
  - grafana
private: false
updated_at: '2024-12-11T01:02:28+09:00'
id: 0918c7c8001831008b09
organization_url_name: null
slide: false
ignorePublish: false
---

## ◇はじめに

本記事は**Linux** Advent Calendar 2024 11日目の記事になります。

https://qiita.com/advent-calendar/2024/linux

今回は、ローカルPC（ミニPC）上にGrafanaとInfinityプラグインを構築する手順を備忘録としてまとめました。

## ◇背景

先日開催された、**GrafanaMeetupJapan #3**をオンライン視聴させていただきました。

https://grafana-meetup-japan.connpass.com/event/330796/

このイベントの中で**Grafanaプラグイン探訪**という内容でLTをしていました。
資料も公開してくれています。

https://mk.sios.jp/GrafanaMeetup_SIOS

このLT内でGrafanaのプラグインの1つであるInfinityプラグインについて紹介されていました。

このプラグインを使うことで、Grafanaにデータソースとして用意されていないデータでも連携（取得）が可能と思い、今回試してみました。

## ◇開発環境等

今回はProxmox上にUbuntuのVMをインストールし、そのUbuntu上でDockerを動かしています。

- 使用デバイス
  - Intel NUC
    - CPU：Intel Core i7-5557U
    - RAM：16G
- ホストOS
  - Proxmox VE 8.1
- VM
  - Ubuntu Server 24.04.1 LTS
  - Docker version 27.2.0
  - Docker Compose version v2.20.3
- Grafana：grafana-oss:11.3.1

## ◇導入手順

今回は、Proxmox上へのUbuntuのインストール手順は割愛します。
なお、以前にProxmox VE環境へUbuntu Server 22.04をインストールする手順を記事にしています。

今回、自分が試した限りでは、Ubuntu 24.04でもインストール手順はそこまで差がなかったため、必要に応じてご参照ください。

https://qiita.com/yankee/items/495e80193070f6e70b65

### UbuntuへのDockerのインストール

今回はUbuntuインストール時に追加パッケージのインストール選択画面上からDockerを選択してインストールしました。
画面キャプチャを取り忘れたため、Ubuntu 22.04インストール時の画像を載せておきます。
多少パッケージのリストは異なるかもしれませんが、`docker`のパッケージはこちらにも載っています。
`docker`のチェックボックスをONにすることで、Ubuntuインストール時にDockerも自動でインストールされます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c7abb03b-f067-58fd-c652-43dc8844a64b.png" width="600">

なお、よくあるやり方として、下のサイトにあるような、aptコマンドを使ったパッケージのインストール方法でも問題ありません。（むしろこっちのほうが一般的と思われる）

https://matsuand.github.io/docs.docker.jp.onthefly/engine/install/ubuntu/

### compose.yamlの作成

次にコンテナを起動するための準備をします。
今回は、Docker Composeを使います。

https://docs.docker.jp/compose/toc.html

Ubuntu内で分かりやすい場所にディレクトリを作成し、その中に`compose.yaml`のファイルを作成します。
ファイルの中身は下のようになっています。
今回は、Infinityプラグインをコンテナ作成時に一緒にインストールしています。
※公式ドキュメントにも記載がありますが、Grafanaインストール後にGUIからInfinityプラグインをインストールする方法でもOKです

```yaml
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

上記コードは以下のサイトを参考にし、カット＆トライで試してとりあえず動いたものになります。

https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/#example-1

https://grafana.com/docs/plugins/yesoreyeram-infinity-datasource/latest/setup/installation/#install-using-docker

### コンテナの起動＆Grafanaへのログイン

以下コマンドでコンテナを起動します。
ちなみに、自分の環境ではVisual Studio CodeにDockerの拡張機能を入れて使用しています。

```bash
docker compose up -d
```

しばらくしてコンテナが起動したら、ブラウザでGrafanaのサーバにアクセスします。

今回はポート番号を3000に設定しているため、
`http://{{host_ip}}:3000/`
にアクセスします。
※`{{host_ip}}`は各自のホスト（今回の場合はUbuntuのVM）のIPアドレスを入力

URLにアクセスすると、Grafanaのログイン画面が表示されます。
（セキュリティ保護がされていない旨の警告が出る場合があります）

初期状態では、Username、Passwordともに`admin`でログインできます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5ff99431-9c83-7faa-9a55-66faebb14dd5.png" width="500">

ログインすると、パスワード変更の画面に遷移するので、適切なパスワードを設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5730b450-3d22-65d5-42d8-cf6e3275cff6.png" width="300">

### Infinityプラグインのインストール確認

Grafanaにログイン出来たら、Infinityプラグインがインストールされているかを確認します。
トップ画面の左側メニューから`Connections`を選択し、検索窓から`Infinity`で検索します。
Infinityプラグインが表示されたら選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/fc246e72-9beb-f480-48cf-9821c96e616f.png" width="700">

Infinityプラグインの画面を開いて、右上に`Uninstall`ボタンが表示されていれば、Infinityプラグインが既にインストールされているため、そのままボタンは押さずにトップ画面に戻ります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/29376159-617c-434c-92ae-527ff86b8715.png" width="800">

### ダッシュボードの作成

最後に、Infinityプラグインを使って実際にダッシュボードを作ってみます。
まず、左側メニューから`Data sources`を選択し、Infinityプラグインの右側にある`Build a dashboard`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/3129d49b-4fb3-1fc6-fa2e-2a1d175cccaf.png" width="700">

空のダッシュボード画面が表示されるので、`Add visualization`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e78830a6-77d6-3f91-46b3-466d23a46d9e.png" width="500">

使用するデータソースの選択画面が表示されるので、Infinitiyを選びます。
ここまでの手順どおりであれば、それ以外のデータソースは選択肢に出てこないはずです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/bcc90c1c-6538-bae0-0820-f1245af093d7.png" width="600">

パネル作成画面が表示されますので、項目を入力していきます。
入力する項目は主に赤枠の中になりますが、とりあえず動作確認のため、参考データが取得できるサイトを探してみます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/08ccb3f3-a76c-fb26-b6af-d2baa01629bc.png" width="700">

ここで改めて公式ドキュメントを確認したところ、確認用のデータ取得用エンドポイントが用意されているため、これを使ってパネルを作成してみます。
なお、今回はJSONデータのみ試しています。

https://grafana.com/docs/plugins/yesoreyeram-infinity-datasource/latest/

#### JSON Data without time field

最初に試したエンドポイントはこちらです。

https://grafana.com/docs/plugins/yesoreyeram-infinity-datasource/latest/json/#json-data-without-time-field

URL:`https://gist.githubusercontent.com/yesoreyeram/2433ce69862f452b9d0460c947ee191f/raw/f8200a62b68a096792578efd5e3c72fdc5d99d98/population.json`

このエンドポイントで取得できるデータは、タイムスタンプが入っていないJSON形式のデータになります。

```json
[
  { "country": "india", "population": 300 },
  { "country": "usa", "population": 200 },
  { "country": "uk", "population": 150 },
  { "country": "china", "population": 400 }
]
```

公式ドキュメントの入力項目を確認しながらパネルの項目に入力していきます。

まず最初に画面右上のメニューから表示するビジュアルを`Gauge`に変更します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7f35d096-8a4f-ddac-c2f9-08cc7969fe67.png" width="700">

次に、公式サイトを参考にしながら、

- Type：`JSON`
- Source：`URL`
- Format：`Time Series`
- Method：`GET`
- URL：`https://gist.githubusercontent.com/yesoreyeram/2433ce69862f452b9d0460c947ee191f/raw/f8200a62b68a096792578efd5e3c72fdc5d99d98/population.json`

と入力します。
FormatがTime Seriesになるので、注意してください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/18ed8d4b-39a0-efcd-f4f7-289304881510.png" width="700">

次に、画面を下にスクロールし、`Add Column`を選択します。
すると、Columnの追加画面が表示されるので、下の画像の通り入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/30649ffb-6bdd-ddd9-e5ab-b97275a2ebee.png" width="700">

ここまで、入力出来たら画面上部の`Refresh`を選択して、各国のデータがゲージ形式で表示されれば成功です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f97ea852-9bfe-75ab-6f08-e8a16f0153a3.png" width="600">

ちなみに、ビジュアルを`Stat`に変更するとこんな感じになります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8006ebc2-df2a-064f-851f-ad716ef09db4.png" width="700">

#### Accessing nested properties of JSON data

最後にもう一つ、こちらのエンドポイントも試してみました。

https://grafana.com/docs/plugins/yesoreyeram-infinity-datasource/latest/json/#accessing-nested-properties-of-json-data

URL:`https://thingspeak.com/channels/38629/feed.json`

このエンドポイントでは、時系列データも入っています。
実際に取得できるデータはかなり多いので、一部抜粋したものを載せます。

```json
{
    "channel": {
        "id": 38629,
        "name": "Traffic Monitor",
        "description": "Traffic Monitor showing density of cars detected",
        "latitude": "42.28",
        "longitude": "-71.35",
        "field1": "Density of Westbound Cars",
        "field2": "Density of Eastbound Cars",
        "created_at": "2015-05-19T20:14:03Z",
        "updated_at": "2019-07-24T20:12:00Z",
        "last_entry_id": 18055445
    },
    "feeds": [
        {
            "created_at": "2024-12-10T14:40:08Z",
            "entry_id": 18055346,
            "field1": "21.000000",
            "field2": "34.000000"
        },
        {
            "created_at": "2024-12-10T14:40:24Z",
            "entry_id": 18055347,
            "field1": "23.000000",
            "field2": "22.000000"
        },
        ・
        ・
        ・
        {
            "created_at": "2024-12-10T15:05:44Z",
            "entry_id": 18055445,
            "field1": "9.000000",
            "field2": "17.000000"
        }
    ]
}
```

ビジュアルの追加を選択して、先ほどと同じ流れで項目を入力していきます。
今回はビジュアルを`Time series`に設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/371ff4e2-62ed-a29e-6226-9e0f4628b4d5.png" width="700">


<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/62730f6b-c573-a3e5-0c03-8f56968e5925.png" width="500">

各項目は、

- Type：`JSON`
- Source：`URL`
- Format：`Table`
- Method：`GET`
- URL：`https://thingspeak.com/channels/38629/feed.json`

と入力します。
今回はFormatが**Table**になります。
（Formatの選び方がまだよくわかっていない・・・）

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0ba70069-b862-5a17-aa0e-d77944a56794.png" width="700">

次に、`Rows/Root`の項目に`feeds`と入力し、Columnも下の画像のように入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8afa384b-f9e2-69f7-f886-8e7975f2c841.png" width="700">

ここで入力した`feeds`についてですが、元のJSONデータでは、`feeds`キーのvalueとして時系列データが入っているため、その時系列データのみを取得するための設定と思われます。

:::note warn
実際のところ、`feeds`を入力しなくてもグラフは問題なく表示できました。
時系列データの部分のみをGrafana側がうまく抜き出してくれていると思われますが、詳細は現状不明です。
:::

最後に先ほど同様、画面上部の`Refresh`を選択して、各国のデータがゲージ形式で表示されれば成功です。
その際、Time Rangeを`30minutes`か`15minutes`くらいにしておくと、グラフ全体が表示できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0d54973c-9665-e6bd-f7f3-1ba133e82056.png" width="700">

ここまでで、サンプルデータを使ってGrafanaでデータを可視化することができました。

## ◇おわりに

まずは環境構築として、自宅PC上にGrafanaとInfinityプラグインを導入できました。
次のステップとして、実際のデータを使用してデータの可視化を試してみたいと思います。
アドベントカレンダー期間中にどこまでできるか・・・

# 🔚END
