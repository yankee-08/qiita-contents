---
title: Postmanモックサーバーを使ってGrafana（Infinityプラグイン）の動作検証を行う
tags:
  - API
  - JSON
  - Docker
  - grafana
  - Postman
private: false
updated_at: '2024-12-12T23:13:59+09:00'
id: 69e5fb4c5b1d79606b35
organization_url_name: null
slide: false
ignorePublish: false
---

## ◇はじめに

本記事は**Postman** Advent Calendar 2024 12日目の記事になります。

https://qiita.com/advent-calendar/2024/postman

今回は、先日のLinuxアドカレの記事で構築したGrafana＋Infinityプラグインの環境とPostmanのモックサーバ機能を使ってデータの取得、可視化までを試してみました。

https://qiita.com/yankee/items/0918c7c8001831008b09

## ◇Postmanモックサーバーとは？

モックサーバーは、実際のサーバ側の構築が完了する前に、APIの動作等をエミュレートするために使われたりします。
Postmanでは、Postmanのサービス内でモックサーバーを立てることができ、HTTP接続が可能なエンドポイントが発行され、自前でサーバを準備するのに比べ、簡単に構築ができます。

なお、この辺については公式サイトやPostman社が実施しているワークショップの資料に詳しく書いてあるため、そちらも参考にしてください。

https://learning.postman.com/docs/designing-and-developing-your-api/mocking-data/setting-up-mock/

https://docs.google.com/presentation/d/1-0yoKxuWgzSmD24akdPQnSDFPYb_T7cFVP7WQyw1UaM

そのほか、参考にしたサイトも載せておきます。

https://zenn.dev/tomigie/articles/postman-mockserver

https://qiita.com/ma2shita/items/e9f120613ece63c16710

## ◇開発環境等

PostmanはWeb版を使用し、Windows PCからEdge経由で使用します。

- Postman側
  - Postman for Web：バージョン11.22.2-241209-1418
  - 使用ブラウザ：Edge
- Grafana側
  - ホストOS
    - Proxmox VE 8.3
  - VM
    - Ubuntu Server 24.04.1 LTS
    - Docker version 27.2.0
    - Docker Compose version v2.20.3
  - Grafana：grafana-oss:11.3.1

## ◇Postmanモックサーバー構築

まずは、Postman上にモックサーバーをつくっていきます。
詳細は省略する部分もあるので、適宜前述の参考サイトもご確認ください。

まずはモックサーバー用のコレクションを作成します。
今回はコレクション名を「Grafana-Mock」としています。

次に、作成したコレクションに対してモックサーバーを有効化します。
コレクションのメニューから「コレクションのモックを作成」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d19773eb-334e-27b1-87dd-5f51959c3522.png" width="400">

モックサーバー名を入力し（今回は「Grafana-Mock-Server」）、赤枠内のチェックボックスも有効にしておきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/21e250fa-678d-3501-b75d-31a0447f7e33.png" width="500">

「モックサーバーを作成」を選択してしばらくすると、URL発行画面が表示されます。
このタイミングで、右上の環境設定で、今回作成したモックサーバー名と同じ名前の環境を選択しておきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/39681dd2-a4cd-ccba-7311-4883adf06fb4.png" width="600">

## ◇HTTPリクエスト、レスポンスサンプルの追加

とりあえず1個リクエストを追加してみます。

- HTTPリクエストメソッド：`GET`
- URL：`{{url}}/hello-world`

と入力して保存します。
なお、ここでの`{{url}}`は環境変数となっており、先ほど発行されたURLが保存されています。

保存後、リクエストの設定から「サンプルを追加」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0c06fe1f-ae1f-9c8f-50cf-f0d2f47b3e66.png" width="800">

レスポンスのサンプルを入力できるようになるので、好きなレスポンスを入力します。
今回は、以下のようなJSONデータにしています。

```json
  {
    "hello": "world!!"
  }
```

