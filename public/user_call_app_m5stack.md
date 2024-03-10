---
title: M5Stack + Azure Logic Apps + Teamsを使った呼出しアプリ
tags:
  - Postman
  - uiflow
  - AzureLogicApps
  - Teams
  - M5stack
private: false
updated_at: '2024-03-10T15:49:31+09:00'
id: b5589373d7be53eceb97
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

最近、Azure logic Appsをさわり始めたので、勉強がてらM5Stackと組み合わせて呼出しアプリをつくりました。

## ◇開発環境等

- 使用デバイス
  - M5Stack Core2
- 使用ツール：
  - Azure Logic Apps
  - Teams
  - UIFlow1.0
  - Postman

Azure Logic AppsはAzure版Power Automateといった形でPower Automate使ったことがあればそこまで戸惑わずに使えました。
個人的に、プレミアムコネクタが従量制で使えるのが使い勝手がよかったです。
Azure Logic Appsの概要は以下のサイトが参考になりました。

https://learn.microsoft.com/ja-jp/azure/logic-apps/logic-apps-overview

https://qiita.com/akihiro_suto/items/725031627d4f1844b210

## ◇最終的にできたアプリ

作成したアプリの各画面です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/aa205cf7-e961-359e-e48f-bbce30ff5328.jpeg" width="250">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/008e7aec-6aa1-eef8-8af2-35e7ffecce3e.jpeg" width="250">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/da43e56b-7f79-6a8f-d28b-844d4a035ab7.jpeg" width="250">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c3a61ef5-18c4-9a99-d6e8-65379bcdaacb.jpeg" width="250">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/db0007cc-8f7d-3fc9-20a6-7cd22c186993.jpeg" width="250">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7160949b-6df6-722a-12ae-7fe52809aec8.jpeg" width="250">

選択画面から、呼び出したいユーザー名の右にある呼出しボタンをタッチしOKを押すと、Azure Logic Apps経由でTeams上のチャネルにメンション付きの通知がアダプティブカード形式で送られます。
（Teamsでの通知は受付に来客がきたという仮定の通知内容になります）
また、選択画面に表示されるユーザー名についても、Azure Logic Apps経由でOffice365 Groupのユーザー一覧を取得しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4e19fb61-223d-1c9e-e2c6-4d792f6c1445.png" width="800">

## ◇全体構成

全体の構成にM5Stackの画面遷移を加えたものを図にしました。
今回は、事前に画面遷移の設計をしていなかったので、完成したM5Stackの実際の画面を載せています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c5b8ff60-e4f9-66b6-a542-bfbd81bb05a6.png" width="800">

Teams通知までの流れとしては、

1. M5Stackが起動すると、Azure Logic Appsその1にアクセスし、Azure Logic Apps経由でOffice365のユーザー情報をjson形式で取得
2. 受け取った情報を基に、M5Stack上にユーザー名を表示
3. ユーザー名の右側にある「呼出し」ボタンを押して、「OK」ボタンを押すと、Azure Logic Appsその2にHTTPリクエストを送る
4. Azure Logic Appsその2では、受け取ったUPNを基に、ユーザー情報の詳細を取得
5. Teamsのチャネル上にメンション付きで通知を送る

という形になります。

## ◇実装

処理順とは異なりますが、Azure Logic Appsその1⏩Azure Logic Appsその2⏩M5Stackの順序で実装内容について記載します。

### Azure Logic Appsその1（ユーザー情報取得）

まず、Azure Logic Appsでユーザー情報を取得するワークフローを記載していきます。
なお、今回はAzure Logic Appsのリソース作成方法については割愛します。
リソース作成方法は公式ドキュメントに手順が載っているので、こちらを参考にしてください。

https://learn.microsoft.com/ja-jp/azure/logic-apps/quickstart-create-example-consumption-workflow

なお、Azure Logic Appsには、Standardプランと従量課金プランがありますが、今回は従量課金プランを選択しています。

#### フロー作成

全体のフローは以下のようになります。今回はトリガは「HTTP要求の受信時」トリガを使用しました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c6256a7d-7655-4ef9-e464-383ca5592f4b.png" width="600">

グループメンバーの一覧表示フローでは、指定したグループのメンバー一覧を取得します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1f3cd70e-6996-09d7-8cfd-e78cdebcedbc.png" width="600">

