---
title: Power AutomateでRSS情報をTeamsにアダプティブカード形式で通知する
tags:
  - RSS
  - Teams
  - iPaaS
  - AdaptiveCard
  - PowerAutomate
private: false
updated_at: '2023-03-28T20:29:48+09:00'
id: b9dfba1b5e461a389b19
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに
本記事では、Microsoft社Power Platformの中のPower AutomateのRSSコネクタを使い、指定したサイトのRSSの更新情報をTeamsにアダプティブ形式カードで投稿する手順をまとめた記事になります。

## ◇開発環境等
<dl>
  <dt>OS：</dt>
  <dd>Windows 10 Home（Ver:22H2）</dd>
  <dt>契約ライセンス：</dt>
  <dd>Microsoft 365 Business Basic</dd>
  <dt>使用ツール：</dt>
  <dd>Power Automate(Microsoft365ライセンス内)</dd>
  <dd>Microsoft Teams</dd>
  <dd>Designer | Adaptive Cards</dd>
</dl>

## ◇実装
### フロー部分
ブラウザからPower Automateのページに入り、フローの作成を行っていきます。
今回はテンプレートをベースに作成を行いました。
左側のメニューからテンプレートを選択し、検索ボックスに「RSS フィード」などと入力し、「条件に基づいてニュースフィードをメールに送信する」テンプレートを選択します。
※「RSSフィードが公開されたときにTeamsにメッセージを投稿する」テンプレートもありますが、条件分岐の処理入れたかったので、今回は選びませんでした

![001.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4401fb24-61b2-1f7b-1e35-201be84df090.png)

![002.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f47332e5-3828-35c1-ee22-92c230f59879.png)

`RSSフィードのURL`には、確認したいサイトのURLをそのまま入力し、次の項目のプロパティは`PublishDate`を選択します。
条件分岐のコネクタでは、`フィードタイトル`、`フィールドの概要`などを選択し、抽出したいキーワードを入力することで、そのキーワードを含むRSSフィードのみ、Teamsに通知できます。

:::note info
補足
著作権情報でも切り分けられないかと試してみましたが、RSSにAuthor情報が入っていても、Power AutomateのRSSコネクタの著作権情報アイテムに入らないようです（海外コミュニティサイトでも似たようなコメントあり）。
:::



![003.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e00593ae-dd68-1815-226d-1cf7019a8e29.png)

次に、`はい`の場合のフローを作成します。
今回は使わないため、予めメールのアクションは削除しておきます。
「アクションの追加」ボタンを押して、使用するアクションを選択します。
Teamsの中から、赤枠のアクションを選択してください。
※なぜかこのアクションだけ英語表記なので注意

![004.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6871efab-bf70-1ca2-fe31-a948eb48cdb6.png)

アダプティブカードをチャネルに投稿アクションで、投稿方法、投稿先を順に選んでいきます。
なお、チャネルに投稿する場合は事前にTeamsでチャネルを作成しておく必要があります。
`Adaptive Card`の項目にアダプティブカードの内容を記載していくのですが、この部分は別サイトで作成したものを貼り付ける形になるため、次の項で説明します。

![005.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1cad9446-8e5f-e037-ff5a-487cfe254e63.png)

#### アダプティブカードの作成
次に、Teamsに通知するためのアダプティブカードの設計・作成をしていきます。
作成には「Designer | Adaptive Cards」のサイトを使用しました。

Designer | Adaptive Cards

https://www.adaptivecards.io/designer/

アダプティブカードの作成にあたっては、以下の記事を参考にさせていただきました。

https://speakerdeck.com/miyakemito/power-automatedefalseadaptive-cards-ji-ben-bian

https://qiita.com/Satoshi_Yoshino/items/f4feaabd0c9258df566a


#### 初期設定
まず、上部にある、
- `Select host app`を`Microsoft Teams`
- `Target version`を1.4
に変更します。

次に、左上の`New card`を押し、`Blank Card`を選択します。

![006.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/92c42113-96c1-31ca-f51b-5004f480de41.png)

#### コンテンツ設定
最終的なカードのデザインは以下のようになります。
全部で4行構成になっているので、1行ずつ説明していきます。

![020.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/777e0824-d1f1-25f2-c862-6bbed8bd1c6a.png)


:one: 1行目
2段組にして、左側にRSSを取得したサイト名、右側にサイトのTOPページへのリンクを張っています。
段組にする場合、最初に左のCARD ELEMENTSの中から`Containers`-`ColumnSet`を選択し、カード上に配置します。
`ColumnSet`を選択した状態で、右下に表示されるアイコンを押す度に`Column`が追加されます（今回は2段組のため、2回押す）。

![007.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/a586c489-7e60-e24c-55dd-38a5483b4d79.png)

左側にはテキストを入力したいので、`Elements`-`TextBlock`を選択し、左側の`Column`の中に配置します。
`TEXT BLOCK`を選択して、`ELEMENT PROPERTIES`の各項目を設定していきます。
今回は、Text、Separator、Sizeあたりをいじってます。