データを入力して保存します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/60a76a44-0eea-0ef8-2d55-7d4608fc0f64.png" width="500">

もういちどリクエストの項目を選択し、「送信」ボタンを実行します。
先ほど設定したレスポンスサンプルが応答として出力されれば成功です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/96f23038-4401-6559-74d7-3de4cb919258.png" width="600">

## ◇Grafanaからデータ取得

先ほど追加したHTTPリクエストをGrafanaのInfinityプラグインを使って取得してみます。

GrafanaとInfinityプラグインのセットアップ手順については、別の記事でまとめてますので今回は省略します。

https://qiita.com/yankee/items/0918c7c8001831008b09

Grafanaにログインしたら、Infinityプラグインの設定画面を開きます。
まず最初にプラグインの設定内の「Base URL」にPostmanモックサーバーのURLを入力します。
（前述の`{{url}}`の部分）
その後、「New dashboard」-「Add Visualization」と選択して、ビジュアルを追加していきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/007048fd-ad0f-3fe4-6308-5cec5397e306.png" width="500">

`Data sources`にはInfinitiyを選択し、TypeやMethodなど、下の画像の通り入力していきます。
この際、`URL`欄には、`/hello-world`のみ入力することで、先ほどの「Base URL」にバインドされたURLにHTTPリクエストを発行します。
右上の表示ビジュアル設定は`Table`を選択し、`hello`と`world!!`が画面上に表示されればOKです。
もし、表示されない場合は`Refresh`を実行してみてください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/67cec987-bd62-2a7c-4690-0e4c46914a68.png" width="700">

これで、GrafanaからPostmanモックサーバーのデータを取得・可視化できました。
続いて、ほかのデータ形式も試してみます。

## ◇時系列データの可視化テスト

次に、Postmanモックサーバー側で時系列データを返すエンドポイントを作成し、それをGrafanaから取得してみます。
リクエストの設定とレスポンスのサンプルは以下のようにしています。

- HTTPリクエストメソッド：`GET`
- URL：`{{url}}/get-temperature`

```json:レスポンスサンプル
{
    "sensor": [
        {
            "timestamp": "2024-12-11T13:33:41.369941",
            "temperature": 20.23
        },
        {
            "timestamp": "2024-12-11T13:43:41.369941",
            "temperature": 20.72
        },
        {
            "timestamp": "2024-12-11T13:53:41.369941",
            "temperature": 21.18
        },
        ・
        ・
        ・
        {
            "timestamp": "2024-12-11T16:33:41.369941",
            "temperature": 22.01
        },
        {
            "timestamp": "2024-12-11T16:43:41.369941",
            "temperature": 22.03
        }
    ]
}
```

レスポンス用のJSONデータについては、10分ごとの温度データが20データ分入っています。
このデータはChatGPTで生成してもらったダミーデータです。

:::note info
当初はPostman変数を使って、`現在日時`、`現在日時の10分前`、`20分前`、・・・
といった時系列データを作りたかったのですが、うまくいかなかったため、直接日時データを入れています。
`$isoTimestamp`変数を使うことで、現在時刻の取得はできそうでしたが、そこから足したり引いたりはできないかも？
:::

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e38349a3-f48c-9e47-658e-44ca77fb7eda.png" width="650">

Postman側の準備ができたら、先ほど同様、Grafana側でビジュアルを追加します。
今回は表示ビジュアル設定は`Time series`を選択し、`timestamp`と`temperature`のColumnを入力しています。

`Refresh`して折れ線グラフが表示されれば正しくデータの取得が行えています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1cab2c71-882e-89d9-d7b6-3535dc6edfbb.png" width="800">

## ◇Postman動的変数を使った可視化テスト

最後に、Postmanモックサーバーで使える動的変数を使ったレスポンスを返してみます。
動的変数については、こちらの記事で非常に詳しく書かれています。

https://qiita.com/nagix/items/bd3b3b154d47b5a3a92e