次に、取得したメンバー情報から必要な項目の抜き出しとjson形式への変換を行っていきます。
ユーザー毎のjson形式データをリスト化し、最後にリスト全体を`employee`キーのvalueとしてjsonの形にしています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1a21be58-114b-27d9-1220-506f0b0206cc.png" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0217268c-7bee-f613-3640-021adf59cb97.png" width="600">

#### 単体テスト

まず、このAzure Logic Appsが問題なく動作するかをテストしますが、この時点でM5Stack側は未実装のため、別の方法で確認を行う必要があります。
Python等のプログラミング言語で確認する方法もありますが、今回はAPIプラットフォームであるPostmanを使用しました。

https://www.postman.com/home

Postmanの使い方については、過去に実施されたセミナー資料がたくさん公開されていますので、こちらを参考にしてもらえればと思います。

https://postman.connpass.com/presentation/

また、私も過去にPostmanとSwitchBotの記事を書いてますので、良かったら読んでみてください。

https://qiita.com/yankee/items/c31b09811c1e27956a62

Postman上で、テスト用のコレクションを作成します。そのコレクション内に、2つのPOSTメソッドのリクエストを追加します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6bbd15bb-e89f-8b42-9262-96a033cb6a27.png" width="600">

まず、コレクション内の変数設定で'baseUrl'という変数を追加し、現在値にはAzure Logic Appsで生成されたURLを入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d0398e9b-61c8-277b-76d8-6b85ceda3a79.png" width="600">

次に、「ユーザー情報取得」リクエストを選択した状態で、「ボディ」項目内にパラメータを入力していきます。
この際、データ形式は`Raw`-`JSON`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f648ab51-f6a2-5d35-ea47-d03175eb6d34.png" width="600">

また、「ヘッダー」項目内で`Content-Type`が`application/json`になっているかを確認し、もしなっていない場合は修正します。
※`Content-Type`項目がない場合は新規追加

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/58e9bd59-db37-a62f-d1a2-6417f7bf9f5d.png" width="600">

この状態で、「送信」ボタンを押してHTTPリクエストを実行し、応答にユーザー情報が取得できればテスト完了です。

### Azure Logic Appsその2（Teamsメンション通知）

次に、Azure Logic AppsでTeams上に通知するワークフローを記載していきます。
なお、こちらのアプリも従量課金プランを選択しています。

#### フロー作成

全体のフローは以下のようになります。今回はトリガは「HTTP要求の受信時」トリガを使用しました。
グループメンバーの一覧表示フローでは、指定したグループのメンバー一覧を取得します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5fd119e9-21af-9f79-71d1-2400df22c845.png" width="600">

リクエストに含まれているUPNデータからそのユーザーの情報取得を行っています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/cb8f0c38-10ef-9bca-5c34-514f3d97b081.png" width="600">

取得したユーザー情報を使ってメンショントークンを取得し、チャネル上にアダプティブカード形式で通知します。

アダプティブカード形式で通知を行う方法は、以前に別で記事をまとめていますので、参考にしてもらえればと思います。

https://qiita.com/yankee/items/b9dfba1b5e461a389b19

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/717a491e-5c82-dadd-d95d-47949fa79fe2.png" width="600">

#### 単体テスト

こちらのAzure Logic Appsについても、先ほどと同様にPostmanを使ってテストします。
基本的には、先ほどと発行されるURLが変わりますので、コレクション内の変数設定内の'baseUrl'を修正します。
次に、「Teams通知」リクエストを選択した状態で、「ボディ」項目内にパラメータを入力していきます。
データ形式はさっきと同じ`Raw`-`JSON`です。
このとき、`UPN`のvalueは、実際に自テナント内で有効なものを入力してください。
なお、`nickName`については、ボディには含めていますが、今回Azure Logic Apps内では特に参照していません（UPNからユーザープロフィールを取得させているため）

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1315006e-2f08-07ce-ba85-dea7953b3220.png" width="600">

この状態で、「送信」ボタンを押してHTTPリクエストを実行し、応答が「○○を呼び出しました。」となっていればテスト完了です。

### M5Stack

最後にM5Stack側のUIFlowの実装内容について記載していきます。
今回はタッチパネル機能をもつM5Stack Core2を使用しています。

https://docs.m5stack.com/en/core/core2

全体としては、下の画像のようになります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1d18b92b-127b-0f77-15e9-6c71df6bc917.png" width="800">

ボリュームが多いため、処理毎に分けて説明していきます。

#### 初期化処理

