---
title: kintone REST APIとPostmanを使って画像ファイルをレコードに追加してみる
tags:
  - REST-API
  - kintone
  - Postman
private: false
updated_at: '2025-05-05T17:45:33+09:00'
id: c1259fd49d3fdff5219a
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

以前に別記事にて、iPhoneショートカットを使ってkintoneレコードを登録する記事を書きました。
この時の記事では、kintoneのREST API機能を用いてレコードを追加していましたが、添付ファイルをレコードに追加する場合、REST APIの呼び出し方が変わってくるため、そのやり方についてまとめつつ、Postmanを使って実際に添付ファイルを登録してみました。
以前書いた記事はこちらになります。

https://qiita.com/yankee/items/47f6fda83dc2778ad13f

## ◇環境等

- 使用デバイス
  - Windows PC
    - Windows 11
- 使用ツール・サービス
  - Edge
  - kintone（PCブラウザ経由）
    - （kintone開発者ライセンスを使用）
  - Postman

## 仕様の確認

まず、kintoneのREST APIに関するドキュメントを確認し、その手順について調べます。

> 一時保管領域にファイルをアップロードします。
> 一時保管領域とは、このAPIを利用してアップロードしたファイルが一時的に保管される場所です。
>
> - アップロードしたファイルのファイルキーを他のAPIで利用することで、一時保管領域のファイルをレコードやスペースなどに添付できます。
> - 一度にアップロードできるファイルは、1つです。
> - レコードを取得するAPIで取得できるファイルキーは、ファイルアップロードには利用できません。

https://cybozu.dev/ja/kintone/docs/rest-api/files/upload-file/

公式サイトに記載されているように、画像を含む添付ファイルをkintoneのレコードに登録する場合、
kintone上の一時保管領域にファイルをアップロードし、そこで受け取るファイルキーを使ってレコードに登録する2段階の対応が必要です。

自分なりのイメージ図をまとめるとこんな感じです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8c97df0c-6a64-4654-9b2a-124195b73bce.png" width="400">

## ◇実装手順

