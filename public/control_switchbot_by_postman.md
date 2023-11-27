---
title: Postmanを使ってSwitchBotデバイスを制御してみた
tags:
  - WebAPI
  - Postman
  - SwitchBot
  - AdventCalendar2023
private: true
updated_at: '2023-11-28T01:09:35+09:00'
id: c31b09811c1e27956a62
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

本記事は[Postman Advent Calendar 2023](https://qiita.com/advent-calendar/2023/postman)の2日目の記事です。

https://qiita.com/advent-calendar/2023/postman

今回は、Postmanを使って、スマート家電デバイスのSwitchBotデバイスの制御を行うAPIリクエストのセットを作成しました。

## ◇開発環境等

- OS：
  - Windows 11 Home（Ver:22H2）
- ブラウザ
  - Edge（バージョン 119.0.2151.72 (公式ビルド)）
- 使用ツール：
  - Postman(Web版)
- 使用バイス：
  - SwitchBot ハブ2
  - SwitchBot ボット
  - SwitchBot 温湿度計
  - SwitchBot 防水温湿度計
  - SwitchBot プラグミニ（JP）
  - SwitchBot 開閉センサ

## ◇Postmanとは📮

[Postman Advent Calendar 2023]([Postman Advent Calendar 2023](https://qiita.com/advent-calendar/2023/postman))ページ内の概要の項目を一部引用します。

> Postmanは、APIを構築し利用するためのAPIプラットフォームです。Postmanを使うとAPIライフサイクルの各ステップを簡単に行えるようになり、コラボレーションが効率化されることで、開発者はより良いAPIを速く作成することができるようになります。

Postmanを使用すると、プログラミング言語や`curl`を使用せずにHTTPリクエストが発行できます（GraphWLやgRPCなどにも対応しているようです）。

Postman社員の方が過去に実施した際のセミナー資料なども公開されているため、この辺も参考にしてもらえればと思います。

https://postman.connpass.com/presentation/

## ◇SwitchBotとは🎚️

SwitchBot社が開発・販売しているスマート家電やホームオートメーションを実現するためのデバイス群です。
今回使用したデバイス以外にも、様々なデバイスが市販されています。
基本的には、スマホアプリを使用してデバイスデータの収集やデバイスの制御を行う仕様となっていますが、
GitHub上にWeb APIも公開されています（最新はVer1.1）。

今回は、このWeb API経由でデバイスの情報取得や遠隔制御を実現していきます。

## ◇最終的にできたもの

基本的に、GitHubに公開されている[SwitchBot API v1.1](https://github.com/OpenWonderLabs/SwitchBotAPI)の内容をベースにPostman上でAPIリクエストのセットを作成しました。
なお、手持ちにないデバイスについては、動作検証できなかった関係上、作成していません。

## ◇実装

実装手順について順にまとめていきます。
なお、このCollection自体はPostman上でも公開していますので、気軽に試したい場合はForkしてみてください。

https://www.postman.com/lunar-resonance-715260/workspace/switchbot-api-ws

### Workspaces、Collections、Environmentsの作成（Postman）

まず、専用の環境を作成していきます。
なお、Workspacesについては、今回は新規で作成していますが、最初に用意されている`My Workspace`を使っても問題ありません。

#### Workspaces

Postmanのトップページから、`Workspaces`をクリックし、続いて`Create Workspace`をクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c4c03efb-41d6-ee7a-eb88-a56bfd9ae8ec.png" width="500">

Workspace作成画面では、1ページ目にtemplatesの選択画面が表示されますが、今回は`Blank workspace`を選択しています。
2ページ目でWorkspaceの名前を適宜入力し、Workspaceのタイプで`Personal`を選択して、`Create`ボタンをクリックすれば完了です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7638e809-a8c1-3c37-89bc-f8226e41780f.png" width="300">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/94ef31fe-4fe0-63d4-7d04-0278721222e4.png" width="320">

#### Collections

今回はSwitchBot API用に1つのCollectionを作成しています。
`Collections`のメニューから`＋`ボタンをクリックし、続いて`Blank collection`を選択します。
すると、`New Colleciton`というCollectionが作成されるので、わかりやすい名称を入力します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ef586e4d-5161-78e4-452a-74103d4b6a52.png" width="500">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/73ca27e3-f9fb-e89c-6be7-629b7d5f5a28.png" width="500">

### Environments

Postmanではプログラミング言語と同じように、変数を扱うことができます。
詳細は前述のセミナー資料を確認もらえればと思いますが、

- グローバル変数
- コレクション変数
- 環境変数

の3種類の変数があります。
先ほど作成したCollection上に変数（コレクション変数）を作成することもできますが、
コレクション変数の場合、secret設定ができないことから、secret設定が可能な「環境変数」も併せて使用しています。
「環境変数」を作成したい場合、先ほど作成したCollectionとは別に、まずEnvironmentを作成する必要があります。
なお、EnvironmentはWorkspace内でのみ有効となりますので、Workspaceを跨いで変数を使用したい場合は「グローバル変数」を使用する必要があります。

作成方法は、先ほどのCollectionとほぼ同じです。
今度は、Postmanのトップページから`Environments`をクリックし、続いて`＋`ボタンをクリックします。
すると、`New Environments`というEnvironmentが作成されるので、わかりやすい名称を入力すれば作成完了です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/dfbe9508-8d08-78e8-98f8-9c696a8543cb.png" width="300">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/278b0f96-c506-376a-dff7-4f297ec7963d.png" width="500">

このあと、この作成した環境上に変数を追加していくのですが、その部分は後述します。

### 認証用Keyの発行（SwitchBot）

SwitchBot API v1.1を使用する場合、tokenとsecretが必要になります。
この2つはSwitchBotのスマホアプリ上から操作を行い、開発者向けオプションを表示する必要があります。

手順としては、[公式サイト](https://support.switch-bot.com/hc/ja/articles/12822710195351-%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E3%81%AE%E5%8F%96%E5%BE%97%E6%96%B9%E6%B3%95)に記載の通りです。
一応、以下に公式サイトから引用した手順を記載しておきます（公式サイト上には画面キャプチャも載ってます）。

> SwitchBot Appバージョン：V6.24以降
> 1．SwitchBotアプリを起動します。
>
> 2．プロフィール→設定→アプリバージョンまで進みます。
>
> 3．アプリバージョン(例えば 6.24)を数回（5回~15回）連続タップすると「開発者向けオプション」が出てきます。
>
> 4．「開発者向けオプション」をタップして、次の画面でトークン情報がございます。

表示されているトークンとクライアントシークレットのランダム文字列を後で使用しますので、
どこかにコピーしておきます。

### Variables（変数）の設定（Postman）

次に、変数化しておきたいパラメータを検討します。
Postmanの変数については、以下の記事が詳しいので、こちらも参考にしてください。

https://qiita.com/yokawasa/items/05913df60aea07395903

SwitchBot API v1.1の`README.md`内のAPI Usageを確認したところ、
ベースとなる[Host Domain](https://github.com/OpenWonderLabs/SwitchBotAPI#host-domain)があるため、こちらは変数化をしておきます。
また、Headerに入れる必要があるパラメータとして、[Request Header](https://github.com/OpenWonderLabs/SwitchBotAPI#request-header)の項目に以下のパラメータが列挙されているので、こちらも変数化しておきます。

- Authorization(token)
- sign
- t
- nonce

なお、Authenticationの項の[Python 3 example code](https://github.com/OpenWonderLabs/SwitchBotAPI#python-3-example-code)を確認してみると、先ほどのパラメータの1つである`Authorization`は前述したスマホアプリから発行したトークンがそのまま入っていることが分かります。
一方、アプリから発行したクライアントシークレットについては、`sign`を算出するために使用されているため、別途変数化する必要があると考えられます。この変数を`secret`とすると、全部で6つのパラメータの変数化が必要になります。

次に、それぞれのパラメータを3種類の内、どの変数の種類にするかについて考えます。

- グローバル変数
- コレクション変数
- 環境変数

今回は、一つのCollectionでしか使用しない変数のため、「グローバル変数」は除外しました。
まず、Host Domainについては、SwitchBot APIで共通の内容のため、「コレクション変数」としました。
次に、スマホアプリ上から発行するトークン（token）とシークレット（secret）については、秘匿したい情報となるため、変数タイプをsecretとしたいため、「環境変数」としました（どっちもsecretで分かりにくい・・・）。
残りの3つのパラメータ（sign,t,nonce）についてはその都度演算されるパラメータのため、どちらでも問題ないかと思いますが、今回は「コレクション変数」としました。
まとめると以下のようになります。

- グローバル変数
  - 未使用
- コレクション変数
  - Host Domain
  - sign
  - t
  - nonce
- 環境変数
  - token
  - secret

#### コレクション変数の追加

先ほど追加したCollectionを選択した状態で、`Variables`をクリックします。
`Variables`の項目に4つの変数を追加します。
なお、Host Domainについては変数名を`baseUrl`として、`Current Value`に`https://api.switch-bot.com`と入力しています。
一通り入力が完了したら、最後に右上の`Save`ボタンをクリックして完了です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ff9c5150-2c37-94ba-9851-abee585ddea7.png" width="800">

#### 環境変数の追加

まず、トップメニューから`Environments`をクリックし、先ほど作成した環境を選択します。
`Variable`として2つの変数を追加します。`Type`については、両方とも`secret`に設定します。
最後に、`Current value`にスマホアプリから発行した値を入力して`Save`ボタンをクリックします。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/59740131-9330-3326-49a5-01d800e90325.png" width="800">

#### 環境の有効化

環境変数を使用したい場合、その環境変数が格納されている環境（Environment）を有効化する必要があります。
手順としては、下の画像の✅マークの部分をクリックして、右上に、有効化したい環境名が表示されればOKです。
※1枚目画像が環境未設定の状態、2枚目が環境が有効化されている状態です

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c3d713f6-b21e-9d84-43bc-7a2402b78743.png" width="800">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b1465ee2-1524-6bfa-ce22-7e8cdfe6310d.png" width="800">

### 認証処理Pre-request Script（Postman）

[SwitchBot API Authentication](https://github.com/OpenWonderLabs/SwitchBotAPI#authentication)にも記載があるように、Ver1.1のAPIでは認証するための情報（署名）をタイムスタンプ等を用いて動的に生成する必要があります。
これをPostmanで実現するためには、Pre-request Scriptという機能を使用します。
この機能を使用することで、先ほど作成した変数へのデータの取得・保存や署名生成処理を実装することができます。
本来は、`README.md`に記載されているサンプルコードなどをベースに自分で作成する必要がありますが、以下の記事が大変参考になりましたので、この記事をベースにPre-request Scriptを作成しています。

https://qiita.com/nakatahr/items/e72662da466c55481273

基本処理の部分は記事と同じですが、変数の保存場所などが記事と異なるため、少し修正しています。コードは以下のようになります。

```JavaScript:Pre-request Script
let token = pm.environment.get("token");
let secret = pm.environment.get("secret");

const t = Date.now(); // UnixTime
const uuid = require('uuid');
const nonce = uuid.v4();
const message = token + t + nonce;
const sign =
    CryptoJS.enc.Base64.stringify(
        CryptoJS.HmacSHA256(message, secret)
    );

pm.collectionVariables.set("sign", sign);
pm.collectionVariables.set("nonce", nonce);
pm.collectionVariables.set("t", t);
```

### APIリクエストの追加＆テスト（Postman）

いよいよAPIリクエストの追加をしていきますが、Collection内にFolderを作成して視認性を良くします。
Folderは、Collection名の3点メニューをクリックして、`Add folder`をクリックすれば作成できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0610eb06-ed6e-efae-2734-4fabfbf71b2e.png" width="400">

作成したFolder毎に追加したAPIリクエストについて解説していきます。

#### デバイス情報取得

`1. デバイス情報取得`というFolder内にAPIリクエストを追加していきます。
APIリクエストは、追加したいFolderの3点メニューをクリックして、`Add request`をクリックすることで追加できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6aa85e86-7751-57ca-4623-3b0810ca67a9.png" width="600">

APIリクエストの書き方について、デバイスリストを取得する例を挙げて説明していきます。

##### ＜登録デバイス一覧＞

まず、HTTPメソッドとエンドポイント（URL）を指定します。この部分は、SwitchBot APIの`README.md`を参考に入力しますが、今回はメソッドは`GET`でエンドポイントは`https://api.switch-bot.com/v1.1/devices`となります。
ただし、ベースのURLはコレクション変数で設定しているため、その変数を活用します。
変数を呼び出す際は、`{{}}`で囲むことで変数を使用できます。
そのため、今回はエンドポイントの部分は`{{baseUrl}}/v1.1/devices`となります。

次に、`Headers`の項目に、各種パラメータを設定します。こちらも公式の`README.md`を参照すると、[Request Header](https://github.com/OpenWonderLabs/SwitchBotAPI#request-header)の部分で4つのパラメータを含める必要があるとの支持がありますので、それに従ってパラメータを追加します。
なお、いずれのパラメータも変数化しているため、その変数を呼び出す形になります。詳細は以下の画像を参考にしてください。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8705e5a3-03f4-4a5e-3dac-8beed0d59f2c.png" width="800">

これで、APIリクエストの設定ができましたので、テストしてみます。
右側にある`Send`ボタンをクリックすると、画面下の`Response`欄に結果が表示されます。
結果はJSON形式で表示され、`statusCode`が100なら成功です。その下にデバイスリストが表示されます。
自分の環境でやった例の画像を下に貼っておきます。
ここに表示されている`deviceId`をこの後の個別デバイスの設定用に使用します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2a56e806-70f8-4c4b-02ca-17a2afd90447.png" width="400">

##### ＜個別デバイス情報＞

次に、各デバイスの情報を取得するAPIリクエストを追加します。
こちらは、前述したようにデバイスの指定として、エンドポイント内にデバイスIDを指定する必要があります。

そこで、今回は以下の画像のようにエンドポイントを`{{baseUrl}}/v1.1/devices/:deviceid/status`という形で指定しています。この中の、`:deviceid`と記載することで、その部分をパラメータ化することができます。
下の`Path Variables`という項目に`deviceid`というKeyが自動で追加されるため、Valueの欄に情報を取得したいデバイスIDを記載することができます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e302620d-454b-4067-1f53-8d009049aeaa.png" width="800">

なお、今回は更に`{{deviceId}}`という形でコレクション変数を追加しています。これは、ある特定のデバイスについて、複数のAPIリクエストを連続で実行したいときに、APIリクエスト毎にデバイスIDを書き換えるのが手間になると考えたためです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9ce5f7b1-0744-d9c0-6675-8b575694f3e5.png" width="800">

:::note 補足
デバイスIDを変数化するなら、エンドポイントを`{{baseUrl}}/v1.1/devices/{{deviceId}}/status`としても良いのではと思う人もいるかと思います。
実際、その記載方法でも問題なく動きます。ただし、今回はベースは変数化した共通の`{{deviceId}}`を使用するが、APIリクエストをデバイスIDを変えて何回も実行したい場合に、コレクション変数切り替え⏩APIリクエスト実行の手順を繰り返す場合、画面の切り替えが発生することから、`Path Variables`上でもパラメータを変更できる形としています。
:::

こちらもデバイスIDを切り替えて実行すると、それぞれのデバイスの情報が取得できます。
温湿度計は温度や湿度、プラグミニは電圧や電流値などデバイスによって取得できる値は異なるので、詳細は[公式ドキュメント](https://github.com/OpenWonderLabs/SwitchBotAPI#get-device-status)を確認ください。
下の画像は、防水温湿度計に対して実行したレスポンス例です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4a6bcf83-acb8-8b1b-b8e6-753bd2cade31.png" width="400">

#### デバイス制御

次にデバイス制御のAPIリクエストを作成していきます。
今回使用したデバイスの中で制御リクエストに対応しているのは、以下2つのデバイスになります。

- SwitchBot プラグミニ（JP）
- SwitchBot ボット

##### ＜プラグミニ＞

基本的に先ほどと変わりませんが、2箇所ほど変更しています。
1つは、HTTPメソッドが制御リクエストの場合、POSTになります。
もう1つは、Bodyにパラメータを指定しています。こちらもパラメータ内容は[README.md](https://github.com/OpenWonderLabs/SwitchBotAPI#plug-mini-jp-2)の内容を参考に設定していきますが、以下の画像のように、`Body`欄を選択して、[raw]-[JSON]を選択した状態でパラメータを記載します。
以下の画像は、給電ON時の画像になります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1e8a80f6-d657-0d09-54a4-c498bd208e40.png" width="800">

給電ON、給電OFF、トグルそれぞれのjsonの記載は以下のようになります。

```json:給電ON
{
    "command": "turnOn",
    "parameter": "default",
    "commandType": "command"
}
```

```json:給電OFF
{
    "command": "turnOff",
    "parameter": "default",
    "commandType": "command"
}
```

```json:トグル
{
    "command": "toggle",
    "parameter": "default",
    "commandType": "command"
}
```

テストする際は、プラグミニのデバイスIDを変数に設定した状態でそれぞれのAPIリクエストを実行し、プラグミニの給電状態が切り替わることが確認できればOKです。

##### ＜ボット＞

ボットの場合もプラグミニとエンドポイントURLや`Body`に設定するパラメータもほぼ同じです。
一点、注意点として、ボットには「Switchモード」と「Pressモード」の2種類があり、このモード変更はスマホアプリ上から行う必要があります。
「Switchモード」の場合は、ON/OFFの概念がありますが、「Pressモード」の場合は実行する度に元の位置に戻るため、APIリクエストは1種類になります。そのため、今回は「Switchモード」と「Pressモード」でFolderを分けてAPIリクエストを記載しています。パラメータ内容は[README.md](https://github.com/OpenWonderLabs/SwitchBotAPI#bot-2)の内容を参考に設定していきます。

ON、OFF、プレスそれぞれのjsonの記載は以下のようになります。

```json:ON
{
    "command": "turnOn",
    "parameter": "default",
    "commandType": "command"
}
```

```json:OFF
{
    "command": "turnOff",
    "parameter": "default",
    "commandType": "command"
}
```

```json:プレス
{
    "command": "press",
    "parameter": "default",
    "commandType": "command"
}
```

テストする際は、「Switchモード」の状態でONとOFFのAPIリクエストを確認し、「Pressモード」の状態でプレスのAPIリクエストを確認できればOKです。

:::note 補足
確認した限りでは、「Switchモード」の状態でプレスのAPIリクエストを実行するとエラーが返ってきました。
一方、「Pressモード」時にONとOFFのAPIリクエストを実行した場合は、いずれもプレス時と同様の動作をしているようでしたが、今回は判別しやすいように、Folder毎に分けています。
:::

#### シーン制御

次にシーン制御のAPIリクエストを作成していきます。
シーンとは複数のSwitchBotデバイスの制御設定を予めひとまとめにして名前を付けておき、そのシーン制御をONにすると、一括でデバイス制御を行う機能となります。

以下に、参考になった記事を紹介します。似たような機能でオートメーションという機能もありますが、そちらについても以下の記事で紹介されています。

https://dekirucha.com/indoor/smarthome/23097/

シーン制御を使うメリットとして、APIでは対応できない複雑な処理（e.g. ハブ2の赤外線機能で家電をONするなど）といった処理を予めシーンに登録しておくことで、APIリクエスト経由でも制御が可能になります。

公開されているAPIについては、登録シーンの一覧を取得するAPIと個別シーンの実行を行うAPIの2つになります。
それぞれ、公式の[README.md](https://github.com/OpenWonderLabs/SwitchBotAPI#scenes)を参考にしながら実装します。

##### ＜登録シーン一覧＞

基本的には、前述の＜登録デバイス一覧＞のAPIリクエストと作り方は一緒で、エンドポイントURLのみ変更しています。
自分の環境でやった例の画像を下に貼っておきます。
ここに表示されている`sceneId`をこの後の＜個別シーン実行＞用に使用します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2ae6796a-79e8-ddb7-e020-50dc3fbfa113.png" width="700">

##### ＜個別シーン実行＞

個別シーンの実行も、＜個別デバイス情報＞のAPIリクエストと作り方はにてます。エンドポイントURL内に`sceneId`を指定する必要があるため、前回同様パラメータを変数化しています。
なお、今回は情報取得ではなく、シーン実行という形になるため、HTTPメソッドはGETではなくPOSTになります。ただし、`Body`欄には特にパラメータは指定しなくてOKです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/98d745d3-a604-c88c-f38b-ffb5897502dd.png" width="800">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/db3e387f-8d23-a7dc-6f08-f295910c832b.png" width="800">

テストする際は、APIリクエストを実行し、予め設定していたシーン通りにデバイスが制御されていればOKとなります。

#### Webhook

公式[README.md](https://github.com/OpenWonderLabs/SwitchBotAPI#webhook)に記載されている最後のAPIとして、Webhookがあります。この機能は、スマホアプリでは設定できないため、現状API独自の機能となっています。

Webhook自体の説明はここでは省略しますが、いくつか参考となるサイトを挙げておきます。

https://kintone-blog.cybozu.co.jp/developer/000283.html

https://circleci.com/ja/blog/webhook-explained/

Webhookを使用する場合、Webhook先のサービスが発行したURLをPostman側で指定する必要があるため、コレクション変数に`{{webhookUrl}}`という変数を追加しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/13130f00-9ff1-5851-639d-ad6aa2775651.png" width="800">

なお、Webhook関係のAPIは全てPOSTメソッドを使います。

##### ＜Webhook登録＞

新たにWebhookを登録するためのAPIリクエストになります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7a5679f9-ccae-4527-606d-2fcb07cf9f2c.png" width="800">

パラメータはURL以外は基本固定値になります。`deviceList`については、現状`ALL`のみサポートしているようです。

エンドポイントURL：`{{baseUrl}}/v1.1/webhook/setupWebhook`
HTTPメソッド：POST

```json:Webhook登録
{
    "action":"setupWebhook",
    "url":"{{webhookUrl}}",
    "deviceList":"ALL"
}
```

登録に成功すればメッセージに`success`と表示されます。既に登録済みの場合は、`webhook is exist`というメッセージが返ってきます。

##### ＜Webhook URL取得、Webhook構成取得＞

登録済みのWebhookのURLや構成情報を取得するAPIリクエストです。

エンドポイントURL：`{{baseUrl}}/v1.1/webhook/queryWebhook`
HTTPメソッド：POST

```json:Webhook URL取得
{
    "action": "queryUrl"
}
```

```json:Webhook構成取得
{
    "action": "queryDetails",
    "urls":["{{webhookUrl}}"]
}
```

##### ＜Webhook有効化、Webhook無効化＞

登録したWebhookを無効化したり、再度有効化するためのAPIリクエストです。
`enable`パラメータの`true/false`を返ることで、有効/無効を切り替えられます。

なお、ここで指定している`url`のパラメータに新たなWebhook URLを入力することで、URLを書き換えることも可能です。

エンドポイントURL：`{{baseUrl}}/v1.1/webhook/updateWebhook`
HTTPメソッド：POST

```json:Webhook有効化
{
    "action": "updateWebhook",
    "config":{
        "url":"{{webhookUrl}}",
        "enable":true
    }
}
```

```json:Webhook無効化
{
    "action": "updateWebhook",
    "config":{
        "url":"{{webhookUrl}}",
        "enable":false
    }
}
```

##### ＜Webhook削除＞

登録したWebhookを削除するAPIリクエストです。先ほどの無効化と異なり、登録情報自体を削除します。

エンドポイントURL：`{{baseUrl}}/v1.1/webhook/deleteWebhook`
HTTPメソッド：POST

```json:Webhook削除
{
    "action":"deleteWebhook",
    "url":"{{webhookUrl}}"
}
```

ここまででWebhookのAPIリクエストを一通り作成できたため、実際にWebhookが正しく動作するかテストしてみました。

しかし、**TeamsやSlackでIncoming Webhookを試したところ上手く動作しませんでした**（いつまで経っても通知がこない）。
なお、Postmanから直接Webhook先URLをPOSTで叩いた場合はWebhookが動作しており、原因は現状不明です。

そこで、一旦内容を解析するために、Azure Logic Appsを使用してTeamsにメッセージを送るフローを試したところ問題なくWebhook経由でTeamsにメッセージが届きました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5f3e9755-f19e-0e06-3d5b-822a20816cd7.png" width="800">

Azure Logic Appsのフローの詳細は省略しますが、内容を解析し、デバイスタイプが開閉センサまたはボットの場合のみ、Teamsのチャネルにメッセージを送るといった内容にしています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e16f13ac-ab16-3747-2c83-7f8e99659b75.png" width="800">

こちらの記事でもWebhookはちゃんと動作しているようなので、Webhookが上手く動作するパターンもあるというのが、現状分かってる内容です。（とりあえず試してみるしかない・・・？）

https://zenn.dev/7oh/scraps/c540b175727f28

ここまでで、README.mdに掲載されているAPIリクエストの登録は完了になります。
なお、本来は各APIの概要やパラメータなどをドキュメントに記載する必要があるのですが、今回は既にGitHubに公式のドキュメントがあることから、記載は最小限にして、後はGitHubへのリンクを張る形にしています。

## ◇おわりに

前述しましたが、今回のワークスペースはPostman上に公開しています。簡単に試してみたい場合は以下URLからforkして実行してみてください。

https://www.postman.com/lunar-resonance-715260/workspace/switchbot-api-ws

また、CollectionとEnvironmentをエクスポートしたjsonファイルをGitHub上でも公開していますので、そちらも参考にしてもらえればと思います。

https://github.com/yankee-08/Postman-API

記事は以上です。[Postman Advent Calendar 2023](https://qiita.com/advent-calendar/2023/postman)の次の記事につなげたいと思います。

# 🔚END
