---
title: iPhoneからJSONをPOSTする備忘録
tags:
  - iPhone
  - API
  - JSON
  - ショートカット
  - REST-API
private: false
updated_at: '2024-11-30T22:50:55+09:00'
id: 356c0216347650af67e0
organization_url_name: null
slide: false
ignorePublish: false
---

## ◇はじめに

自動化の処理を行う際に、自分が持ってるスマホをトリガーにして処理を開始したい場面があります。

以前は、[「Scriptable」](https://scriptable.app/)というアプリを使ってHTTPのリクエストを行ったりしてたのですが（以下の記事）、ショートカットアプリ内でHTTPリクエストがJSON形式で行えることを知ったため（結構前からできる模様）、備忘録も兼ねて記事にします。

https://qiita.com/yankee/items/afffe089fcc049cdb61a

## ◇環境等

- 使用デバイス
  - iPhone SE3
    - OS：iOS17.5.1
- 使用アプリ
  - ショートカットアプリ
- 使用ツール・サービス
  - webhook.site
  - Azure Logic Apps
  - Postman Flows

## ◇実装手順

### ショートカットアプリ

今回は、iPhoneにプリインストールされている「ショートカットアプリ」を使用します。

https://support.apple.com/ja-jp/guide/shortcuts/welcome/ios

このアプリでは、複数の手順をひとつにまとめた、文字通りショートカットを作成できます。
また、詳細は後述しますが、ショートカットアプリ内のオートメーションを使うと、特定の条件をトリガとして、作成したショートカットを自動実行することも可能です。

### 新規ショートカットの作成

まず、ショートカットアプリを開き、右上の`+`ボタンを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/378ba314-3592-e6a3-c004-df9f5d5a2d11.jpeg" width="300">

次に、ショートカットの名称やアイコンを分かりやすいように変更します。
今回はそのまま「JSONポスト」という名称に設定しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/643ac6bc-98c7-dc4d-18d6-b99874dcff2b.jpeg" width="300">

アクションを追加していくのですが、今回はカテゴリの中から「Web」を選択します。
カテゴリ内にある「URLの内容を取得」アクションを選択します。
※検索から探してもOK

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f6b01d8c-0990-5efa-e527-d418641ed35b.jpeg" width="400">

アクションの名称だけで判断すると、特定のURLからデータをとるためのアクションと勘違いしやすいですが、アクションの詳細に書いてあるように、HTTPメソッドも指定可能で、APIリクエストの実行にも使えます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1e073bf3-4e07-2df1-9bfd-e50c7fe16a93.jpeg" width="400">

アクションを選択すると、URLや方法（HTTPメソッド）、ヘッダなどを入力できるようになります。
URLの箇所はとりあえずスキップし、方法を`GET`から`POST`に変更します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/161e424c-c9ee-16ca-68f7-b7dbebeadff0.jpeg" width="400">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/047f8d0d-367c-f8af-d91b-7775bf9bff75.jpeg" width="300">

`POST`を選択すると、本文（body）を入力する項目が追加されます。要求方法は初期設定がJSONになってるため、このままにしておきます。
なお、JSON以外に「フォーム」と「ファイル」という選択肢がありました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4bcea93e-a26e-6610-13da-696d0644c6f6.jpeg" width="400">

JSONを選択時、本文には以下の5つのデータ型が選択できます。とりあえず今回はそれぞれの項目を入力して実際にPOSTしてみます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a1b6e141-ca0e-9a7d-a5c6-63d84c01d0f1.jpeg" width="400">

こちらが実際に入力したデータです。
上から順に、

- テキスト：`userName`
- 数字：`userId`
- 配列：`tagList`
- 辞書：`SNS`
- ブール値：`isDeleted`

の項目を選んで入力しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/39a9f575-0a53-d49e-0a15-a8ec973d133d.jpeg" width="400">

#### テキスト

テキスト型を選択した場合、キーとテキストを入力する項目が表示されるため、適宜入力します。
なお、通常JSONではキーにはダブルコーテーションをつけ、値（value）にも、文字列型の場合はダブルコーテーションで括る必要がありますが、ショートカットアプリ内で入力する際はダブルコーテーションは不要でした。

#### 数字

数字型もテキスト型とほぼ同じです。キーと数字を入力する項目が表示されるため、適宜入力します。

#### 配列

配列型を選択した場合、キーは同じように入力しますが、値の方には、項目を追加してそれぞれに対して、テキストや数字を入力していきます。
今回は、テキスト型の要素を4つ入力しています。
なお、配列の各要素にも、更に配列や辞書の選択肢があるため、ネストすることもできそうです（今回は試してません）。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/22959c96-4e59-384c-bbdd-044532e180f6.jpeg" width="400">

#### 辞書

辞書型はネスト構造のJSONを作成する際に使用します。
今回は、`SNS`をキー名とし、その中に各SNSの項目を入力しています。

配列同様、各要素内には配列や辞書の選択肢があるため、更にネストすることもできそうです（今回は試してません）。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b54893b0-6d7a-47cd-b95a-28d9dff69aa8.jpeg" width="350">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/56ec3c21-d5aa-f82a-7a77-2ceebd2a6730.jpeg" width="350">

#### ブール値

ブール型の場合はキーの入力は他と変わりませんが、値部分は`真`or`偽`のどちらかを選択する形になります。

ここまでで、URL以外のアクションが完成したので、一旦完了ボタンを押して終了します。

## ◇動作確認

作ったショートカットを実行して、実際にデータを受け取ってみます。
今回はいくつかのサービスを使っています。
なお、それぞれのサービスで発行されるURLが異なるため、発行されたURLを先ほど作成したショートカットのURL欄に入力しています。

### webhook.site

webhook.siteは個別にWebhook用のURLを割り振ってくれ、そのURLにアクセスする度にその詳細を表示してくれるサイトです。

https://webhook.site

参考にしたサイトを以下に記載します。

https://cybozu.dev/ja/tutorials/hello-kinapi/webhook/

https://sendgrid.kke.co.jp/blog/?p=11340

まずサイトにアクセスすると、自動的に個別のURLが割り振られます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e5d22976-1fbb-5b77-4b0a-e2f7d5417c2a.png" width="800">

そのURLを先ほどのショートカット内のURLに入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/08b76e58-846d-654a-3e25-7d574edc08fa.jpeg" width="350">

その後、ショートカットを実行するのですが、以下のようなポップアップが出た場合は許可を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d248b179-5cf7-c577-cd4f-9636dff8cbb6.jpeg" width="350">

ショートカットを実行後、再度webhook.siteを見ると、アクセス内容が左側に表示されます。
出力結果は下の画像のようになります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/80952499-3d5b-6c21-7d1b-18427e5b75ed.png" width="800">

```json:出力結果
{
  "": [],
  "SNS": {
    "Qiita": {
      "name": "@yankee",
      "articleNum": 17
    },
    "Zenn": {
      "name": "@yankee",
      "articleNum": 8
    },
    "Github": {
      "name": "yankee-08",
      "repository": 5
    }
  },
  "tagList": [
    "IoT",
    "ローコード",
    "マイコン",
    "自宅サーバ"
  ],
  "isDeleted": false,
  "userId": 123,
  "userName": "yankee"
}
```

問題なく、JSON形式でPOSTされていることが確認できました。
なお、今回はショートカットアプリ内でヘッダーに`"Content-Type:application/json"`を指定していませんでしたが、webhook.site上の出力を見ると自動で`"Content-Type:application/json"`に設定されているようです。

***

### Azure Logic Apps

次に、Azure Logic Appsを使って動作を確認してみます。
Azure Logic Appsは別の記事でも使っていますが、Azure版Power Automateといった形でPower Automate使ったことがあればそこまで戸惑わずに使えました（厳密にはできることが少し異なります）。
Power Automateのプレミアムコネクタが従量制で使えるので、個人的にちょっと試す際などに使ってます。
Azure Logic Appsの概要は以下のサイトが参考になりました。

https://learn.microsoft.com/ja-jp/azure/logic-apps/logic-apps-overview

https://qiita.com/akihiro_suto/items/725031627d4f1844b210

Azure Logic Appsのリソース作成方法については割愛します。
リソース作成方法は公式ドキュメントに手順が載っているので、こちらを参考にしてください。

https://learn.microsoft.com/ja-jp/azure/logic-apps/quickstart-create-example-consumption-workflow

なお、Azure Logic Appsには、Standardプランと従量課金プランがありますが、今回は従量課金プランを選択しています。

全体のフローは以下のようになります。今回はトリガは`When a HTTP request is received`トリガを使用しました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8b97112c-9d7b-b409-503f-7ce4fd6a4063.png" width="400">

以下の公式サイトとほぼ同じ手順です。

https://learn.microsoft.com/ja-jp/azure/connectors/connectors-native-reqres?tabs=consumption

`When a HTTP request is received`トリガでは、POSTメソッドのみ有効に設定しJSONスキーマについては、サンプルのペイロードから生成する方法を使っています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6b3bc031-322c-3b77-61ea-0318c08dc391.png" width="800">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/61e8789a-266e-0afd-e05d-5ad212e46f64.png" width="800">

`Response`アクションでは、受け取ったデータのbodyをそのまま送り返しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9cda1f0a-c2d1-bdb7-e639-acb9a4f1bb56.png" width="600">

ここまでできたら、保存ボタンを押します。すると、`When a HTTP request is received`トリガにWebhook用のURLが発行されるため、そのURLをショートカットアプリ内のURL欄に入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/990db550-23c4-cfd5-438d-9867e34b09d7.png" width="600">

ショートカットを実行後、作成したAzure Logic Appsのフローの実行履歴画面を開くとフロー実行履歴が表示されているはずです。
各行の実行結果の項目をクリックすることでフローの結果詳細を確認できます。
`When a HTTP request is received`トリガ内の「出力」-「body」を確認すると以下のようになっており、JSON形式でデータが送られていることが確認できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6ed8a9da-b919-75d4-9235-b406e72e91dc.png" width="800">

```json:body全文
{
  "": [],
  "SNS": {
    "Qiita": {
      "name": "@yankee",
      "articleNum": 17
    },
    "Zenn": {
      "name": "@yankee",
      "articleNum": 8
    },
    "Github": {
      "name": "yankee-08",
      "repository": 5
    }
  },
  "tagList": [
    "IoT",
    "ローコード",
    "マイコン",
    "自宅サーバ"
  ],
  "isDeleted": false,
  "userId": 123,
  "userName": "yankee"
}
```

***

### Postman Flows

最後に、Postman Flowsを使って動作を確認してみます。
Postman FlowsはPostmanのサービスの1つで、ワークフローを構築するためのビジュアルエディターです。
詳細は、公式の資料が参考になります。

https://www.postman.com/product/flows/

https://docs.google.com/presentation/d/1xpf1V8J0PtPooFRgFNISMtcS9W1fDCjMOtJxSBi0U08/edit#slide=id.g2da6a9d2968_1_689

つい最近、Postman Flowsのオンラインハッカソンがあり、これに向けてiPhoneショートカットと上手く連携できないかと言うところを試していましたが、結局完成には至らず、ハッカソンには応募できずじまいでした。
なので、せめてそのときに調べた内容を記事にしておこうと思ったのがこの記事を書くキッカケです。

Postman Flowsで新規フローを作成するには、トップ画面の左側から「フロー」を選択します。
なお、フローのアイコンが表示されていない場合は、黄色枠のアプリ追加アイコンを選択し、フローのチェックボックスを有効化してください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b32dbd28-ac23-8059-48e7-a248cc138c8a.png" width="350">

「新規」ボタンを追加すると新たにフローが作成されますので、フローの名称を分かりやすい名前に設定しておきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e05d3154-2bb6-ab62-3557-48d3c407f4cd.png" width="600">

ここにブロックを追加していく流れになるのですが、今回はHTTPリクエストした内容が正しく届いているかを確認するだけなので、`Log`ブロックを配置します。
ブロックを配置するには、`Start`ブロック右端をマウスクリックしてドラッグするか、何もない画面上で右クリックすることで新規ブロック追加メニューが表示されます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d57c21c5-e1bb-9258-c422-37efe56dc3a5.png" width="800">

各ブロックの役割はこちらの記事が参考にしてもらえればと思います。

https://qiita.com/nagix/items/2ebedd0273e7eb13c2cf#%E5%87%BA%E5%8A%9B%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF

出力系のブロックとしては、`Log`ブロックに加えて、`Output`ブロックもあり、こちらはデータの表示やチャート、テーブル形式の表示など様々な表示が可能なのですが、今回試した限りでは、HTTPリクエストのbodyに格納したJSONデータを上手く表示することができませんでした。
ブロックの設定方法が誤っているのか、仕様なのか現状不明なため、解決した場合は追ってこの記事に反映したいと思います。

ブロックの配置が完了したら、左側に`Webhook`の`Create`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5f0e21da-8aeb-e592-4d29-5b504b0cb4da.png" width="600">

その後、このフローを実行するためのURLが表示されるため、このURLをショートカットのURL欄に入力します。
なお、このときに必ず`Publish`を実行してください。
**（フローの中身を変更する度に`Publish`を実行しないとそのフローが更新されないため注意）**

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/51b31e54-be55-be13-ec6f-675a7d0e2093.png" width="500">

次に、三点リーダーを選択後、`View published version`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/abfded3a-1a34-6ba4-54fa-b7c9e60a03ea.png" width="400">

この状態でログ出力画面が下部に表示されていない場合は`View Live Logs`ボタンを選択して、ログ出力画面を表示します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7fa19119-3134-3d8a-3dc4-3bcca873e12b.png" width="800">

ここまでできたら、先ほど同様ショートカットを実行すると、Postman Flowsのログ出力画面に新規でログが出力されます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d3ffa905-be81-4455-d712-144f383f07aa.png" width="300">

こちらでも問題なく、JSON形式のデータが送られていることを確認できました。

## ◇オートメーション機能

ショートカットアプリには、オートメーション機能があり、この機能を活用すると先ほどのJSONをPOSTするトリガーを設定することができます。
オートメーションの概要は公式ページに詳しい記載があります。

https://support.apple.com/ja-jp/guide/shortcuts/apd690170742/ios

試しに、簡単なオートメーションを新規作成し、その中で先ほど作成したショートカットを呼び出してみます。

まず、ショートカットアプリを開き、オートメーションのタブを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f6e212f8-e481-2434-c59b-6827f7b71269.jpeg" width="250">

右上の`+`ボタンを選択し、新規オートメーションの作成画面に遷移します。
その後、オートメーションの実行条件（トリガ）を一覧から選択します。

今回は、Wi-Fiへの接続をトリガとしています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f434425c-8099-0684-e2a4-d424921327f7.jpeg" width="400">

なお、使用できるトリガについては、公式ドキュメントに記載があります。

https://support.apple.com/ja-jp/guide/shortcuts/apd932ff833f/7.0/ios/17.0

ネットワークの項目では、過去に接続したネットワーク（SSID）を選択するか、任意のネットワークを選択します。
今回は、任意のネットワークとしています。
オートメーションの実行時は「確認後に実行」としています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5717210b-1e08-c6be-50e0-a0f52e007d4e.jpeg" width="400">

「次へ」を選択すると、アクションを選択すル画面になるので、先ほど作成した、「JSONポスト」ショートカットを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e77f9ea3-a58b-c96a-9d30-4126e6b38255.jpeg" width="400">

これでオートメーションが作成できましたので、Wi-Fiを一旦切断後に改めてWi-Fiを接続してみます。
すると、画面上部にショートカットの実行に関する通知が表示されるので、これを選択すると、ショートカットの実行確認画面が表示されます。
ここで、実行を選択すると、先ほど同様ショートカットが実行され、JSONがPOSTされました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0b4045dd-7e90-71d8-c2c6-1d9c08e7ef12.jpeg" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7572fbc3-ea4f-6327-fc28-f976868e5906.jpeg" width="600">

ちなみに、オートメーション登録時に「確認後に実行」ではなく、「すぐに実行」を選択した場合、下のような通知が表示され、ショートカットが自動的に実行されました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/353425de-d376-bc62-c58a-aeb6ca6c8e54.jpeg" width="600">

## ◇おわりに

今回は、JSONをPOSTするまでを試しましたが、次回はこのショートカットを使って、自動化を試してみたいと思います。

# 🔚END
