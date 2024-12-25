---
title: kintoneのタイムカードデータを基にGrafanaで在室ダッシュボードをつくって可視化する
tags:
  - grafana
  - kintone
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

本記事は**M5Stack** Advent Calendar 2024 17日目の記事になります。


12月のアドカレ内でいくつかの記事を書かせていただきました。
今回は、それらの記事で使った、kintoneとGrafanaを連携してみました。

## ◇背景

先日、[RPA Advent Calendar](https://qiita.com/advent-calendar/2024/rpa)にて、kintoneタイムカード打刻に関する記事を書きました。

https://qiita.com/yankee/items/9b950f41319f8f5a4a3b

このデータを基に、社員の在室・不在状況を可視化するひと目でわかるようなダッシュボードを作ろうと考えたのですが、kintoneの基本機能のみだと自分が作りたい感じにできませんでした（プラグインを探せばあるかも）。

そこで、[Linux Advent Calendar](https://qiita.com/advent-calendar/2024/linux)で書いた記事で使用した、Grafana＋Infinityプラグインの組み合わせを使ってkintoneのデータを取得し、ダッシュボードを作成できないかと考え、試すことにしました。

https://qiita.com/yankee/items/0918c7c8001831008b09

なお、上記記事ではDockerを使ってローカル上にGrafanaを構築していますが、今回はGrafana Cloudを使用しています（詳細は後述します）。

## ◇最終的にできたもの

最終的なGrafanaダッシュボード画面はこんな感じです。

上側のステータスカード表示では、各社員の最終出退勤ステータスを取得し、

- 出勤⇒在室
- 退勤⇒不在

という形で表示しています。

また、下のステータスタイムラインでは、出勤と退勤の状態を色分けて表示し、灰色が退勤、赤色が出勤を示しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/29c10ebb-0f37-c047-95ad-d9cd8f720969.png" width="600">

## ◇開発環境等

- 使用ツール・サービス
  - Edge
  - kintone（PCブラウザ経由）
    - ライセンス：kintone開発者ライセンス
  - Grafana Cloud
    - ライセンス：Cloud Free

## ◇実装手順

1. Grafana Cloudの環境構築
2. kintone側APIトークン発行
3. Postmanを使った動作確認
4. Grafana Cloudでのkintoneデータ取得
5. ダッシュボード作成

の順で記載していきます。

### Grafana Cloudの環境構築

Grafana Cloudはクラウド上でGrafanaのダッシュボードを構築できるサービスです。

https://grafana.com/products/cloud/

ライセンスがFree、Pro、Advancedの3つが用意されており、制限はありますが、Freeプランも存在します。

https://grafana.com/pricing/

今回は、このFreeプランを使用して、ダッシュボードの作成を行っていきます。

アカウントを持っていない場合、まずはアカウントの作成を行います。

https://grafana.com/get/

上記ページから「Create an account」を選択し、必要事項を入力していきます。

アカウント登録が完了すると、登録メールアドレス宛てに、Grafana CloudのURLが届くので、そのURLにアクセスします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/254e18cc-c06c-c222-f933-4655e90f1dff.png" width="500">

ログインしたら、画面左側のメニューから`Add new connection`を選択します。
次に検索ボックス上に`Infinity`と入力し、表示されるInfinityアイコンを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c642f5b6-dd30-1539-6658-58ea113b294f.png" width="400">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/3c675b17-e018-e9b2-3c3d-15c10f738fc2.png" width="600">

Infinityの画面を開いたら、右側にある`Install`を実行します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/adb47cf8-56c3-b61a-8f6d-88dc2a91ed5c.png" width="600">

ここまでできたら、kintoneのAPIトークンを発行します。

### kintone側APIトークン発行

次に、kintone側の設定を行います。
アプリ自体は前回の記事で作成したものをそのまま使います。

https://qiita.com/yankee/items/9b950f41319f8f5a4a3b#%E3%82%BF%E3%82%A4%E3%83%A0%E3%82%AB%E3%83%BC%E3%83%89kintone%E3%82%A2%E3%83%97%E3%83%AA%E4%BD%9C%E6%88%90rest-api%E7%94%A8%E3%81%AE%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E7%99%BA%E8%A1%8C

今回は、Grafanaからkintoneのデータを取得するためのAPIトークンの発行だけ行います。

タイムカードアプリの設定画面から、「APIトークン」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4a578ef3-c622-39e3-0b34-e8387e94c315.png" width="600">

`生成する`ボタンをクリックし、APIトークンを発行します。アクセス権については、今回はデータの取得だけで問題ないため、「レコード閲覧」のチェックのみ有効にしておきます。
このトークンは後で使うので、メモ帳等に保存しておきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d9b4765f-a5e5-adce-7ec0-ca6d612d70c7.png" width="600">


### Postmanを使った動作確認

APIトークンを使って動作確認をするにあたり、まずはPostmanを使ってデータ取得を試してみます。
なお、こちらも別記事で詳細を書いているため、今回はコレクション作成などの部分は省略します。

今回はタイムカードのデータを複数取得したいので、複数のレコードを取得するためのREST APIを呼出します。
kintoneの公式ドキュメントを参考にして、URLやパラメータを設定していきます。

https://cybozu.dev/ja/kintone/docs/rest-api/records/get-records/

公式ドキュメントを参考にしつつ、今回は以下のようなリクエストAPIを作成します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/dc1051fa-adbe-7340-aeb3-ca850039d4b5.png" width="700">

なお、事前に以下3つの変数を設定しておきます。

- `baseUrl`：kintone REST API用のベースURL
- `appId`：最初に登録したアプリのIDを入力
- `apiToken`：先ほど発行したトークンを入力

URL部分の`?`以降のクエリパラメータは、下の「クエリパラメーター」に項目を追加していけば自動的にURLのほうにも反映されます。

今回設定したクエリパラメータでは、
2024/12/17以降でレコード番号が大きい上位10件のデータを取得する
といった形になります。

ヘッダー側にはAPIトークンを以下のように設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e680529a-2e53-5a4b-1a83-97491542119e.png" width="600">

「送信」ボタンを選択して、`200 OK`のレスポンスが表示されれば成功です。
レスポンスデータの一部を記載します。
この応答結果を参考にして、次にGrafanaからkintoneデータの取得を試します。

```json
{
    "records": [
        {
            "レコード番号": {
                "type": "RECORD_NUMBER",
                "value": "56"
            },
            "時刻": {
                "type": "TIME",
                "value": "14:00"
            },
            "出退勤ステータス": {
                "type": "DROP_DOWN",
                "value": "出勤"
            },
            "従業員名": {
                "type": "SINGLE_LINE_TEXT",
                "value": "佐藤 幸子"
            },
            "日付": {
                "type": "DATE",
                "value": "2024-12-22"
            }
        },
        ・・・
        {
            "レコード番号": {
                "type": "RECORD_NUMBER",
                "value": "47"
            },
            "時刻": {
                "type": "TIME",
                "value": "13:00"
            },
            "出退勤ステータス": {
                "type": "DROP_DOWN",
                "value": "出勤"
            },
            "従業員名": {
                "type": "SINGLE_LINE_TEXT",
                "value": "鈴木 達夫"
            },
            "日付": {
                "type": "DATE",
                "value": "2024-12-21"
            }
        }
    ],
    "totalCount": null
}
```

### Grafana Cloudでのkintoneデータ取得

あたらめてGrafanaを開いて、今度はメニューから`Data source`を選択し、先ほどと同様にInfinityを検索して表示されるアイコンを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f98e7422-8e5b-731c-1673-e525f2bf3919.png" width="700">

`Settings`の項目でkintoneデータと接続するための各種設定を行います。

`Authentication`の項目では、認証用の設定を行います。
先ほどPostmanでやったように、`X-Cybozu-API-Token`キーのバリューにAPIトークンを設定します。
追加先は`Add to Header`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/47087830-dd22-5ebd-fb5b-a8fbd7c256b8.png" width="700">

もう一つ、画面下部にある、`Allowed hosts`に接続するkintoneのURLを入力します。
たとえば、`https://xxxxxxx.cybozu.com`という感じです。

`URL, Headers & Params`にはkintoneのBase URLを入力しておきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c70fdd82-fde7-382c-62f8-4001f7be2145.png" width="700">

ここまでで、データソースの設定ができたので、実際にデータの取得を試します。

ビジュアライズの追加を選択し、各項目を入力していきます。
Postmanで設定した入力項目を参考にしつつ、`URL Query Params`の各項目を入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a5109aba-f4e7-4b10-2747-ae44ecfbbe2c.png" width="700">

設定画面をそのまま下にスクロールし、`Rows/Root`、`Columns`の項目を以下のように設定します。
これは、Postmanで試した結果を基に、`"type"`を取り除いて`"value"`の値だけを取り出すための処理になります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/415214a7-ac35-d6d7-eb34-4cdea620eecf.png" width="700">

最後に、画面情報の`Table view`を有効にして、データが表形式で表示されていればデータが正しく取得できています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5fd66ef6-97a1-3145-8285-72039b9cbe88.png" width="800">

### ダッシュボード作成

最後に、Grafana上でダッシュボードを作成します。

まずは、下側のステータスタイムラインのほうから試します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/638fae12-3b21-3f88-a268-44a53c4718b3.png" width="500">

ビジュアルは`State timeline`を選択します。
`Transformation`タブを開いて、

1. 日付データと時刻データを結合して日時データを作成
2. 表示するフィールドを「従業員」と「ステータス」、「日時」に限定
3. 従業員ごとに分割

といった処理を順に行っています。

**1.日時データの作成**では、`Add field from calculation`と`Convert field type`を実行します。
`Add field from calculation`では、既存フィールドを結合するなどして、新規のフィールドを作成できます。
今回は、日付フィールドと時刻フィールドを文字列結合して、**日時**フィールドを作成しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/16580cd4-15d3-05dc-2af2-edcf25133ef3.png" width="700">

処理後のテーブルビューを確認すると、右側に**日時**フィールドが作成できていることがわかります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/acbfb59b-6fb2-f504-0d37-e431cec8deac.png" width="700">

作成した日時フィールドはこのままだと文字列型になってしまうので、`Convert field type`処理で型をにt時形式の型に変換します。

日時フィールドを指定して、型を`Time`に変更します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c2c7b642-b916-b099-6be8-bfe71cb19612.png" width="700">

処理後、日時フィールドの型が変わっていることを確認します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b8e4fa11-4bfa-8948-6f7b-27ca50bb6e1d.png" width="700">

**2.表示するフィールドの限定**では、`Organize fields by name`を使用します。

現在、表示されているフィールド一覧から非表示にするフィールドの👁️アイコンをオフにします。
なお、今回は変更していませんが、フィールド名をリネームすることもできます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1703c013-bcc1-e6e2-2c29-58f6531e2614.png" width="700">

処理後に表示されるフィールドが酸くなっていることが確認できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d02ef83d-aa77-f962-ec5a-a85e9571d48f.png" width="700">

**3.従業員ごとに分割**では、`Partition by values`を使います。

今回は、従業員ごとにステータスバーを分けたいので、従業員フィールドを指定します。
なお、右上にもあるように、この処理は**Alpha**版になるので、動作が不安定な可能性があります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0f0aadca-83df-899b-3858-77df6f047eb5.png" width="700">

テーブルビューで確認すると、従業員ごとにデータが表示されていることが確認できます。
従業員の項目を選択すると、別の従業員のデータに切り替えられます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/76c9498c-7fb5-ff0a-cfd2-65cf8558be4c.png" width="700">

ここまでで、データの変換処理は終わったので、ビジュアルの設定を行います。

表示は`State timeline`を選択し、`Value mapping`の設定を行います。
現状、出退勤ステータスは出勤と退勤という文字列情報のため、これを扱いやすくするために、

- 出勤は1
- 退勤は0

となるようなマッピング処理を行います。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/416064a3-b9e1-151c-4bc6-f846f886a608.png" width="350">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/26309c33-e607-a228-5997-9fe34eff95ec.png" width="700">

あとはバーの太さや透明度などを好みの設定にしてあげれば最初に示した画像と同じような表示が可能です。

次に、上側のステータスカードについても表示を行っていきます。
ビジュアルは`Stat`を選択します。
`Transformation`タブを開いて、

1. 日付データと時刻データを結合して日時データを作成
2. 「日時」フィールドでソート
3. 従業員ごとグループ化してそれぞれの最新のステータスを取得

といった流れでデータ変換処理を行っていきます。

**1.日時データの作成**は先ほどと同じ処理のため省略します。

**2.「日時」フィールドでソート**では、`Sort by`を使います。
テーブルビューをONにし、最新日時のデータが上側に来ることを確認します。
逆に古いデータが上に来た場合は、右側の`Recerse`をONにしてみてください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/dab33fea-9d98-c521-1b56-b11195cdb077.png" width="800">

**3.従業員ごとグループ化して最新のステータスを取得**には、`Group by`を使用します。
`Group by`でグループ化するフィールドは従業員フィールドを使用し、ステータスフィールドでは、最初の値1個を抽出するように設定しています。
こうすることで、事前にソートしていたデータの最初の値（最も新しい日時のデータ）を取得する形になります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ca31aaf9-f607-5bab-5c42-75a5fe86e0d0.png" width="800">

テーブルビューで確認すると、従業員名とステータスのみのデータ構造になっていることがわかります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f5cc974d-a7cf-777d-4830-f6fb215cea4a.png" width="800">

最後にこちらも、ビジュアルの設定を行います。

表示は`Stat`を選択し、先ほどと同様`Value mapping`の設定を行います。
今回は以下のように設定しています。

- 出勤⇒在室
- 退勤⇒不在

各社員の最終出退勤ステータスを取得しているため、出勤＝在室という前提にしています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/293c761d-c17d-e4f5-c498-fe5c8060021a.png" width="800">

また、`Fields`や`Text mode`の設定は以下の画像のように設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/eb4b94ef-b0b9-fd0b-9753-2de25b44d69a.png" width="350">

ここまでで、ダッシュボードの設定ができたので、保存します。

動作確認の方法については、実際にkintoneアプリ上から出勤or退勤データを追加し、その追加データがダッシュボード上に表示されているかをか確認すればOKです。（今回は画像キャプチャなどは省略します）

## ◇おわりに

今年のアドベントカレンダー内で主に使用した、Grafanaとkintoneを組み合わせた記事を最後に書くことができました。

kintoneデータの可視化の手段の一つとして、参考になれば幸いです。

# 🔚END
