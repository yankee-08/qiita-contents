---
title: 神Excelをそのままデジタル化！セルフホスト型n8nで実現する連携事例（非推奨）
tags:
  - Excel
  - Docker
  - DX
  - kintone
  - n8n
private: true
updated_at: '2025-11-29T14:08:32+09:00'
id: 432bb58fc99478b1133b
organization_url_name: null
slide: false
ignorePublish: false
---

## ◇はじめに

本記事は[Excel業務をDX化したい。あなたならどうする？ by MESCIUS Advent Calendar 2025](https://qiita.com/advent-calendar/2025/mescius)
2日目の記事です。

https://qiita.com/advent-calendar/2025/mescius

今回は、【**Excel業務をDX化したい。**】というテーマに沿って、神Excelをデジタル化するというネタを紹介します。

## ◇背景

### 神Excelとは？

**神Excel**という言葉を聞いたことはありますか？
正式な定義は多分ありませんが、

- セルの縦横比が方眼紙状
- 入力したものをそのまま印刷することを想定したレイアウト
- そのまま印刷して手書きで記入することも想定されている

といった感じのExcelファイルで、主に官公庁の申請書等で使われることが多い印象です。

https://wa3.i-3-i.info/word18772.html

https://e-words.jp/w/%E7%A5%9E%E3%82%A8%E3%82%AF%E3%82%BB%E3%83%AB.html

### 別サービスとの連携が難しい

神Excelはそのレイアウトの関係でデータの抽出が難しく、他のクラウドサービスとの連携が難しいことが多いです。
そのため、１項目ごとに手作業で転記するといった作業が発生することがあり、手間がかかります。

### 何とか別サービスと連携させたい

後述しますが、本来の対策は神Excelを使わないようにすることです。
ただし、~~それだと記事にならないので~~実際には自社の判断だけではどうにもならないケース（フォーマット自体を決めているのは、別会社のパターン）も多く、改善が難しい場面もあります。

そういった場合、Excelのフォーマットはそのままにしつつ、入力データを別システムに別途連携させる方法が考えられます。

神Excelを使いつつ、その入力データを自社内のシステムに連携させたい場合、

1. システムに入力したデータから神Excelに転記する
2. 神Excelに入力したデータをシステムに転記する

のどちらかの方法が考えられますが、
今回は、2番目の方法をn8nを使って実現した事例を紹介します。

## ◇環境等

- Microsoft Excel
  - Microsoft365 Business Standard
  - デスクトップ版
- n8n
  - ホストOS
    - Proxmox VE 8.3
  - VM
    - Ubuntu 24.04.3 LTS
    - Docker version 28.1.1+1
    - Docker Compose version v2.33.1
    - n8n: 1.118.2
- kintone
  - kintone開発者ライセンス

## ◇実装手順

### n8nのセットアップ

#### n8nとは

今回、Excelを処理するのに、n8nというサービスを使用しています。
n8nはいわゆるiPaaS（Integration Platform as a Service）に分類されるサービスの1つで、最近ではAIワークフローと呼ばれたりもしてます。
これらのサービスを使うことで、様々なクラウドサービスを連携することができます。

https://n8n.io/

n8nの特徴の1つとして、セルフホスト型での利用が可能な点があります（Community版では無料で利用可能）。
そして、このセルフホスト型n8nはオンプレデータに対するフローの実行も可能なため、オンプレミス上のExcelファイルの処理も行えると考え、本ツールを採用しました。

#### n8nのインストール手順

セルフホスト型のn8nについては、ローカルのPC上にインストールする形としました。
今回はDockerを使ってn8nをインストールしましたが、Docker環境までの構築手順は本記事では割愛します。

以前、別記事にてProxmox VEのインストールからVMの構築までをまとめていますので、必要に応じてそちらをご覧ください。
（Ubuntuインストール時にインストールするパッケージでDockerを選択することでDockerのインストールが可能です）

https://qiita.com/yankee/items/1d576f7a25d6f33c6cb5

https://qiita.com/yankee/items/495e80193070f6e70b65

基本的には、公式ドキュメントやGitHubの手順に従いつつ、生成AIにもサポートしてもらいながら以下の`compose.yaml`を作成しました。

https://docs.n8n.io/hosting/installation/server-setups/docker-compose/

https://github.com/n8n-io/n8n-hosting/tree/main/docker-compose/withPostgres

```yaml
services:
  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_POSTGRESDB_USER}
      POSTGRES_PASSWORD: ${DB_POSTGRESDB_PASSWORD}
      POSTGRES_DB: ${DB_POSTGRESDB_DATABASE}
      TZ: ${TZ}
    volumes:
      - ./postgres:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    depends_on:
      - db
    ports:
      - "5678:5678"
    environment:
      TZ: ${TZ}

      N8N_SECURE_COOKIE: ${N8N_SECURE_COOKIE}
      N8N_PROTOCOL: http

      # 認証
      N8N_BASIC_AUTH_ACTIVE: ${N8N_BASIC_AUTH_ACTIVE}
      N8N_BASIC_AUTH_USER: ${N8N_BASIC_AUTH_USER}
      N8N_BASIC_AUTH_PASSWORD: ${N8N_BASIC_AUTH_PASSWORD}

      # DB 設定
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: db
      DB_POSTGRESDB_DATABASE: ${DB_POSTGRESDB_DATABASE}
      DB_POSTGRESDB_USER: ${DB_POSTGRESDB_USER}
      DB_POSTGRESDB_PASSWORD: ${DB_POSTGRESDB_PASSWORD}
      DB_POSTGRESDB_PORT: 5432

    volumes:
      - ./.n8n:/home/node/.n8n
      - ./local-files:/files
```

`compose.yaml`内に記載されている変数（`${DB_POSTGRESDB_USER}`など）は同フォルダ内に`.env`ファイルを作成し、そのファイル内で別途定義しています。

ファイル作成後、以下のコマンドを実行してn8nを起動します。

```bash
docker compose up -d
```

しばらくすると、n8nが起動するので、今回使用しているVMのIPアドレスにアクセスします。

```http
http://{{vm_ip_address}}:5678/
```

n8nの画面が表示されれば、とりあえずOKです。
具体的なn8nでのフロー作成については、別途後述します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d8005ddf-27ae-48d3-8884-48c4d811ba26.png" width="600">

### Excelファイルの準備

次に、n8nで処理するためのExcelファイルを準備します。

実際に使われているExcelを使ってもよいのですが、今回はせっかくなので、生成AIを使ってそれっぽいフォーマットを作成してもらい、一部手直ししています。

作ったExcelファイルは以下のような感じです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8f66bd74-e642-4c09-9071-9bcbf29bee3e.png" width="600">

### n8nでのフロー作成

次は、実際に作成したExcelファイルをn8nで処理するためのフローを作成します。

実際に今回作成したフローの全体像を以下に示します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/bf8c1390-f8e7-4fb8-b569-1f1ea49ffa94.png" width="400">

それぞれのセクションを順に説明します。

#### Excelファイルを検出

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/de1e1ebe-b537-47ed-bc5c-6369c6d1ac98.png" width="800">

このセクションでは、指定したフォルダ内にあるExcelファイルが置かれたことを検出して、フローが開始されます。

`Local File Trigger`ノードにて、監視するフォルダを指定しています。
今回は、`/files/inbound/excel`フォルダを監視するように設定しています。
トリガが発生する条件として、ファイル追加や更新、フォルダ追加などを指定できますが、今回はファイルの追加時のみを指定しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/38aa3b18-1992-4cd5-aee7-535070bf97fa.png" width="400">

`Read/Write Files from Disk`ノードにて、追加されたファイルを読み込みます。
なお、この時点ではexcelファイルかのチェックは行っていないので、置かれたファイルの種類に限らず、読み込みます。
※当初、このノード内で拡張子チェックを行おうとしましたが、うまくいかなかったため、次の`Filter`ノードでチェックしています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d783d4fa-3beb-4d6d-a435-804a125ff8b2.png" width="400">

`Filter`ノードにて、読み込んだファイルの拡張子が`.xlsx`であるかをチェックしています。
`.xlsx`以外のファイルが置かれた場合は、ここでフローが終了します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4ee40ef4-5d1c-4b99-b5f8-d615b0c8bfb6.png" width="600">

#### Excelファイルからデータを抽出

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0bc71030-95b1-4487-91bb-6b0f1e69978a.png" width="500">

このセクションでは、読み込んだxlsxファイルから各項目のデータを抽出します。

最初に、`Extract From File`ノードを並列に並べていますが、これは各行ごとにデータを抽出するための処理になります。
本来のExcelファイルでは見出し行の下にデータ行が連続して並んでいることを想定していますが、今回は申請書様式の特殊な構成のため、各項目行ごとにデータを抽出する必要があります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/701bf680-e4b9-4536-af2b-a21a5e2a8be1.png" width="600">

氏名、生年月日、性別の項目が含まれている6-7行目の例を説明すると、`Range`のOptionsで`A6:AJ7`を指定しています。
これは、元のExcelファイルで、項目名と入力欄が含まれている範囲を指定しています。
`Sheet Name`欄は今回のExcelファイルのシート名をそのまま記載します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/be46676d-6121-4586-951b-33786aa352eb.png" width="300">

並列化されているフローの中で、一部`Edit Fields (Set)`ノードを配置しています。
Excelの日付データは1900年1月1日（又は1904年1月1日）を起点としたシリアル値で保存されるため、そのままではn8nで扱いにくいため、ここで変換を行っています。

Excelの日付システムについてはこちらのMicrosoft サポートサイトにも記載があります。

https://support.microsoft.com/ja-jp/office/excel-%E3%81%AE%E6%97%A5%E4%BB%98%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0-e7fe7167-48a9-4b96-bb53-5612a800b487#id0ebbh=newer_windows_versions

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1bd31820-cfd4-49ae-bcd1-9a370f11d53c.png" width="500">

並列化されたフローは、`Merge`ノードと`Aggregate`ノードで1つにまとめています。
`Merge`ノードは前段のノードすべての完了を待つ効果もあります。

https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.merge/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.merge

`Aggregate`ノードで1つの`data`オブジェクトにまとめています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c11317ff-b8ea-48c5-b046-17a8734fd988.png" width="800">

#### データをクラウドサービスに登録

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9be379eb-141d-4c61-97d7-f637774ff3c3.png" width="600">

このセクションで、先ほどまでに抽出したデータをクラウドサービスに登録します。
そのために、まずはデータを保存するクラウドサービスを用意する必要があります。
今回は、例としてkintoneを使用しています。

kintoneは、サイボウズ社が提供している業務アプリ作成用クラウドサービスです。

https://kintone.cybozu.co.jp/

今回はkintone開発者ライセンスを使って、アプリを作成しています。

https://cybozu.dev/ja/kintone/developer-license/

アプリの作成手順については、公式チュートリアルがあるので、今回は詳細は割愛します。

https://jp.cybozu.help/k/ja/app/setup/create_app/tutorial.html

今回作成したアプリの画面は以下のような感じです。
※縦長になってしまったので、スクリーンショットを分割しています

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5846c40d-e5d2-4968-870c-166c419fc6c0.png" width="800">

各項目のフィールドは以下のように設定しています。

| フィールド名                         | フィールドコード       | フィールドタイプ         |
|--------------------------------------|------------------------|--------------------------|
| 氏名（漢字）                         | name_kanji             | 文字列（１行）           |
| 氏名（フリガナ）                     | name_kana              | 文字列（１行）           |
| 生年月日                             | birthdate              | 日付                     |
| 性別                                 | gender                 | ドロップダウン           |
| 住所                                 | address                | 文字列（１行）           |
| 電話番号（携帯可）                   | phone                  | リンク                   |
| メールアドレス                       | email                  | リンク                   |
| 申請の種類                           | application_type       | ドロップダウン           |
| 提出先（部署名）                     | department             | 文字列（１行）           |
| 申請理由（できるだけ具体的に）       | application_reason     | 文字列（複数行）         |
| 処理期限                             | doc_date               | 日付                     |
| 関連番号（例：住民票コード等）       | reference_number       | 文字列（１行）           |
| 添付書類                             | attachment             | ドロップダウン           |
| その他資料詳細                       | attachment_detail      | 文字列（１行）           |
| 連絡希望手段                         | contact_method         | ドロップダウン           |
| 連絡可能時間帯（例：9:00-18:00）     | contact_time_range     | 文字列（１行）           |
| 署名                                 | signature              | 文字列（１行）           |
| 署名日                               | sign_date              | 日付                     |
| 担当者記入欄（受付番号など）          | staff_receipt_memo     | 文字列（１行）           |

フォームの配置が完了したら、最後にREST APIトークンを発行します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a0abaec1-a678-47bb-a69c-ba9340acd675.png" width="700">

kintoneのレコード登録手順については、公式ドキュメントがあるので、こちらを参考にしながら、n8nのフローを作成します。

https://cybozu.dev/ja/id/e7f3125797f78aaec154b918/#create-record

`Edit Fields (Set)`ノードでkintoneにデータを登録するために必要な各種パラメータを変数にセットしています。

ここで、セットしているのは、`base_url`と`appId`の2つで認証に必要な`X-Cybozu-API-Token`はここでは設定不要です。
この2つのパラメータはkintoneのドキュメントを参考にしながら、先ほど作成したアプリのIDと自分のkintoneドメインのURLを設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/83666d61-2ec6-4a71-9b87-68cd999d50cd.png" width="400">

次に、`HTTP Request`ノードで実際にkintoneにデータを登録しています。
ノード設定の上部で認証関係のパラメータを設定します。

`Authentication`、`Generic Auth Type`欄はそれぞれ下の画像のように設定した後、`Header Auth`項目右側`🖊`アイコンを選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/80b03d2a-65f1-4611-baed-364a4be5991c.png" width="600">

Header Authの詳細設定画面が表示されたら、`Name`欄に`X-Cybozu-API-Token`を入力し、`Value`欄に先ほど発行したAPIトークンを入力します。
この状態で`Save`を実行すると、Credentialが保存されますので、次回以降は今回の設定を再利用できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c64648ca-2ae8-42e5-be30-7021a74d846e.png" width="600">

`Body Parameters`欄では、kintoneのレコード登録APIの仕様に従って、登録するデータを順次設定します。
基本的に、`Name`欄はkintoneアプリのフィールドコードに合わせて入力し、`Value`欄にn8n内で抽出したデータをドラッグして設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/34df604a-e65f-4663-9f17-58b5aaa0fca1.png" width="500">

ここまで設定が完了したら、フローは完成です。

### テスト

最後に作成したフローをテスト実行します。
先ほどのフローの最初のノードの左側にある"⚡"アイコンにマウスを合わせると表示される`Execute workflow`をクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/88225b89-0c8b-4d0d-baf7-32a30d628a2b.png" width="500">

`Local File Trigger`ノードが実行待ちの状態となるので、事前に設定したフォルダ内にExcelファイルを配置します。
ファイル配置後、フローが動き出し、以下の画像のように、画面右下に`Workflow executed successfully`と表示されれば成功です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/fb3e640c-5d3d-478d-a7a2-4d31126b79ea.png" width="500">

実際に今回テストで使ったExcelファイルがこちらです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/dfecc607-5987-4ebf-88cc-6ab4bd5fc642.png" width="600">

そして、kintone側に登録されたデータがこちらです。
データが正しく登録されていることが確認できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8333a7da-1e5a-416b-a271-57784bedeefa.png" width="500">

これでテスト実行が確認できたので、画面右上からフローを`Active`に変更すればフローが常時実行されるようになります。

## ◇今回見送った内容

当初、実装する予定でしたが、解決策が時間内に見つからなかったため、見送ったものもあります。

### kintoneのチェックリスト形式での登録

当初、Excel側で○×形式で項目を入力させ、○が入っている項目のみ抜き出してkintoneに登録することを検討していました。
しかし、ループ処理での抜き出しがうまくいかず、今回は見送っています。

### 空白セルがあった場合の処理

入力項目によってはブランクのままの項目も実際にはあるかと思います。
ただし、今回のフローではすべての入力欄に何かしらの値が入っていないとエラーが発生します。
現状のフローですと、セルが空白の場合、そのセルのデータは取り出さない設定となっており、後段でkintoneにデータを登録する際にそのセルのデータが存在しないため、エラーとなります。

空白セルも含めて項目を取り出すように設定を変えることはできるのですが、その設定に変えた場合、セル結合で隠れた部分のセルが全て空白として取り出されてしまいました。
後段でのデータの整形が難しかったため、このような対応なっています。

それぞれ、今後の課題としていつか改善できればと考えています。

## ◇おわりに

冒頭でも書きましたが、根本的な解決策は**神Excelを使わないようにすること**であり、今回の方法は暫定的な非推奨の対策となります。
ただし、どうしても既存のExcelフォーマットを使い続けなければならない場合、今回のようにn8nのようなセルフホスト型ワークフローを活用することで、作業を効率化できる方法もあることを知っていただければ幸いです。

この後もアドカレの記事が続きますので、ぜひお楽しみください。

https://qiita.com/advent-calendar/2025/mescius

## 🔚END
