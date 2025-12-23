---
title: NFCタグをトリガーとしてiPhoneショートカットを動かす
tags:
  - iPhone
  - ショートカットApp
  - RPA
  - 自動化
  - NFC
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

本記事は🎄[RPA Advent Calendar 2025](https://qiita.com/advent-calendar/2025/rpa)🎄21日目の記事です。

https://qiita.com/advent-calendar/2025/rpa

今回はNFCタグを使ったiPhoneショートカットの実行方法についてまとめたので、紹介します。

## ◇背景

こちらは私が今までに書いた、iPhoneショートカット関連の記事のストックリストです。

https://qiita.com/yankee/stocks/cc37c3a8ddda30f7d424

過去の記事でも少し紹介していますが、iPhoneショートカットでは特定の条件をトリガーとして、iPhoneショートカットを実行する**オートメーション**機能があります。

https://support.apple.com/ja-jp/guide/shortcuts/apd690170742/8.0/ios/18.0

トリガーの種類には、Wi-FiやBLEの接続、充電の開始などがあります。
このほかに、NFCを検出した時もトリガーとすることが可能です。

そこで、今回は安価に手に入るNFCタグを使用し、このNFCタグの検出をトリガーとしてiPhoneショートカットを実行する方法について備忘録も兼ねてまとめてみました。

## ◇環境等

- 使用デバイス
  - iPhone SE3
    - OS：iOS 18.7.1
  - 使用アプリ
    - ショートカットアプリ
  - NFCタグシール
    - 無地円形27mm
    - 通信方式：NFC Forum Type2 ISO/IEC 14443A

## ◇試した手順

### NFCタグの準備

今回使用したNFCタグシールはこちらを利用しました。

https://www.akiba-led.jp/product/1561

Amazon等で販売されているNFCタグシールでも動作するとは思いますが、NFCの規格は色々種類があるようなので、各自でご確認ください。
（NFC規格については以下の記事で詳しく解説されています）

https://qiita.com/gpsnmeajp/items/49db2212e632869edaf8

### テスト用のiPhoneショートカットを作成

NFC読み込みをトリガーとして実行するためのショートカットを作成します。
今回は簡単に、以下のような「何もしない」ショートカットを作成しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/df53a184-dec7-4568-9b12-2dc89d8c4f4c.png" width="300">

ショートカットの詳細や作成方法等については公式のドキュメントや私の過去記事を参考にしてください。

https://support.apple.com/ja-jp/guide/shortcuts/welcome/ios

https://qiita.com/yankee/items/356c0216347650af67e0#%E6%96%B0%E8%A6%8F%E3%82%B7%E3%83%A7%E3%83%BC%E3%83%88%E3%82%AB%E3%83%83%E3%83%88%E3%81%AE%E4%BD%9C%E6%88%90

### 1️⃣iPhoneショートカットのオートメーションを使う方法

NFCをトリガーとする方法ですが、今回2つの方法を紹介します。

まず1つ目は、前述した、ショートカットアプリ内にある「オートメーション」を使う方法です。

https://support.apple.com/ja-jp/guide/shortcuts/apd690170742/8.0/ios/18.0

> パーソナルオートメーションはショートカットに似ていますが、手動で起動するのではなくイベントが起こると作動します。

と記載があるように、特定のイベントをトリガーとして、ショートカットを実行できる機能です。

オートメーションはショートカットアプリ上から画面下にあるアイコンをタップすることで作成できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0d6cf391-8869-4beb-a1a4-f792fcef1a15.png" width="500">

このオートメーションのメニューから右上の`+`をタップすると、トリガーとするイベントを選択できます。
今回はその中にある「NFC」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f0be1f47-99b0-4009-a11b-0c07d7c08c48.png" width="300">

NFCタグの「スキャン」をタップすると、NFCタグのスキャンモードになるので、今回準備したNFCタグを読み込ませます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9cfb70a7-d95e-4b06-bfa9-ae654b6548af.png" width="300">

スキャンが成功すると、読み込んだNFCタグを区別するための名称を入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9cbfc50a-b32c-4fdc-be01-957f6826b8f0.png" width="400">

NFCタグを登録すると、実行するアクションを選択する画面になるので、今回は先ほど作成したショートカットを実行するように設定します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/79ac81b9-b860-4062-996f-75be5dee9561.png" width="400">

これで、準備は完了です。
先ほど読み込んだNFCタグシールにiPhoneを近づけて、下の画像のようなポップアップがでれば正しくオートメーションが動作しています。
※今回は、オートメーションの実行を「確認後に実行」にしていたため、実行を押さないとショートカットが動作しません

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/34d15008-9ecb-4c58-a069-8263359c5b95.png" width="500">

### 2️⃣サードパーティー製のNFC読み書きアプリを使う方法

もう1つのやり方は、NFCタグの読み書きアプリを使う方法です。

今回使用したアプリは「NFC Tools」というアプリです。
このアプリを使うことで、NFCタグにデータの書き込みが可能です。

https://apps.apple.com/jp/app/nfc-tools/id1252962749

:::note warn
**注意**
App Storeから提供されているアプリですが、インストールの判断は各自でお願いします
:::

アプリを開いたら、トップ画面から「書く」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7a37990c-86be-46d4-861e-94c8926925d7.png" width="300">

書き込み用メニューから「レコードを追加」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8a57af3c-63cc-4772-933a-93329f7ea8ff.png" width="300">

書き込むレコード種類の選択リストが表示されるので、今回は画面一番下の「ショートカット」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/09a121ec-dba7-4c4b-9f75-f3c8a3d09350.png" width="300">

「ショートカット名を入力」の項目に、先ほど作成したショートカット名を入力します。
※アプリのメモにも記載がある通り、ショートカットアプリに設定した名前と同じ名前にする必要があります

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d8925952-341b-457a-be94-91c199481e1d.png" width="400">

ここまで完了したら、書き込み用メニューのトップ画面に戻って、「書き込み」をタップします。
スキャンモードになるので、NFCタグシールにiPhoneを近づけることで書き込みが完了する・・・・はずでした。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/830e0787-3338-4a95-876c-8796c7aedb6c.png" width="300">

#### 書き込みエラーが発生

ここまでのやり方でNFCタグへ書き込みを試すと、エラーが発生しました。
エラー内容も書き込みが失敗したという意味合いのエラーで原因がパッと見でわからず。

ここで、長年の経験？からとりあえず日本語（マルチバイト文字）をやめようと思って試したところ、すんなり書き込めました。
以下の画像が書き込みに成功した時の設定です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/241a485a-516a-4ffa-a563-d4ef3a046c3f.png" width="300">

キャプチャは割愛しますが、iPhoneショートカット側のショートカット名称を「Nothing to do」に変更し、NFC Toolsアプリ側のショートカット名称も同様に修正しています。

:::note info
「NFC Toolsアプリ」でショートカットを呼び出す際は、
ショートカット名を日本語（マルチバイト文字）ではなく半角アルファベットを使うようにしましょう。
:::

こちらも先ほど同様、書き込みを行ったNFCタグシールにiPhoneを近づけて、下の画像のようなポップアップがでればショートカットが実行されています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a94dbac3-a401-4900-911f-6c1add102ac8.png" width="400">

なお、こちらのやり方の場合、実行時に確認をタップさせるような動作は設定できなそうでした。

## ◇おわりに

今回は、NFCタグとiPhoneショートカットの連携について紹介しました。

具体的なソリューションや事例はこれから何か考えていきたいと思いますが、どなたかのご参考になれば幸いです。

RPA Advent Calendarの他の記事もぜひご覧ください。

https://qiita.com/advent-calendar/2025/rpa

## 🔚END
