---
title: Power AppsのVisibleプロパティを使って、入力内容に応じて入力項目の表示・非表示を切り替える
tags:
  - SharePoint
  - PowerApps
  - PowerPlatform
  - ローコード
private: false
updated_at: '2023-10-13T00:05:13+09:00'
id: a183e575c74ca4e832eb
organization_url_name: null
slide: false
ignorePublish: false
---
## ◇はじめに

Microsoft Power Appsを使ってアプリをつくる際に、入力フォーム画面の項目の表示を動的に変えたい場面があります。（例を挙げると、アンケートのフォームで「その他」を選択した場合のみ、テキスト入力画面を表示するなど）
そこで、今回はVisibleプロパティを使った入力項目の切り替えを主要なタイプについて備忘録としてまとめました。

## ◇開発環境等

- OS：
  - Windows 11 Home（Ver:22H2）
- 契約ライセンス：
  - Microsoft 365 Business Basic
- 使用ツール：
  - Power Apps
  - SharePointリスト

## ◇試したデータ型

今回は、以下のデータタイプについてまとめました。
SharePointリストで列を追加するときのタイプ名で記載していますが、括弧内はPower Appsでのデータ型になります。

- テキスト（テキスト）
- 選択肢（複合）
- 日付と時刻（日時）
- ユーザー（複合）
- 数値（数値）
- はい/いいえ（ブール値）

## ◇事前準備

### SharePointリストの作成

まず、それぞれのデータタイプを列に追加したSharePointリストを作成し、そのリストをベースにテスト用のPower Appsのアプリをつくっていきます。

- テーブル名
  - SampleList
- 列名
  - タイトル
  - Text：テキスト
  - Choice：選択肢
  - Datetime：日付と時刻
  - User：ユーザー
  - NumericalValue：数値
  - OnOff：はい/いいえ
  - PopupWindow：複数行テキスト

最終列のPopupWindowが表示・非表示を切り替えるための項目となります。
サンプルデータを2行分入力していますが、実際にはこのデータは使いませんでした。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d58058b7-b6bd-c842-70f2-3051edb2902f.png" width="800">

### キャンバスアプリの作成＆SharePointリストとのデータソース接続

以下の手順で、テスト用のPower Apps画面をつくっていきます。

- Power Appsのメニュー上から、「作成」－「空のアプリ」をクリック
- 「空のキャンバスアプリ」の作成をクリック

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b9b23f3c-80fe-f605-c822-3e7a17ecd3cd.png" width="400">

- アプリ名と形式を選んで作成ボタンをクリック（今回は電話形式にしています）
- スクリーン上に「挿入」－「編集フォーム」でフォームを追加

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4b9b2c81-75df-1bfb-6480-a9f2397f35e4.png" width="500">

- 挿入したフォームを選択し、「プロパティ」－「データソース」から、先ほど作成したSharePointリストを接続

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4bf52961-06bb-7f92-58d4-98233dca9eb5.png" width="400">

- 「フィールドの編集」を選択し、タイトルフィールドを削除（今回は特に使用しないため）

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f0390321-2594-3d9c-f726-a50ccb0c7600.png" width="600">

ここまでの作業で、各入力項目が表示された画面が作成できます。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ceb6d5bf-09d3-6e03-6a37-d922142b5a44.png" width="300">

## ◇各データ型の場合のPower Fxの実装

### データ型：テキスト

#### Visible条件

Textの項目（今回は`DataCardValue9`）の内容が`"detail"`の場合、PopupWindowの項目を表示

#### Power Fx

