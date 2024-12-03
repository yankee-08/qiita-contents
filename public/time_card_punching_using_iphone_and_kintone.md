---
title: iPhoneとkintoneを用いたタイムカード打刻の自動化
tags:
  - kintone
  - iPhone
  - ショートカット
  - REST-API
  - 自動化
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

本記事はRPA Advent Calendar 2024 7日目の記事になります。

https://qiita.com/advent-calendar/2024/rpa

今回は、[kintoneアドカレ](https://qiita.com/advent-calendar/2024/kintone)で作成した以下の記事をベースにし、iPhoneのオートメーション機能も活用して、kintoneへのデータ自動登録アプリを追加していきます。
仮のユースケースとして、出退勤のタイムカード打刻を自動化するシステムをつくってみます。

https://qiita.com/yankee/private/47f6fda83dc2778ad13f

## ◇最終的にできたもの（イメージ図）

下の画像がイメージ図となります。
社員が出社して会社支給のiPhoneが社内Wi-Fiにつながったことをトリガとして、kintone上にあらかじめつくっておいたタイムカードアプリに出勤レコードを登録します。
逆にWi-Fiの接続が切れた場合は退勤レコードを登録するといった流れになります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ae8bf541-59b6-e07e-7716-02f76a3edf2c.png" width="800">

## ◇環境等

- 使用デバイス
  - iPhone SE3
    - OS：iOS18.1
  - WindowsPC
    - Windows 11
- 使用アプリ
  - ショートカットアプリ
- 使用ツール・サービス
  - Edge
  - kintone（PCブラウザ経由）
    - （kintone開発者ライセンスを使用）

### アプリ設計

今回つくるアプリ間の関係をイメージしてみます。
まず、メインとなるタイムカードアプリが必要になります。
タイムカードアプリでは、

- 打刻日時
- 出退勤のステータス（出勤or退勤）
- 従業員情報

あたりが最低限必要になると想定されます。
この際、従業員情報は別テーブルで持っていたほうがいいので、従業員マスタは別に作ることにします。

また、今回はiPhoneから自動でタイムカードの打刻レコードを登録する予定です。
iPhone側で従業員情報を持たせてもいいのですが、iPhoneの端末毎の固有IDからレコード登録時に従業員情報を取り出す仕組みにしたいと考え、端末マスタを別に持たせることとしました。

これらを踏まえて、簡易的なER図を描いてみました。
※あくまで設計時点のイメージ図で、kintoneでの実装はこれとは一致していませんので注意

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/909f1da4-ec82-7e99-fc40-3d7249c3fd79.png" width="800">

## ◇実装手順

1. マスタテーブル用のkintoneアプリ作成
2. タイムカードkintoneアプリ作成＆REST API用のトークン発行
3. Postmanを使ったREST APIの動作確認
4. iPhoneショートカットの作成
5. iPhoneオートメーションの作成

の順で記載していきます。

### マスタテーブル用のkintoneアプリ作成

最初に、従業員マスタと端末マスタ用のアプリをつくっていきます。
2つのアプリの作る順序ですが、端末マスタから従業員マスタの情報を参照する形になりますので、先に従業員マスタからつくっていきます。

従業員マスタについては、サンプルアプリの中に「従業員名簿」アプリを含む「人事労務パック」があるため、こちらを使用します。
アプリ作成を選択し、「人事労務パック」を選択します。
「サンプルデータを含める」のチェックが入っていることを確認し、アプリパックを追加します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/be733aa2-7eb8-3dea-f0a7-0753c6bbf51c.png" width="500">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/48e4b08f-545f-64ef-ae3c-84c9b60f54d2.png" width="700">

アプリを開くと、架空の従業員情報が入っていることが確認できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/cbef26ed-3f83-0a69-e642-9d744f2e7e73.png" width="600">

次に、端末マスタ用のアプリをつくります。
こちらは、流用できそうなサンプルアプリが見つからなかったため、「はじめから作成」を行います。

下のように、4つのフィールドを配置しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a843c6b4-bd34-80a2-3c16-3a486f44aabd.png" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0c646487-5c78-eb8d-b12c-5a09d5fd8241.png" width="300">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7faf2d34-e0c4-5152-c24e-a1987f61f4f6.png" width="300">

従業員名フィールドには、ルックアップを設定し、先ほど追加した[従業員マスタ] - [氏名]のフィールドと紐づけます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/aedde269-09b6-2bdc-625a-2edb3e220f3e.png" width="500">

ルックアップの設定方法については、公式ヘルプを参考にしてください。

https://jp.cybozu.help/k/ja/user/app_settings/form/lookup/set_lookup.html

アプリを公開したら、サンプルデータとして何件かデータを入力しておきます。（今回は3件入力）
従業員名は従業員マスタから好きな名前を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/868f4721-cc67-7b49-d189-63caf99cb8ab.png" width="600">

### タイムカードkintoneアプリ作成＆REST API用のトークン発行

次に、タイムカードアプリをつくっていきます。
こちらもサンプルアプリ内に「タイムカード」アプリがあるため、とりあえずこちらを使って必要に応じてカスタマイズしていきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2109c678-1115-0168-5695-4211716460ac.png" width="400">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/811e75b0-e24e-58a7-d905-85f473315712.png" width="400">

サンプルアプリのフォームを確認してみます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f35b436c-28d5-828c-1081-b09277f45f53.png" width="600">

このアプリでは、出勤時刻と退勤時刻を同時に入力し、さらにプロセス管理で承認フローも行う仕様となっています。
この中で、今回のアプリでは必要ない

- 承認者フィールド
- 退勤時刻フィールド
- 勤務時間フィールド
- レコードタイトル

を削除します。
このとき、「承認者フィールド」については、**プロセス管理**でフィールドが使用されているため、そのままだと削除できません。
そのため、まず[設定] - [プロセス管理]から、承認者フィールドが入ってる箇所を削除します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8ad537b3-363c-14b4-536a-abe98f4f2823.png" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4e47a748-4ef8-6903-4c34-8799a58bf1dc.png" width="400">

最後にプロセス管理を無効にしたら、改めて不要なフィールドを削除します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/071aa54f-8f39-895f-729a-0550c1a7e5b4.png" width="500">

フィールドを削除したら、今回のアプリで必要なフィールドを新規追加します。

- 時刻フィールド（出勤時刻フィールドの名前変更で対応）
- 出退勤ステータス
- 端末ホスト名
- 従業員名

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/13e44039-d8c8-8740-0293-ab1c0564e4cd.png" width="500">

出退勤ステータスフィールドについては、ドロップダウンを使用しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/3871a55c-f193-34c4-9f2c-0cf771feac7f.png" width="500">

端末ホスト名フィールドは先ほど作成した端末マスタからドロップダウンで紐づけます。
その際、従業員名のフィールドもコピーします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b79057fb-a3ea-1210-fcea-7b9e3ece382e.png" width="600">

ここまででフォームの配置が完了したので、レコード一覧の表示形式も変更します。
今回は、以下のように配置しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/291e0d18-a3e9-5e60-7559-54c7190165d8.png" width="800">

ここまででアプリができましたので、最後にAPIトークンを発行して保存します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/67926f2e-194e-0fe5-327b-6dc7493378e7.png" width="800">

### Postmanを使ったREST APIの動作確認

[前回](https://qiita.com/yankee/private/47f6fda83dc2778ad13f#postman%E3%81%A7%E3%81%AE%E6%A4%9C%E8%A8%BC)の記事と同じ手順でレコードの登録を試します。
APIトークンやAppIDの変数の値を変更し、リクエストのボディの中身を今回の実装に合わせてデータを送信したところ、**エラー**が発生しました。

```JSON
{
    "code": "xxxxx",
    "id": "xxxxx",
    "message": "フィールド「端末ホスト名」の値「社用スマホ1号」が、ルックアップの参照先のフィールドにないか、またはアプリやフィールドの閲覧権限がありません。"
}
```

エラーログを見ると、`ルックアップの参照先のフィールドにない`といった記載があります。
このログを調べたところ、以下の公式ドキュメントに記載がありました。

https://cybozu.dev/ja/kintone/docs/rest-api/overview/authentication/#api-token

https://cybozu.dev/ja/kintone/tips/development/customize/development-know-how/multiple-api-token-capabilities/#auth-multi-token

> 複数のAPIトークンを指定すると、ルックアップフィールドがあるアプリで、ルックアップ元のアプリの値をコピーしてレコードを登録できます。
>
> 複数のAPIトークンを指定するには、カンマ区切り、または別のヘッダーでAPIトークンを指定します。
> 1回のリクエストで指定できるAPIトークンは9個までです。10個以上指定すると、エラーになります。

公式ドキュメントを読むと、ルックアップフィールドがある場合、ルックアップ元（今回の場合は端末マスタ）のレコードに対する参照権限が必要であり、そのためのAPIトークン（端末マスタアプリ側で発行）が必要ということがわかりました。

そこで、端末マスタアプリでも同様にAPIトークン（こちらは閲覧権限のみでOK）を発行し、トークンを結合します。

Postman上では、2個目のトークン用の変数を追加し、呼び出す際に2つの変数をカンマ区切りで結合しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7bb53699-c8bc-dfb4-82f2-5268f3d1d1b3.png" width="400">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/58316c88-92ce-e46d-4faf-204a15b40ea9.png" width="700">

改めてリクエストを実行すると、問題なくレコードが追加されました。

### iPhoneショートカットの作成

Postmanを使ったレコード登録ができたので、続いてiPhoneショートカットを作成します。
基本的な手順は[前回](https://qiita.com/yankee/private/47f6fda83dc2778ad13f#iphone%E3%82%B7%E3%83%A7%E3%83%BC%E3%83%88%E3%82%AB%E3%83%83%E3%83%88%E3%81%AE%E4%BD%9C%E6%88%90)と同じため、詳細は省略しますが、今回端末固有の情報を取るためのアクションを使う必要があります。

当初予定では、端末固有のIDなどを取得する予定でしたが、それらを取得する方法が見つけられなかったため、今回はデバイス名を取得する形としました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f0e89fe1-7d0d-ae05-f244-4211939f0a94.png" width="300">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1f62be7b-76f2-f899-df1e-c5dd76ca97ef.png" width="325">

`record`には、`端末ホスト名`に先ほど取得したホスト名、`出退勤ステータス`には、出勤or退勤の文字列を入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a88e953c-94db-5238-bfe9-f21ae97c5b99.png" width="700">

ショートカットが作成できたら、試してみます。

その際の注意点として、iPhoneの設定から端末の名前を先ほどkintoneの端末マスタに登録した端末ホスト名と一致させておきます。
※iPhoneの名前を変えたくない場合は、kintoneの端末マスタにiPhoneの端末名を新規レコードとして追加する形でもOK

ショートカットを実行して、kintoneのレコードが新規追加されていればOKです。
ちなみに、初めてショートカットを実行した場合は以下の確認画面が出ることがあります。
常に許可を選択すればその後は確認画面は出てきません（1度だけ許可の場合はその都度表示されます）。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b9b13e0b-a2bc-7bd5-1eca-f5f1a1208c05.png" width="300">

ショートカットを作ったら、そのショートカットを複製してもう一つショートカットを作成します。
最初に出勤用のショートカットを作成した場合、つぎは退勤用のショートカットを作成します。
「ショートカット名」と`出退勤ステータス`に入力するテキストを修正し保存します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/05797502-23a0-f22b-7883-bb8685e7e152.png" width="250">

### iPhoneオートメーションの作成

ここまでで、iPhoneショートカット機能を使ってkintoneのタイムカードアプリにデータを追加できるようになったので、次にオートメーション機能を使って自動化します。
オートメーションの概要は公式のユーザーガイドをご覧ください。

https://support.apple.com/ja-jp/guide/shortcuts/apd690170742/ios

iPhoneオートメーションの使い方については以前に別記事でも手順を載せているので、こちらも参考にしてもらええればと思います。

https://qiita.com/yankee/items/356c0216347650af67e0#%E3%82%AA%E3%83%BC%E3%83%88%E3%83%A1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E6%A9%9F%E8%83%BD

出退勤を模擬するため、会社に到着した条件でオートメーションを動かす形にしたかったので、今回は特定のWi-Fiへの接続時をトリガとしてオートメーションを動かす形としました。
このほか、GPSを使った方法もありましたが、動作確認する際に手間がかかるので、今回は見送ってます。

オートメーションの中から、`Wi-Fi`を選択し、ネットワークの中から特定のSSIDを選択します。
接続時は出勤用ショートカットを実行するように選択して完了ボタンを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c37b87a6-898a-f4e1-6fe0-75b009335548.png" width="700">

同様に、、`Wi-Fi`接続解除時は退勤用のショートカットを実行するオートメーションを作成します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/141e340c-620b-0b61-e91b-db625f981b1e.png" width="700">

これでタイムカード打刻自動化の準備ができたので実際に動作確認してみます。

## ◇動作確認

動作確認は単純にオートメーションで設定したWi-Fiへつないだり、接続を切ってみるだけです。
もちろんWi-Fiの接続を切った場合はLTE回線等でインターネットアクセスができないとオートメーションは失敗します。

Wi-Fi接続時or接続解除時のタイミングで画面の上部にオートメーション実行中のテキストが表示されます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0a3909ca-3429-994c-f3b0-05f29d39c48c.png" width="500">

しばらくすると、画面上にレスポンス結果が表示されるので、完了をタップします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b91f607b-4b77-a838-1a61-2957fcefbc84.png" width="500">

Wi-Fi接続、Wi-Fi接続解除後にそれぞれ完了画面が表示されるのを確認し、kintoneのレコード一覧に出勤と退勤のレコードが追加されていればOKです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f45a78bc-63ac-6408-d539-c1eb83c43fd4.png" width="700">

## ◇おわりに

前回の記事をベースにkintoneのタイムカードアプリの打刻の自動化をしてみました。
そのままこのアプリが使えるユースケースはなかなかないかもしれませんが、自動化の一例として参考になれば幸いです。

このあともRPA Advent Calendar 2024は続いていくので、ぜひほかの記事もご覧ください。

https://qiita.com/advent-calendar/2024/rpa

# 🔚END