1. [レコード登録用のkintoneアプリの作成](#レコード登録用のkintoneアプリの作成)
2. [REST API用のトークン発行](#rest-api用のトークン発行)
3. [Postmanを使った画像ファイルのアップロード](#postmanを使った画像ファイルのアップロード)
4. [Postmanを使った画像ファイルのレコード登録](#postmanを使った画像ファイルのレコード登録)

の順で記載していきます。

なお、添付ファイルではない文字列などのフィールドをPostmanを使って登録する手順については、公式ドキュメントでも紹介されています。

https://cybozu.dev/ja/kintone/tips/development/development-productivity/http-client/kintone-rest-api-using-postman/

### レコード登録用のkintoneアプリの作成

まずは登録用のkintoneアプリを作成します。
※今回作成したアプリもkintone開発者ライセンスでつくってます

「はじめから作成」から、以下３つのフィールドを配置します。

- レコード番号
- 日時
- attach

アプリ名はわかりやすく、「ファイル添付APIテスト」としています。
`attach`フィールドに画像ファイルを登録します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a12bbdc1-4cea-4c9a-899b-2f67d21e540f.png" width="500">

### REST API用のトークン発行

アプリの作成完了後、設定画面からAPIトークンを発行します。
APIトークンの作成方法は公式ヘルプも参照ください。

https://jp.cybozu.help/k/ja/user/app_settings/api_token.html

先ほど作成したアプリの設定画面から、`⚙`アイコンを選択し、設定タブ内にあるAPIトークンを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0a77dd16-8adb-489c-9c23-d1640dc51c85.png" width="500">

APIトークン設定の画面左上にある`生成する`ボタンをクリックし、APIトークンを発行します。発行されたAPIトークンは`レコード追加`のチェックが有効ではないので、チェックを入れて有効化します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2abdc7aa-9cb3-4ff2-b27c-0f512dcf782d.png" width="600">

これでAPIトークンの準備が完了しました。

### Postmanを使った画像ファイルのアップロード

kintoneアプリにREST APIで登録する準備ができたので、Postmanからデータを登録します。

まずは、一時保存領域に画像ファイルをアップロードする手順です。
先ほどのイメージ図の左側部分です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/bece59e2-7b74-4837-a89c-cc2087b1b8d4.png" width="300">

[過去](https://qiita.com/yankee/items/47f6fda83dc2778ad13f#postman%E3%81%A7%E3%81%AE%E6%A4%9C%E8%A8%BC)の記事でも解説しているので、一部手順は省きます。
ファイルを登録する以外は基本的に手順は変わりません。

#### 変数の設定

Postman上で、コレクションを登録し、変数のタブを開きます。
以下、3つの変数を追加します。

- `baseUrl`
- `apiToken`
- `appId`

作成したアプリのURL上の表記から、`baseUrl`と`appId`を探して入力し、`apiToken`は[先ほど](#rest-api用のトークン発行)発行した文字列を入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c4bb1cd0-838c-423a-849a-895437271189.png" width="800">

#### 一時保存領域へのファイルの登録

作成したコレクション内で「リクエストを追加」を実行します。
任意の名称を入力し、メソッドは`POST`を選択します。

URL部分には、先ほど作成した変数を使用し、

```text
{{baseUrl}}k/v1/file.json
```

と入力します。

ヘッダー部分には、キーに`X-Cybozu-API-Token`、値に`{{apiToken}}`と入力します。
なお、この辺の記載方法は公式ドキュメントの記載を基に設定しています。

※公式ドキュメント上では、`Content-Type`の設定も記載がありますが、Postmanが自動で設定してくれるため、今回は省略しています

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1e5cb816-58ad-4cf0-b7c0-2e1dc0d7e9e9.png" width="800">

https://cybozu.dev/ja/kintone/docs/rest-api/files/upload-file/

最後にボディ部分についてですが、形式は`form-data`を選択します。
ここでは、1つのデータを登録します。

キーに`file`と入力し、形式を`ファイル`とします。
値の部分は、Select filesの部分で`New file from local machine`をクリックしてローカルにある画像ファイルを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/bdef5e70-867c-4496-8c92-a3e010a2670e.png" width="800">

今回は、Qiitaのアイコン画像（`qiita-icon.png`）を添付しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/12baad46-59b4-4ef7-af40-09c113f86663.png" width="800">

ここまでで、APIを実行する準備ができたので、右上の「送信」を実行します。

画面下のレスポンス画面上に`200 OK`と出れば成功です。

レスポンスで出力される`fileKey`の文字列がこの後のレコード登録で必要となるので、どこかにコピーしておきます。

```json
{
    "fileKey": "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx"
}
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/72b52986-c4bd-455a-9d7a-93c97fdbc36c.png" width="800">

:::note warn
注意
公式ドキュメントにも記載のあるように、以下の制限があります。

- 一時保管領域に保存されたファイルは、レコードやスペースなどに添付されない場合、3日間で削除されます。
- 一時保管領域のファイルも、ディスク使用量に含まれます。
:::

https://cybozu.dev/ja/id/bb1225dd8d192245c3645a95/#limitations

### Postmanを使った画像ファイルのレコード登録

次に、この`fileKey`を使って、レコードに画像ファイルを登録します。
先ほどのイメージ図の右側部分です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/83e42a30-45a7-4b4a-b1a6-2f43ec53aebe.png" width="300">

ここでのやり方は[過去](https://qiita.com/yankee/items/47f6fda83dc2778ad13f#postman%E3%81%A7%E3%81%AE%E6%A4%9C%E8%A8%BC)の記事とほぼ手順は変わりません。

公式ドキュメントは以下を参考にしています。

https://cybozu.dev/ja/kintone/docs/rest-api/records/add-record/

まず、先ほどの`fileKey`を扱いやすくするため、変数に追加します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/240075b1-3966-48cc-8d44-e7362d9e8129.png" width="800">

次に、新しいHTTPリクエストを追加します。
今回も、任意の名称を入力し、メソッドは`POST`を選択します。

URL部分には、今回は以下のように設定します。

```text
{{baseUrl}}k/v1/record.json
```

と入力します。

ヘッダー部分は先ほど同様、キーに`X-Cybozu-API-Token`、値に`{{apiToken}}`と入力します。

次に、ボディの部分は形式を`Raw`-`JSON`を選択し、以下のように入力します。
ここで記載の`attach`は、kintoneアプリで作成したフィールドコードになります。別のフィールドコードを設定している場合はそれに合わせて変更してください。

```json
{
  "app": {{appId}},
  "record": {
    "attach": {
        "value": [
            {
                "fileKey": "{{fileKey}}"
            }
        ]
    }
  }
}
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/60ab6b07-41a4-4b36-97b0-f99a200a9147.png" width="800">

添付ファイル形式のフィールドを追加する場合のフィールド形式は公式ドキュメントの以下のサイトに従って入力しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0e9ce8df-2777-461a-be10-2d40edb19ee1.png" width="600">

https://cybozu.dev/ja/id/2736678ef8d2aad09a33e8bb/#field-type-update

ここまでで、APIを実行する準備ができたので、右上の「送信」を実行します。

画面下のレスポンス画面上に`200 OK`と出れば成功です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6a9f804e-8d6a-476a-870a-bcf763330301.png" width="500">

ここで出力される`id`がkintone側のレコード番号となります。

kintone側を確認すると、以下のようにレコード番号4にQiitaのアイコン画像が登録されていることを確認できました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c33825df-7000-42c0-b5c0-897f3a91db97.png" width="500">

## ◇Postmanコレクションランナーを使った動作確認

ここまでのやり方の場合、手動でAPIを2回実行する必要があり、さらに1回目のリクエストのレスポンス内容(`fileKey`)を基に変数を書き換える必要がありました。

そこで、最後にPostmanが提供するスクリプト機能とコレクションランナー機能を使って、この作業を自動化してみます。

:::note warn
注意
アップロードする画像は固定（事前に設定した画像ファイル）です。
:::

### Postmanスクリプト機能

Postmanスクリプト機能には、リクエスト実行前に処理される**Pre-request**とリクエスト実行後に処理される**Post-response**があります。

この辺の詳細は、以下サイトに掲載されている資料や動画が参考になります。

https://postman.connpass.com/event/342162/presentation/

今回は、**Post-response**機能を使います。
ファイルアップロードのリクエスト実行後のレスポンスに含まれる、`fileKey`の内容を取得し、変数に格納します。

「ファイルをアップロードする」リクエスト側の「スクリプト」タブ-「Post-response」を選択します。

スクリプト入力欄に以下の内容で記述します。

```javascript
const responseJson = pm.response.json();
const fileKey = responseJson["fileKey"];

pm.collectionVariables.set("fileKey", fileKey);

pm.test("Response status code is 200", function () {
    pm.response.to.have.status(200);
});
```

レスポンスで返ってくるJSONデータの中から、`fileKey`の中身を取り出し、コレクション変数である`fileKey`の中に代入しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/fff5bc3f-eb6f-4728-859d-35b1145cfeb0.png" width="800">

最後に、保存して完了です。

### Postmanコレクションランナー

次に、コレクションランナー機能を使って、2つのリクエストを順に実行させます。

コレクションランナーはコレクション内のAPIを実行するための仕組みで、APIテストなどを行うための機能です。
こちらも、詳細は以下サイトに掲載されている資料や動画が参考になります。

https://postman.connpass.com/event/340643/presentation/

コレクションランナーを実行する場合、コレクション右側にある三点リーダーから、「コレクションを実行」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/55db76ab-60cb-4236-9566-11a0fb4b46be.png" width="600">

コレクションランナーの設定画面では以下のように設定します。
リクエストの実行順序だけ注意し、後は基本的に初期設定（マニュアル実行、実行回数1回）でOKです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e779045f-a0c8-4ceb-ab07-573a3f246112.png" width="700">

設定が終わったら、「Run kintone」を実行し、結果を待ちます。

実行結果画面では、2つのリクエストの結果が`200 OK`になっていることを確認します。
また、1つ目のリクエストのレスポンスの`fileKey`の内容が変数に格納されていることも確認します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8bbc5a75-d641-4ad5-b2e7-3aea93dde5fc.png" width="800">

今回も最後にkintone側でレコードが追加されていることを最後に確認すれば成功です。

## ◇おわりに

今回は、Postmanを使って、kintoneに画像ファイルをアップロードする手順をまとめました。
次のステップとして、iPhoneショートカットを使って、同様に画像ファイルをアップロードする記事をまとめたいと考えています。
最終的には、以前LTした以下の内容について、その手順をQiitaの記事にまとめる予定です。

https://www.docswell.com/s/yankee/ZR27E2-2025-04-02-rpalt

たぶん、あと2つ記事を書く必要がありそう・・・（続編を公開したらここにリンクを貼ります）

# 🔚END