起動後、変数の初期化とトップ画面表示のための画面初期化を行っています。
それらの初期化処理が完了後、起動画面用のテキスト（`lblBoolTexg`）とスクリーン（`lblBoolScreen`）をクリアしています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d5b81f39-e018-c6c2-60ed-a14c2c788041.png" width="500">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/aa205cf7-e961-359e-e48f-bbce30ff5328.jpeg" width="250">

変数初期化関数では、画面表示用の各種フラグ変数の初期化を行い、前述の[Azure Logic Appsその1（ユーザー情報取得）](#azure-logic-appsその1ユーザー情報取得)を呼び出しています。
レスポンスのjsonデータをM5Stack内で処理しやすいように、`社員リスト`、`社員名リスト`、`社員数`、`最大ページ数`といったデータを取り出しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/997ca9a9-ea2c-bf70-2090-ad8c0400a865.png" width="500">

#### トップ画面表示

初期化処理が完了すると、トップ画面を表示します。
この画面では、「呼出」ボタンを表示するだけの画面を表示し、ボタンを押すと、選択画面の1ページ目に遷移します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/054f49b3-ee6b-ca83-f717-40f451be5172.png" width="500">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/008e7aec-6aa1-eef8-8af2-35e7ffecce3e.jpeg" width="250">

#### 選択画面表示

選択画面では、`社員名リスト`変数から1ページにつき2ユーザー分を取り出し、表示しています。
ボタンA（M5Stack左下）、ボタンC（M5Stack右下）にはページ移動ボタンに割り当て、表示するユーザーを切り替えています。なお、最大ページ数を超えないようにクリップしています。
ボタンB（M5Stack下）を押すと、先ほどの[トップ画面](#トップ画面表示)に戻ります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9065e24c-da14-35a9-bc85-859aca2b7c63.png" width="700">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/da43e56b-7f79-6a8f-d28b-844d4a035ab7.jpeg" width="250">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c3a61ef5-18c4-9a99-d6e8-65379bcdaacb.jpeg" width="250">

ユーザー名の右側にある「呼出し」ボタンを押すと、呼出し確認画面に遷移します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/cf6e6463-f22b-be48-54ec-05f288ed45f0.png" width="500">

#### 呼出し確認画面表示

呼出し確認画面では、先ほど選択したユーザー名を表示し、呼出し確認用の「OK」、「Cancel」ボタンを配置しています。
「Cancel」ボタンを押した場合は先ほどの選択画面に戻ります。
「OK」ボタンを押した場合は、[Azure Logic Appsその2（Teamsメンション通知）](#azure-logic-appsその2teamsメンション通知)を呼び出しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/623b17c7-3d0f-aad3-f9fe-e09bfebc4382.png" width="500">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/db0007cc-8f7d-3fc9-20a6-7cd22c186993.jpeg" width="250">

`呼出し実行`関数の中では、先ほど選択したユーザー名の社員リスト要素番号をキーとした社員データをbody内に入れています。関数実行後に呼出し完了画面に遷移します。

:::note info
実際にテストしたところ、Azure Logic Apps側を無効にした状態でもHTTPリクエストが成功時側に倒れてしまったので、応答内容でチェックする必要がありそうです。
:::

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/283c68ab-d459-3800-dcad-cd9d13baab6f.png" width="500">

#### 呼出し完了画面表示

完了画面では、予め設定したテキストと「HOME」ボタンを配置しています。
「HOME」ボタンを押すと、[選択画面](#選択画面表示)に戻ります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/199239c8-2776-5f0e-21aa-fb24f0acc66e.png" width="500">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7160949b-6df6-722a-12ae-7fe52809aec8.jpeg" width="250">

### 結合テスト

最後に、全体を通してテストを行い、Teamsで通知されれば完了です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4e19fb61-223d-1c9e-e2c6-4d792f6c1445.png" width="600">

もし、うまく動作しない場合、

- Azure Logic Apps側の実行履歴が**ある**場合、その中身をチェック
- Azure Logic Apps側の実行履歴が**ない**場合（Azure Logic AppsまでHTTPリクエストが届いていない場合）、M5Stack側でデバッグ（UIFlow上でシリアル表示など）していく

といった切り分けで進めるのがいいと思います。

## ◇おわりに

今回は、マイコンボード（M5Stack）とiPaaS（Azure Logic Apps）の連携事例について記事を書いてみました。どなたかの参考になれば幸いです。

# 🔚END
