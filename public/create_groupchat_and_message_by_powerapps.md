---
title: Power AppsからTeamsのグループチャットに直接メッセージを送る方法
tags:
  - PowerApps
  - Teams
  - PowerPlatform
  - ローコード
  - AdventCalendar2023
private: true
updated_at: '2023-11-28T20:50:01+09:00'
id: fc127636876755b24935
organization_url_name: null
slide: false
ignorePublish: false
---

## ◇はじめに

本記事は[Microsoft Power Apps Advent Calendar 2023](https://qiita.com/advent-calendar/2023/powerapps)の4日目の記事です。

https://qiita.com/advent-calendar/2023/powerapps

今回は、Power Apps上から、Teamsにグループチャットを送る方法について記事にしてみました。
Power AppsからTeamsのチャネル上にメッセージを送る例はいくつか事例を載せているサイトがありましたが、
グループチャットの例はなさそうだったので、記事にしてみました。

## ◇開発環境等

- OS：
  - Windows 11 Home（Ver:22H2）
- 契約ライセンス：
  - Microsoft 365 Business Basic
- 使用ツール：
  - Power Apps
  - Teams
  - SharePointリスト

## ◇作成した背景

Power Apps上からTeamsにメッセージを送る仕組みを作りたいと考え、調べたところ、よくあるパターンとして、Power Automate経由でメッセージを通知する事例が紹介されています。
しかし、この場合、Power Automateのフロー作成者またはユーザーからメッセージを通知する仕様となってしまいます。
送信先がチャネルの場合は送信元にフローボットを指定すればそこまで問題にはなりませんが、今回はグループチャットにメッセージを投げる仕様としたかったため、フロー作成者が入っていないグループチャットにもメッセージを送る仕様としたかったというのが背景です。

:::note
アプリ作成者A、アプリ実行者B、通知先Cの場合、Bがアプリを実行した場合にBからCにTeamsのチャットメッセージが送られるようにし、この際、アプリ作成者Aには特に通知されない仕様とする
:::

## ◇最終的にできたアプリ

以下のようなアプリケーションになります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/de2d2e48-75f9-eea9-7add-e67a9518ee42.png" width="800">

- タイトル、送信先、メッセージを入力
- 送信先は複数ユーザーを選択可能
- 実行ボタンでチャットメッセージ送信
- SharePointリスト上にもデータを保存

といった最低限の機能になっています。

## ◇実装

### データ保存部（SharePoint）

Teams通知には直接関係ありませんが、今回はSharePointリストを使用してTeamsで送る送信先ユーザーやメッセージを保存する仕組みとしています。

まず、SharePointリストの作成を行います。
テーブル名、列名は以下のような構成としています。

- テーブル名
  - GroupChatbyPowerApps
- 列名
  - タイトル
    - 用途：グループチャット名（1対1チャットの場合無効）
  - SendUser
    - データ種類：「ユーザーまたはグループ」（複数選択を許可）
    - 用途：チャット送信ユーザー
  - Message：
    - データ種類：「複数行テキスト」
    - 用途：チャット送信メッセージ

### データ登録UI部（Power Apps）

#### SharePointリストからPower Appsキャンバスアプリの作成

次に、SharePointリスト上からアプリを作成する方法でアプリの作成を行っています。  
まず、Power Appsのトップページから、「データで開始する」を選択後に「外部データを選択」を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/73346cc3-9547-c14b-3e18-aa0e3e36294d.png" width="800">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1e58e64c-308e-0917-91d2-ef831f12377b.png" width="800">

次に、データセットの選択肢では「SharePointから」を選択し、先ほど作成したSharePointリストを選びます。
基本的に、プルダウンから選べるはずです。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/3438f18a-c129-3356-9a0c-9c9a9be689cd.png" width="800">

作成ボタンを押して、しばらくすると、アプリのひな形が作成されます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/45d60bc6-face-b330-bdb5-c8a057797a4a.png" width="800">

#### レイアウトの微調整

今回、項目が少ないため、文字サイズを大きくしたり、必要の無い添付ファイルのフィールド表示を削除しています。
詳細は省略しますが、表示フィールドの設定については、`Form`のプロパティ内の「フィールドの編集」からフィールドの削除が可能です。
その他、各項目（DataCardKey,DataCardValue）のフォントサイズを自分が見やすいサイズに変更します。
加えて、アプリ名や各項目の名称を日本語化しています。

最終的に、以下のようなレイアウトに変更しています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/28a7ebee-ddd1-6166-25a9-2aef6957b83a.png" width="800">

#### データソースTeamsの追加

Power AppsからTeamsに通知を送るには、データソースとして、Teamsを追加する必要があります。
手順としては、

1. 左側メニューから「データ」を選択
2. 「＋データの追加」を選択
3. データソースの選択メニューから「Microsoft Teams」を選択

すれば完了です。
問題なく追加されれば、データメニュー上にSharePointに並んで、MicrosoftTeamsが表示されます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4ef295d6-cff0-0c36-8f96-a51a0dd8457e.png" width="600">

#### グループチャットへのメッセージ送信処理の実装

今回の記事のメイン部分になります。
今回は、データの新規作成を登録する際にだけTeamsにメッセージを送信する処理としています。
そのため、修正した部分は以下の画像にある、「SubmitFormButton」クリック時の処理（OnSelect）に関数を追加していきます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/7e8e960a-eb67-6570-e595-62b9522be4f6.png" width="800">

元々のOnSelect時の処理は以下になります。

```java
SubmitForm(Form1)
```

ここにメッセージ送信処理を追加したのが以下のコードになります。

```java
ClearCollect(sendUserTbl,DataCardValue2.SelectedItems);
//選択したユーザー一覧から、Emailを取得して、カンマ区切りで並べる
UpdateContext({emaiList:Concat(sendUserTbl,Email,",")});
//新規でグループチャットを作成し、そのチャットIDを変数に保存（※1対1チャットの場合、タイトルは無視される）
UpdateContext({chatId:MicrosoftTeams.CreateChat(emaiList, {topic:DataCardValue1.Text}).id});
//Teamsにメッセーを遅れるように変換して、送信
UpdateContext({postMessage:{recipient:chatId, messageBody:DataCardValue3.Text}});
MicrosoftTeams.PostMessageToConversation("Flow bot", "Group chat", ParseJSON(JSON(postMessage)));
SubmitForm(Form1)
```

各行でやっていることを順に補足していきます。

---

```java
ClearCollect(sendUserTbl,DataCardValue2.SelectedItems);
```

`DataCardValue2`は「送信先」欄のデータになります。今回、**複数ユーザーを選択した場合はグループチャットで送る**形にしたかったため、データを`SelectedItems`の形で取得しています。
この場合、取得できるデータはテーブルになるため、`ClearCollect`関数で`sendUserTbl`に格納しています。
`Collect`関数の場合、データが削除されず保持されてしまうため、`ClearCollect`で毎回データをクリアするようにしています。

https://learn.microsoft.com/ja-jp/power-platform/power-fx/reference/function-clear-collect-clearcollect#clearcollect

---

```java
UpdateContext({emaiList:Concat(sendUserTbl,Email,",")});
```

次に、作成したテーブルから、Emailの列のデータを抽出し、カンマ区切りで連結した文字列を作成しています。
データの抽出・連結処理を`Concat`関数で行い、その結果を`emaiList`という変数に格納しています。
変数はこの画面上でしか使う予定がないため、`Set`関数ではなく、`UpdateContext`関数にしています。

余談ですが、この処理部分はなかなか思い通りにいきませんでしたが、「Copilot with Bing Chat」を使用したら一発で求めていた結果が得られました（むっちゃ有能✨）

https://learn.microsoft.com/ja-jp/power-platform/power-fx/reference/function-concatenate

https://learn.microsoft.com/ja-jp/power-platform/power-fx/reference/function-updatecontext

---

```java
UpdateContext({chatId:MicrosoftTeams.CreateChat(emaiList, {topic:DataCardValue1.Text}).id});
```

次に、新規チャットを作成します。`CreateChat`関数の引数は以下のサイトを参考に記載しています。
※よくみると、セミコロン`;`区切りでユーザーのアドレスを記載しろと書いてありました。上のコードではカンマ`,`区切りで記載してしまっていますが、これでも動きました（後でセミコロンに直して検証します）

https://learn.microsoft.com/en-us/connectors/teams/#create-a-chat

上記サイトにも記載されていますが、
> Enter title (displays in group chats, not 1:1 chats)

となっており、送信先を1人しか指定しなかった場合、1:1のチャットとなるため、タイトルの記載内容は無視される形になります。
チャットの作成に成功すると、作成されたチャットIDが戻り値になるため、その結果を`chatId`に格納しています。

---

```java
UpdateContext({postMessage:{recipient:chatId, messageBody:DataCardValue3.Text}});
MicrosoftTeams.PostMessageToConversation("Flow bot", "Group chat", ParseJSON(JSON(postMessage)));
```

最後に、チャット上にメッセージを送信する処理です。まず、送信するメッセージを`PostMessageToConversation`関数の引数に合うように変換する必要があるのですが、以下のサイトを見た限りでは、PostMessageの変数のタイプが`dynamic`となっており、具体的な指定方法が分かりませんでした。

https://learn.microsoft.com/en-us/connectors/teams/#post-message-in-a-chat-or-channel

そこで、以下の記事を参考にして引数の指定方法を調べました。

https://qiita.com/nayoshik/items/d40710c4be57440f9cb5

まず、1行目の`UpdateContext`関数でメッセージのBodyの中身を変数に格納し、2行目で先ほど作成したチャットID二体してメッセージを送信しています。

ここまでで、アプリの実装が一通り完了したので、アプリの保存と公開を実行します。

### 動作確認

#### 検証用ユーザの作成

一通り、実装が完了したため、アプリの動作確認を行います。
今回は、アプリ作成者の自分を介さずに、アプリの実行からチャットメッセージ送信ができているかを確認する必要があります。
そこで、検証用ユーザとして、4人分のユーザを定義しています。

1. Alice
2. Bob
3. Charlie
4. David

#### アプリとSharePointリストの共有

次に、追加したこの4つのアカウントに今回作成したPower AppsアプリとSharePointリストを共有します。
この対応を行わないと、アプリを正しく開けませんので、注意してください。

Power Apps側
なお、共有する際はユーザー権限レベルで問題ありません

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/127a7c8b-491d-7776-c9af-317f887aae37.png" width="600">

SharePointリスト側
※なぜか1度に4アカウント分追加できなかったため、Bobアカウントのみ後から追加しています

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/644f8d2a-7d97-1e08-099a-3f536ecd58bd.png" width="300">

#### 送信テスト：送信先が1名の場合

今回のアプリは複数名チャットに対応していますが、まず最初に1対1のチャットを試してみます。
送受信は、

**送信者**：Alice
**受信者**：Bob

という形にしています。

まず、Aliceアカウントからアプリを起動して、下の画像のようにタイトル、送信先、メッセージを入力して、右上の「✅」ボタンをクリックします。

＜Alice＞
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2bfb867a-8904-4f8a-edeb-e635989d1c47.png" width="800">

その後、少しすると実行が完了し、AliceとBobのチャットが開始され、入力したメッセージが表示されます。
なお、今回は送信先を`Flow bot`としているため、Workflow経由のAliceからメッセージが送信された形になります。

＜Alice＞
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4eaa77e8-5eaa-bf70-2658-43eccd3a1cff.png" width="800">

次に、Bob側のTeams画面キャプチャを見ると、こちらもAliceとの1対1チャットになっており、メッセージを正しく受け取れていることが確認できます。
前述したように、今回は1対1チャットのため、アプリ画面で入力したタイトル（AliceからBobへ）は無視されています。

＜Bob＞
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0d7a5afe-7825-2f9d-6c0e-07e540f98370.png" width="800">

このとき、アプリの作成者である自分のアカウント上のTeamsには何の通知も来ていませんので、当初の想定通りの動作をしていることが確認できました。

#### 送信テスト：送信先が複数名の場合

次に、複数ユーザーを指定して、アプリの実行を試してみます。
送受信は、

**送信者**：Alice
**受信者**：Charlie、David

という形にしています。

先ほどと同様、Aliceアカウントからアプリを起動して、下の画像のようにタイトル、送信先、メッセージを入力して、右上の「✅」ボタンをクリックします。
今回は、送信先に2ユーザー追加していることに注意してください。

＜Alice＞
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/de2d2e48-75f9-eea9-7add-e67a9518ee42.png" width="800">

今回もしばらくすると、実行が完了し、Alice、Charlie、David　3名のグループチャットが作成され、入力したメッセージが表示されることが確認できます。
今回は、先ほどと違い、アプリのタイトルで指定した名称がグループチャットの名前になっているのが分かります。

＜Alice＞
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/541c2277-3677-b73a-f247-2a688553f8c1.png" width="800">

Charlie、David双方のTeams画面も以下のようになっており、問題なくチャットの作成・メッセージの受信ができていることが確認できました。

＜Charlie＞
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2c9a539e-5da8-146a-77dd-5428dc70cbd0.png" width="800">
＜David＞
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/99d9b7d7-e725-20d8-8ce6-f863d80f7815.png" width="800">

### コードの整理

一通り動くことが確認できたので、最後にPowerFxの関数を少し手直しします。
具体的には、`If`関数、`IfError`関数、`Notify`関数を使い、どこかで関数の実行が失敗した場合は、処理を止めてバナーメッセージを表示するようにしています。

```java
If(IfError(
    ClearCollect(sendUserTbl,DataCardValue2.SelectedItems), Notify("ユーザーリストの取得に失敗しました", NotificationType.Error);  false,
    //選択したユーザー一覧から、Emailを取得して、カンマ区切りで並べる
    UpdateContext({emaiList:Concat(sendUserTbl,Email,",")}), Notify("Mailリストの連結に失敗しました", NotificationType.Error);  false,
    //新規でグループチャットを作成し、そのチャットIDを変数に保存（※1対1チャットの場合、タイトルは無視される）
    UpdateContext({chatId:MicrosoftTeams.CreateChat(emaiList, {topic:DataCardValue1.Text}).id}), Notify("チャットの作成に失敗しました", NotificationType.Error);  false,
    //Teamsにメッセーを遅れるように変換して、送信
    UpdateContext({postMessage:{recipient:chatId, messageBody:DataCardValue3.Text}}), Notify("送信メッセージの作成に失敗しました", NotificationType.Error);  false,
    MicrosoftTeams.PostMessageToConversation("Flow bot", "Group chat", ParseJSON(JSON(postMessage))), Notify("Teamsメッセージの送信に失敗しました", NotificationType.Error);  false,
    true
),
Notify("登録が完了しました。Teamsで通知されているか確認してください。", NotificationType.Success);
SubmitForm(Form1)
)
```

最後まで関数が実行された場合のバナー画面が左側画像、途中で失敗した場合のバナー画面が右側画像となります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ca5f3acc-9611-b9f9-7e23-3cb9747b942b.png" width="350">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ed9abb4c-6eb0-06ce-0ab0-4ca3d6878639.png" width="350">

一点アプリ使用上の注意点として、送信先ユーザーにアプリ実行者自身を含めるとエラーが発生します。
もし、このエラーを回避したい場合、自ユーザーが入っていた場合は結合時に外すと言った処理を実装する必要があります（今回は実装してません）。

## ◇おわりに

記事上ではサラッとアプリが作成できたように書いてますが、実際はデータがどんな風に抽出されているかなどで結構ハマりました。
下の画像キャプチャは途中のテーブルデータや変数を表やラベルに表示してデバッグしている状態の画像です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/25745a14-073f-f84a-8680-a63712a62534.png" width="800">

とりあえず悩んだときは1ステップずつデータや処理の中身を確認しつつ、進めていくことが重要かと思います。
今回作成したアプリは動作確認用の簡素な物ですが、もう少し整理すれば、例えば、伝言メモアプリのようなものをつくることもできますので、ぜひ参考にしてもらえればと思います。

# 🔚END