これらの変数からいくつかピックアップし、以下のようにリクエスト設定とレスポンスサンプルを設定しました。

- HTTPリクエストメソッド：`GET`
- URL：`{{url}}/rand-user`

```json:レスポンスサンプル
{
    "users":
    [
        {
            "name": "{{$randomUserName}}",
            "mail": "{{$randomEmail}}",
            "title": "{{$randomJobTitle}}",
            "country": "{{$randomCountry}}",
            "number": "{{$randomInt}}"
        },
        {
            "name": "{{$randomUserName}}",
            "mail": "{{$randomEmail}}",
            "title": "{{$randomJobTitle}}",
            "country": "{{$randomCountry}}",
            "number": "{{$randomInt}}"
        },
        {
            "name": "{{$randomUserName}}",
            "mail": "{{$randomEmail}}",
            "title": "{{$randomJobTitle}}",
            "country": "{{$randomCountry}}",
            "number": "{{$randomInt}}"
        },
        {
            "name": "{{$randomUserName}}",
            "mail": "{{$randomEmail}}",
            "title": "{{$randomJobTitle}}",
            "country": "{{$randomCountry}}",
            "number": "{{$randomInt}}"
        },
        {
            "name": "{{$randomUserName}}",
            "mail": "{{$randomEmail}}",
            "title": "{{$randomJobTitle}}",
            "country": "{{$randomCountry}}",
            "number": "{{$randomInt}}"
        },
        {
            "name": "{{$randomUserName}}",
            "mail": "{{$randomEmail}}",
            "title": "{{$randomJobTitle}}",
            "country": "{{$randomCountry}}",
            "number": "{{$randomInt}}"
        },
        {
            "name": "{{$randomUserName}}",
            "mail": "{{$randomEmail}}",
            "title": "{{$randomJobTitle}}",
            "country": "{{$randomCountry}}",
            "number": "{{$randomInt}}"
        }
   ]
}
```

Username,Emailなどをランダムに返す変数のセットを7セット分並べています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/60404ffd-8615-0481-ac37-30cf5a8d603b.png" width="800">

Grafana側では、`Table view`を有効化し、データ全体を確認しています。
`Refresh`するごとにデータの中身が変わればOKです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d942c932-381a-c4ca-4df2-eb01d4dd6688.png" width="800">

次に、JSONデータの中からUsernameと数値データのみを取り出してバーグラフを表示してみます。
表示ビジュアル設定は`Bar gauge`を選択し、先ほどと同じようにColumnを設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b8ece8be-6a72-19e4-4dd1-373a7db0dc18.png" width="800">

このままだと、Userごとにゲージが分かれないため、`Transformation`のタブを開き、`Partition by values`という項目を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8e4eb94a-c23b-99e9-2ecf-e421c5b1f3c7.png" width="500">

`+ Select field`を選択して、`name`を選ぶとname毎のバーゲージが表示されます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b5bfb016-fe09-8659-8875-f667b4c18581.png" width="600">

このやり方については、たまたま見つけた海外の方のYoutubeを参考にしています。

https://www.youtube.com/watch?v=vAoR83g6CE4

こちらも`Refresh`するたびにNameとゲージの数値が変わればOKです。

## ◇注意事項

PostmanモックサーバーはFreeプランでも使用可能ですが、リクエスト回数の制限があります。
公式サイトの記載を見る限り、1か月あたり1,000回までとなっています。
頻繁にGrafana側からデータを読み込むと上限に達する恐れがありますので、ご注意ください。

> Usage metered on per team, per month basis.
> 1,000 requests

https://www.postman.com/pricing/

## ◇おわりに

[前回の記事](https://qiita.com/yankee/items/0918c7c8001831008b09)で構築したGrafanaを使って、Postmanモックサーバーからデータが取得できることを確認できました。
Postmanモックサーバーの一般的な使い方とは少しズレるかもしれませんが、こんな使い方もできるということで参考になれば幸いです。

# 🔚END
