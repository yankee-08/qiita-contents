---
title: iPhoneが80%充電されたら、自動的に給電を止めるようにする
tags:
  - 自動化
  - SwitchBot
  - iPhone
  - AzureFunctions
  - AdventCalendar2023
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

本記事は[RPA（ロボティック・プロセス・オートメーション） Advent Calendar 2023](https://qiita.com/advent-calendar/2023/robotic_process_automation)の12日目の記事です。

https://qiita.com/advent-calendar/2023/robotic_process_automation

今回は、iPhoneとSwitchBotデバイスを組み合わせて、iPhoneの充電を80%超えたら停止させるといった処理を自動化させてみました。

## ◇開発環境等

- 使用バイス：
  - iPhoneSE2
  - SwitchBot プラグミニ（JP）
- 使用アプリ・ツール：
  - Azure Functions
  - iPhoneショートカットアプリ
  - iPhone App「Scriptable」

## ◇作成した背景

iPhoneに限らずですが、スマホの過充電・過放電は良くないと言われています。
そのため、iPhoneでもiOS13以降では、「バッテリー充電の最適化」といった機能によってiPhoneがフル充電される時間をできるだけ減らすようにすることができます。

https://support.apple.com/ja-jp/108055

また、iPhone 15では、さらに「上限80%」という項目が追加され、充電を80%で止めることもできるようになったようです。以下の文章は先ほどのサイトからの引用文です。

> iPhone 15 モデルの「上限 80%」について
>
> iPhone 15 モデルでは、「バッテリー充電の最適化」「上限 80%」「なし」のいずれかを選択できます。
> 「上限 80%」を選択した場合、iPhone はバッテリー残量が 80 パーセントになったところで充電を停止します。バッテリーの残量が 75 パーセントを下回ると充電が再開し、再び 80 パーセントになると停止します。
>
> 「上限 80%」を有効にしている場合も、iPhone はバッテリーの充電状態の推測精度を維持するため、ときどき 100 パーセントまで充電されます。

そこで、今回はiPhone 15**以外**のモデルで擬似的に同様のことができないか試してみました。

## ◇実装

### 構成

まず、iPhoneを充電するコンセントに、SwitchBot社のSwitchBotプラグミニ（JP）を繋ぎます。

https://www.switchbot.jp/products/switchbot-plug-mini

1. SwitchBotプラグミニ（JP）経由でiPhoneを充電する
2. iPhoneの充電が80%を超えると、ショートカットアプリのオートメーションが走る
3. オートメーションのフロー内で、iPhoneアプリの「Scriptable」を起動する
4. 予め「Scriptable」内で作成しておいた、Azure FunctionsにHTTPリクエストを発行するスクリプトを実行する
5. Azure Functionsはリクエストを受け取ると、SwitchBot APIを使用して、SwitchBotプラグミニ（JP）の給電を停止する

といった流れになります。
この後に、詳細を説明していきます。

### iPhoneショートカットアプリで、オートメーションを作成する

:::note info
実際の実装順とは異なりますが、イメージがしやすいように、処理が行われる順序で説明しています。
実装する際は逆から実装していかないと、テストがしにくいので注意ください。
:::

ショートカットアプリは公式サイトには以下のような説明があります。

> ショートカットとは
>
> ショートカットは、アプリでの1つまたは複数の作業を素早く完了するための機能です。ショートカットアプリでは、複数の手順を組み合わせた独自のショートカットを作成できます。例えば、海の波情報を取り込み、ビーチまでの所要時間をチェックし、サーフミュージックのプレイリストを再生する、という「サーフタイム」ショートカットを構築できます。

https://support.apple.com/ja-jp/guide/shortcuts/welcome/ios

まず、ショートカットアプリを開き、オートメーションを選択します。次に、右上にある➕ボタンをクリックして、新規作成します。オートメーションの種類は、「個人用」と「ホームハブ」の2種類がありますが、今回は個人用を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e734d053-17bb-3097-f4d8-90e7eaaee302.jpeg" width="300">
　　　　　　
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/10c5daa5-24f4-956b-b38d-625265d4692f.jpeg" width="300">

次に、オートメーションを実行するトリガを選択します。
「バッテリー残量」を選択し、「80%より上」に設定して「次へ」をクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9a715f5d-045d-e4f0-4181-d2b7a719fc57.jpeg" width="300">
　　　　　　
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/92ca944c-bc13-7276-ed25-00d4f85e1be9.jpeg" width="300">

トリガ条件を設定したので、次に実行するアクションを入力していきます。
「アクションを追加」を選択後、「Scriptable」アプリを選択します。
※「Scriptable」アプリの実装内容については次の項で説明します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/562efee3-9ef9-e773-8403-5a5c03dce538.jpeg" width="300">
　　　　　　
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d45694d3-1191-ffad-83e9-a51bea5e30b8.jpeg" width="300">

「Run Script」内で、予め作成しておいたスクリプト（今回は「給電オフ命令」スクリプト）を選択します。今回は、スクリプト実行時のパラメータは特に指定せずに「次へ」をクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0a11d8d0-26d8-bbde-104b-bbca143076cb.jpeg" width="300">
　　　　　　
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/953de53d-a18a-13d2-be33-2fe01e887000.jpeg" width="300">

最後に、以下の設定にして完了をクリックします。
「実行の前に尋ねる」項目はオフにしておかないと自動で給電が停止されないため必須ですが、「実行時に通知」項目は好みで選んで問題ないと思います。

- 実行の前に尋ねる：オフ📴
- 実行時に通知：オン🔛

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/278a91dc-9a4f-3914-1a07-5eb32161cd9f.jpeg" width="300">

### 「Scriptable」アプリからAzure FunctionsにHTTPリクエストを発行する

ショートカットアプリから呼び出す「Scriptable」アプリ上にスクリプトを追加していきます。

「Scriptable」アプリは公式ページに以下の記載がひと言で書かれており、iOS上でJavaScriptプログラムを走らせることができる非常に単純明快なアプリです。

> Automate iOS using JavaScript

https://scriptable.app/

:::note warn
**アプリのダウンロードの判断については、自己責任でお願いします。**
:::

まず、アプリを立ち上げて、「Scripts」側を選択した状態で右上の➕ボタンをクリックして、新規作成します（左下のキャプチャ画面は既にスクリプトがありますが、これは無視してください）。
なお、今回は使用しませんが、「Gallery」を選択すると、色々なスクリプトサンプルがありますので、これをベースにして、修正していくことも可能です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ba53f4fb-f27a-4fb7-346e-e29bc9599ee7.jpeg" width="300">
　　　　　　
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ad672bc7-7e34-d4e4-7e04-b940576933fe.jpeg" width="300">

➕ボタンをクリックすると、Blank状態の画面が表示されますので、ここにコードを書いていきます。
左下の「📄」ボタンをクリックすると、Documentationを閲覧することもできます。なお、Documentationは、以下のサイトでも閲覧可能です。

https://docs.scriptable.app/

iPhone上で直接コーディングするのは現実的ではないため、PCのエディタ上で作ったコードを持ってくるなりすることをお勧めします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/72577870-58df-0e6c-b7c9-3cb3564e32d2.jpeg" width="300">
　　　　　　
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/94b24099-1f9a-dd58-2e19-471943bed8c7.jpeg" width="300">

今回、「Scripts」では、Azure FunctionsにHTTPリクエストを発行するスクリプトを作成します。
HTTPリクエストを送るには、左下画像にある`Request`クラスを使用します。

最低限のコードですが、実装したものを以下に記載します。
`url`の部分はこの後説明するAzure Functionsで生成されるURLを入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a28b7606-b973-10a3-20a9-ac148a2a0c80.jpeg" width="300">

```JavaScript
let url = "**********";
let req = new Request(url);
req.method = 'GET';

let res = req.loadString();
console.log(res);
```

最後に、スクリプト名をわかりやすい名称に設定して完了です。
前述のショートカットアプリでは、ここで設定したスクリプト名を選択するようにします。

### Azure Functionsの作成

Azure Functions上でSwitchBot APIを制御する関数を追加します。
まず、Azure上に環境を構築していきますが、具体的な手順は公式ドキュメントの通りのため、割愛します。
参考とした公式ドキュメントのURLを以下に載せておきます。

https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-function-app-portal

https://learn.microsoft.com/ja-jp/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-decorators

なお、今回こちらで作成した環境条件は以下のようになります。
`Python`を選んでいますが、自分が使いやすいプログラミング言語で問題ありません。

- 地域：Japan East（東日本）
- OS：Linux
- プログラミング言語：Python（v2）
- ホスティング オプション：従量課金プラン
- テンプレート：HTTP trigger
- 承認レベル：function

実装に当たっては、以前書いたPostmanの記事を参考にしています。

https://qiita.com/yankee/items/c31b09811c1e27956a62

Postman上でSwitchBotデバイスを制御するAPIリクエストを作成した後、コード出力する機能を使って、Python用のコードを出力しています。
コード生成については、Postmanのドキュメントを参照ください。

https://learning.postman.com/docs/sending-requests/generate-code-snippets/

また、生成されたコードとこちらの記事も参考にしつつ、Azure Functions上に関数を作成しました。

https://qiita.com/Chroma7p/items/8e1817d313beca2b2641

本来であれば、ここに作成したPythonコードを載せるところですが、Advent Calendar 2023に間に合わせるために突貫でプログラミングした関係で、トークンや各種IDをコードに直接埋め込み、HTTPレスポンスが正しく返ってこない不具合などがあるため、コードは後日追記したいと思います😅。

関数の作成が完了すると、Azure上から、関数URLの取得ができるようになるため、前述の「Scriptable」アプリのスクリプト上にそのURLをコピペします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b036aba1-8b3e-4e2d-3ed8-11d68c00ceb6.png" width="800">

### テスト

最後に、

- ショートカットアプリのオートメーション
- 「Scriptable」アプリのスクリプト
- Azure Functionsの関数

を全て有効化した状態で、実際にSwitchBotプラグミニ経由でiPhoneを充電してみます。

バッテリーが81%に充電された状態で以下の通知が表示され、SwitchBotプラグミニの給電がオフされていることを確認できました。
なお、「**オートメーションを実行できませんでした**」という通知が出ていますが、これは先ほど書いたように、Azure Functions上のコードが正しい応答を返していないためです。実際には、SwitchBot APIは正しく呼び出せています（後日要修正）。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4a2093fa-a9ee-4536-3f10-bb2517924f03.jpeg" width="600">

何度かオートメーションを実行させた後のiPhoneバッテーリ状態のキャプチャ画面になります。赤枠で囲んだ部分がオートメーションを有効化した部分になり、想定通りに動作していることが確認できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6b419e01-7472-e57f-0d4f-75ea152fecc7.jpeg" width="800">

## ◇おわりに

とりあえず目的の自動化は行えましたが、他にも色々考慮すべき点があり、今後の課題です。

＜今後の課題＞

- Azure Functions上の関数が未完成
⏩もう少しAzure Functions学んだら作り直す
- SwitchBotプラグミニが給電オフ時にこのオートメーションが動作するとエラーになる
⏩Azure Functions上で現在の状態を取得する処理が必要
- SwitchBotプラグミニ経由で充電していない場合（外出先など）でもオートメーションが動作する
⏩GPSや接続しているWi-Fi情報での判断分岐である程度対応可能？

といった感じです。少しRPAとは離れるかもしれませんが、自動化といえなくはないので、[RPA（ロボティック・プロセス・オートメーション） Advent Calendar 2023](https://qiita.com/advent-calendar/2023/robotic_process_automation)に掲載させて頂きました。

何かしらの参考になれば幸いです。

# 🔚END
