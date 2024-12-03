---
title: iPhoneからREST APIでkintoneにレコード登録する
tags:
  - iPhone
  - ショートカット
  - REST-API
  - kintone
  - Postman
private: false
updated_at: '2024-12-03T21:57:39+09:00'
id: 47f6fda83dc2778ad13f
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

本記事は、「kintone Advent Calendar 2024」の3日目の記事です。

https://qiita.com/advent-calendar/2024/kintone

先日は[ADVENTARのkintoneアドカレ](https://adventar.org/calendars/10522)に記事を載せましたが、続いて今回はQiitaアドカレでも記事を書いていきます。

今回はkintoneのレコードを外部から制御できる **kintone REST API**をiPhoneショートカットを使って呼び出してみるという記事です。

## ◇環境等

- 使用デバイス
  - iPhone SE3
    - OS：iOS18.1
  - WindowsPC
    - Windows11
- 使用アプリ
  - ショートカットアプリ
- 使用ツール・サービス
  - Edge
  - kintone（PCブラウザ経由）
  - Postman

## ◇最終的にできたもの

iPhoneショートカットをタップして、名前を入力すると、kintoneアプリ上に入力したデータをレコードとして追加できます。
モバイル版kintoneアプリを使用せずにREST APIを使ってレコードを登録する形になります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f321058a-5a80-4669-418e-45d9c0ab58c6.png" width="800">

## ◇実装手順

1. レコード登録用のkintoneアプリの作成
2. REST API用のトークン発行
3. Postmanを使ったREST APIの動作確認
4. iPhoneショートカットの作成

の順で記載していきます。

:::note info
記事書いてから気づきましたが、kintoneの公式ドキュメントで上記の1～3までを行った記事が公開されていました。
自分が書いた部分は今更消すのもあれなので、そのまま記載しています。
適宜公式の記事も参照ください。
:::

https://cybozu.dev/ja/kintone/tips/development/development-productivity/http-client/kintone-rest-api-using-postman/

### kintoneアプリ登録

まずはレコード登録用のkintoneアプリを作成します。
なお、kintoneは開発者向けのkintone開発者ライセンス（12か月間利用可能）やお試しライセンス（30日間利用可能）があるため、契約をしなくてもアプリ作成を試せます。
※今回作成したアプリもkintone開発者ライセンスでつくってます

https://cybozu.dev/ja/kintone/developer-license/

https://kintone.cybozu.co.jp/trial/

作成したライセンスでkintoneにログインしたら、アプリの作成を行います。
なお、ユーザーの追加やスペースの追加方法については、公式サイトやヘルプサイトにたくさん情報があるため、詳細は割愛します。

https://kintone.cybozu.co.jp/material/

https://jp.cybozu.help/k/ja/start.html

ポータル上からアプリを作成アイコンをクリックして、「はじめから作成」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b55c97d2-df7e-1209-d20a-afa0528a371a.png" width="500">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/66a4b50c-e6f4-a1c9-84ff-126b99c6e39a.png" width="700">

アプリの名前は好きな名前を入力します。
（自分の環境ではREST APIテスト レコード登録アプリにしてます）
次に、フォームタブから`文字列（１行）`フィールドをフォーム画面上にドラッグして配置します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/fcdabe03-be9f-1967-bc68-9f239fdca1d2.png" width="500">

配置したフィールドの設定画面から赤枠の箇所を変更します。
今回はフィールド名、フィールドコードともに`Name`にしています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/cc49a913-f338-1f07-8ae6-b5ee731296bd.png" width="800">

ここまできたら、一旦アプリを公開します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b17de9e9-e838-b29c-5b22-02f9d6e81d79.png" width="500">

公開されたアプリに試しにレコードを追加します。
右上のレコード追加の`＋`ボタンをクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b4a94abb-3a6d-cd37-ec11-3ea9bac338a1.png" width="700">

入力項目は`Name`フィードだけなので、適当に名前を入力して保存します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4cb47d18-a885-0525-dc81-ea392c64ae13.png" width="400">

一覧に先ほど登録したレコードが追加されていればOKです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d988bae2-d9ec-9e7c-3206-67d455b86ea7.png" width="400">

### APIトークン発行

次に、先ほど登録したアプリのレコードを外部から制御するために、APIトークンを発行します。
APIトークンの作成方法は公式ヘルプも参照ください。

https://jp.cybozu.help/k/ja/user/app_settings/api_token.html

先ほど作成したアプリの設定画面から、`⚙`アイコンを選択し、設定タブ内にあるAPIトークンを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/dc52351e-b5ed-4018-0462-e439943b3981.png" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2e98e0f8-a841-a83c-e80b-26db3b1b3808.png" width="500">

APIトークン設定の画面左上にある`生成する`ボタンをクリックし、APIトークンを発行します。発行されたAPIトークンは`レコード追加`のチェックが有効ではないので、チェックを入れて有効化します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5132c67c-534d-b1a9-52b7-5211722bffac.png" width="600">

これでAPIトークンの準備ができたので、改めてアプリを公開します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/71dc0ead-ec74-3b67-a267-f0c461d89a12.png" width="600">

### Postmanでの検証

APIトークンが取得できたので、いよいよ外部からレコードの登録を試します。
いきなりiPhoneショートカットで試した場合、処理がうまくいかなかった場合のデバッグが難しいため、まずはAPIプラットフォームのPostmanを使ってレコードの登録を試してみます。
Postmanに関する情報は、Postman社員の方が過去に実施した際のセミナー資料を色々公開しているので、こちらを参考にしてください。

https://postman.connpass.com/presentation/

自分も以前にPostmanでSwitchBotを制御する記事を書いたりしてます。

https://qiita.com/yankee/items/c31b09811c1e27956a62

まず、新しいコレクションを作成し好きな名前を付けます。（自分は`kintone REST API`という名前に設定）
その次に変数タブに移動し、3つの変数を追加します。

- `baseUrl`：レコード登録用のURLを入力します
以下のURLの`xxxxxxx`の部分を個人の環境に合わせます
`https://xxxxxxx.cybozu.com/`

https://cybozu.dev/ja/kintone/docs/rest-api/records/add-record/#url

- `apiToken`：[先ほど発行](#apiトークン発行)したトークンを入力

https://cybozu.dev/ja/kintone/docs/rest-api/records/add-record/#request

- `appId`：最初に登録したアプリのIDを入力
アプリIDは、そのアプリを開いた時のURLの数字が該当します。

入力が完了したら、右上の保存ボタンをクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8ebd1309-d8a5-3706-95d1-49ea4e728f61.png" width="700">

次に、コレクション内にAPIのリクエストを追加します。
名前を適当に決めて（自分は`1件のレコードを登録する`という名前に設定）、リクエスト方法を`POST`に変更します。
URL部分は以下のように入力します。ここで、`{{baseUrl}}`は先ほど登録した変数を表します。

`{{baseUrl}}k/v1/record.json`

ヘッダータブでは、公式ヘルプを参考にし下の画像のように入力します。
APIトークンは先ほど作成した変数を使います。

https://cybozu.dev/ja/kintone/docs/rest-api/records/add-record/#request

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/cb17357c-1d97-db10-44c8-9f4494e2773c.png" width="700">

ボディのタブでは、`Raw`モードの`JSON`を選択し、以下のように入力します。
名前は好きな値を入力して問題ないです。
画像では、postmanという名前を設定しています。

```JavaScript
{
  "app": {{appId}},
  "record": {
    "Name": {
      "value": "xxxxxxx"
    }
  }
}
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ee146b16-3a84-ba47-9e91-ba105d916554.png" width="700">

ここまできたら、APIリクエストの内容を保存し、送信ボタンをクリックします。
少しすると、画面下側のレスポンス部分が更新されます。
ここで、赤枠で囲んだ部分が`200 OK`と表示されていれば成功です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d14994cd-92e6-a6bb-303a-449abf3aec72.png" width="700">

kintoneアプリ側に戻って一覧表示のページを更新すると、Postmanからリクエストしたレコードが追加されています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/34b0a023-5578-e3c2-c8d6-0e38a0abb5d1.png" width="700">

### iPhoneショートカットの作成

Postmanを使ったレコード登録ができたので、今度はiPhoneショートカットを使ってレコード登録に挑戦します。
なお、iPhoneショートカットを使ったHTTPリクエストのPOST方法は以前記事を書いていますので、細かい部分はこちらの記事をご覧ください。

https://qiita.com/yankee/items/356c0216347650af67e0

ショートカットアプリを開き、新規のショートカットを追加して好きな名前やアイコンを設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0c5dbfa1-ed1a-9473-7397-e48b38017150.png" width="300">

最初に、レコードに登録する名前を入力するアクションを追加します。
`登録する名前を入力してください`の部分は入力するウィンドウのタイトルなので、自由に設定できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a6db5804-66f6-d512-c137-9a38a755054f.png" width="300">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/3e35a655-31f0-95d9-e0df-28c057ea587b.png" width="400">

なお、テキストの項目は選択すると、数字やURL、日時データなどを選ぶことも可能です。（今回はテキストのままでOK）

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e77454b2-34e2-d5b9-9fe0-d334da830cce.png" width="300">

次にkintoneにHTTPリクエストするためのアクションを追加します。
`URLの内容を取得`アクションを利用します。取得となっていますが、実際にはPOSTも可能なアクションです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/01639c89-9ffe-e96a-a192-942911d83a7d.png" width="300">

入力項目は先ほどPostmanで登録した内容を参考に入力していきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/842d4a9b-b75d-8ba2-758d-5991c339c8df.png" width="400">

ボディの部分については、階層構造となっています。
階層構造の部分（`record` - `Name` - `value`）については、フィールドタイプを**辞書**にすることで階層でのデータ入力が可能になります。
[こちら](https://qiita.com/yankee/items/356c0216347650af67e0#%E8%BE%9E%E6%9B%B8)も参考にしてください。

```JavaScript
{
  "app": {{appId}},
  "record": {
    "Name": {
      "value": "xxxxxxx"
    }
  }
}
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/aa773c0c-2145-a059-a033-c265d5710007.png" width="500">

最後にリクエスト結果を表示するためのアクションを追加します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7ac9264b-4c5b-8615-7eed-0d35e6ddde02.png" width="400">

表示内容は`URLの内容`を選択してください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/493ba4df-73f3-afb4-6b57-1eb691808d24.png" width="400">

ここまでで、iPhoneショートカットが作成できましたので、実際に動作確認してみます。

## ◇動作確認

作ったショートカットを実行して、実際にレコードを登録してみます。

ショートカット一覧画面から、先ほど作成したショートカットをタップします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d540fa34-b0d5-646e-179e-9b378474bcfd.png" width="400">

タップ後、画面上部に登録するレコードのNameを入力するウィンドウが表示されるので何か文字を入力します。（自分は`iPhone`という名前に設定）
完了ボタンを押すと、アクションが進み、最終的に以下のような画面が表示されるので、完了ボタンをタップして終了です。
※Postmanの時のように、HTTPレスポンスのステータスコードを表示したかったのですが、結局出し方がわからず断念してます

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c25c34c8-062e-2384-0c3d-bb1e23e30a9b.png" width="400">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/fcea9221-97e5-4f79-e23e-bdad3c0fb1bc.png" width="400">

kintoneアプリ側に戻って一覧表示のページを更新すると、今回のレコードが追加されています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/41c83867-06a3-2b8f-1bcf-80142b3a85f9.png" width="600">

## ◇おわりに

kintoneはスマートフォン用のアプリも提供されているため、そのアプリを使えばレコードの追加は可能です。
ただ、送るデータの入力項目がほぼない、入力する内容がほぼ決まっているといったシチュエーションでは、今回のようにショートカット経由でレコード登録すると少し手間が省けると感じ、今回の記事を書いてみました。
今回はテスト用のアプリでしたが、例えば毎日の体調チェック（選択肢を用意しておき、それを選ぶだけのアプリなど）のようなアプリとかだと使えるかもしれません。
また、iPhoneショートカットはオートメーションの機能もあるため、これを活用することで自動化といったことも可能です。（できれば今年のアドカレ期間内にその辺の記事も書きたい・・・）

乞うご期待！！

# 🔚END