```Power Fx
Visible =
If(DataCardValue9.Text="detail",true,false)
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/25ab4e44-8841-2d24-f947-2d037e768a5f.png" width="800">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d1e6eb17-fc61-a149-2465-e28b3bd8f7ec.png" width="300">   
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ccd5b52e-bd16-4c12-5d3c-52d5e70113e1.png" width="300">

テキスト型の場合はそのまま`***.Text`を指定すれば値の取得ができます。

### データ型：選択肢

#### Visible条件

選択肢（今回は`DataCardValue10`）の選択が`"fuga"`の場合、PopupWindowの項目を表示

#### Power Fx

```Power Fx
Visible =
If(DataCardValue10.Selected.Value="fuga",true,false)
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/70b47dfc-9d2d-5acd-e2ed-f5f00f5d175f.png" width="800">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/08c1a091-471b-b500-c1fc-6c63f61b3aba.png" width="300">   
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/de969a21-23ec-f45a-8680-d5f2c478e240.png" width="300">

選択肢型の場合、現在選択されている選択肢を取得する場合、`DataCardValue**`の後ろに`Selected`を付ける必要があります。
なお、今回は選択肢の複数選択を不可にしているためこの方法で上手くいきましたが、複数選択可にした場合は未検証です（たぶん`SelectedItems`プロパティあたりを使う必要がある？）

### データ型：日付と時刻

#### Visible条件

入力した日付（今回は`DateValue2`）が今日の日付より前の日付の場合、PopupWindowの項目を表示

#### Power Fx

```Power Fx
Visible =
If(DateValue2.SelectedDate<Today(),true,false)
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5438d57b-b18c-53af-6bbc-f8e17e32325f.png" width="800">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4dc7cd7a-850d-739a-3cfc-1346d705771e.png" width="300">   
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f78ec08b-62cc-2566-3986-d743e225f785.png" width="300">

:::note info
補足
今回の実験は2023/9/20に実施してます
:::

日付と時刻型の場合、`SelectedDate`というプロパティで日付の情報を取得することが可能です。

### データ型：ユーザー

#### Visible条件

選択されたユーザー（今回は`DataCardValue11`）の名称が特定の名前の場合、PopupWindowの項目を表示
※本名のアカウント名のため、名前の部分は伏せてます

#### Power Fx

```Power Fx
Visible =
If(DataCardValue11.Selected.DisplayName="*******",true,false)
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/ce0ac76d-32ee-5d0f-839e-402f2c78820a.png" width="800">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/516a833a-31e3-9ab6-1bf8-76cd43f57303.png" width="300">   
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/458bc27b-de98-d513-8950-afc1a2f490a2.png" width="300">

ユーザー型の場合も値の取り方は選択肢型と同様、`Selected`プロパティが必要になります。
ただし、ユーザー名で取得する場合はその後に`DisplayName`プロパティを使用します。

### データ型：数値

#### Visible条件

入力された値（今回は`DataCardValue12`）の値が100を超える場合、PopupWindowの項目を表示

#### Power Fx

```Power Fx
Visible =
If(Value(DataCardValue12.Text)>100,true,false)
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/dba05d4d-b1a7-d157-7391-2cb4b44127c8.png" width="800">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/98b4f5c7-a1d6-0b76-b0eb-aed4eb413361.png" width="300">   
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/66eca405-1ff7-fa7a-d276-0fe049c5b55c.png" width="300">

数値型の場合、テキスト型と同様に`***.Text`を指定すれば値の取得ができますが、このままだと数値データとして扱えないため、取得したテキストデータに対して`Value()`関数で括っています。

### データ型：はい/いいえ

#### Visible条件

ON（今回は`DataCardValue13`）の場合、PopupWindowの項目を表示

#### Power Fx

```Power Fx
Visible =
If(DataCardValue13.Value,true,false)
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6cfd5a61-19d2-4fc1-b2a0-6e727c4cafd8.png" width="800">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2df134f3-c2d4-8ce6-8afe-40515dd4bf25.png" width="300">   
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8d51804c-ec87-04c9-6d2a-a77545906b43.png" width="300">

はい/いいえ型の場合、`***.Value`プロパティで値の取得ができます。
取得結果がそのままtrue or falseとなるため、そのまま`If()`関数で判定できます。

## ◇おわりに

使い慣れている人からすれば調べるまでもない内容かと思いますが、自分の備忘録も兼ねて記事にしてみました。
どなたかの参考になれば幸いです。

# 🔚