![008.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d3123c08-2e04-dc5c-79bb-19a567edd492.png)

次に、右側の`Column`には、`Elements`-`ActionSet`を配置し、`Action.OpenUrl`を選択します。

![009.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c4f99047-3308-9321-9dad-6f518e5a2c58.png)

`ELEMENT PROPERTIES`では、表示名（Title）と開きたいURLを入力します。
今回は、RSSを配信しているサイトのTOPページのリンクを書く想定です。

![019.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/adf1fd7f-48cb-bd61-dafe-96ff59ef0dde.png)


最後に、右側の`Column`を選択した状態で、`Layout`-`Width`を`Automatic`に変更しています。
これによって、右側のリンクボタンの幅が自動調整されます。

![011.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/35c99843-b245-6f24-5636-3ba144482a3f.png)

:two: 2行目
2行目はRSSのタイトルを表示したいので、そのまま`TextBlock`を配置し、先ほど同様SizeやTitleを記入します。
ここで、`Text`部分はPower Automate上で値を置き換える予定ですので、わかりやすいように★タイトル★としていますが、分かれば何書いてもOKです。

![012.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6fc896c8-6e37-4690-51b3-bc4b2999592c.png)

:three: 3行目
3行目は左側に画像、右側にRSSの概要を載せていきます。
先ほど同様、`ColumnSet`を配置し、2個`Column`を追加します。
画像の`ELEMENT PROPERTIES`には、参照するURL、画像サイズを指定します。
なお、画像を直接添付する場合は少し手間が必要なようです。
（以下のサイトが参考になるかと思います）

https://mofumofupower.hatenablog.com/entry/AC_Image_2023


![013.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/78910426-8e7a-a192-f97b-8a88a734397f.png)


:::note warn

一般的なイラストサイトの画像URLを指定すると、いわゆる直接リンク（直リン）状態になってしまうので、そのサイト自体が明確に許可している場合（見たことはない）以外はやらない方が無難かと思います。
:::

右側の`Column`には2行目と同様に、`TextBlock`を配置し、TextやSizeを設定します（詳細は省略）。


:four: 4行目
4行目はRSSの記事のサイトへのリンクボタンを載せていきます。
リンクボタンを右に寄せる方法が分からなかったため、他と同じように、`ColumnSet`を配置し、2個`Column`を配置しています（左側には何のブロックも入れない）。
右側の`Column`には、1行目と同じように、`ActionSet`を配置し、`Action.OpenUrl`を指定します。
`Url`の項目はPower Automateで書き換えるため、適当なURLを入力しておきます。

![014.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9b943a25-7319-e97f-c955-18d678dea65f.png)

ここまでで、アダプティブカードの作成ができました。データは上部の`Copy card payload`を押すとコピーできます。

![018.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/201fa2b7-aa96-7fba-3ea7-aacb8d27c8e2.png)

### フローへのアダプティブカードの入力
コピーしたPayloadをPower Automateフローの`Adaptive Card`の項目にペーストします。
その後、
- 「★タイトル★」:fast_forward:「フィードタイトル」
- 「★概要★」:fast_forward:「フィードの概要」
- 「https://★★/」:fast_forward:「プライマリフィードリンク」
にそれぞれ置き換えます。

![015.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/17194ee9-d450-fdd9-fca5-7b4c9f39d41b.png)

![016.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d58452e9-5d2c-5e5d-cd80-fe3077ca6599.png)

![017.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ed464794-659c-61be-fe8e-27364fb89fb1.png)

ここまでできたら、フローを保存して、テスト実行します。
※RSSの頻度によってはテストがエラーになるかもしれません

## ◇実際の配信画面
Teamsのチャネルに実際に配信されるとこんな感じになります。
※IPAのRSSでは、タイトルしか含まれていなかったため、記事の概要が空っぽになっています（事前確認不足）
タイトル部分と概要部分の記述量はサイトによって異なるので、フォントサイズはサイトによって変えてもいいかもしれません。
また、同様のPower Automateフローを各RSS毎に作成し、RSSフィード毎にフォントカラーや画像アイコンを変えて見分けをつきやすくしてもいいと思います。

![021.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/892fb558-6fc0-a2c6-6ca1-cb944548a75a.png)


## ◇おわりに
今回は、アダプティブカードを使って、RSSフィードのTeams配信を行いました。
今回は行っていませんが、RSSフィードの中身を解析して、話題によって投稿するチャネルを変えるとかの対応を入れてみても面白いかもしれません。
最後にこの記事を作成している中で気になった点を記載しておきます。

:::note warn
注意点
- RSSフィードについては、サイトによっては個人利用に限定している場合もあり、チーム内への再配信が禁止の場合もあるため、各サイトのルールに従う必要あり
- アダプティブカードに画像を載せる際、イラストサイト等の画像URLを直接指定すると、直接リンク（直リン）状態になるため、注意が必要
:::


以上
