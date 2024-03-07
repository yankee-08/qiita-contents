---
title: Power Appsでつくる服薬管理アプリ
tags:
  - SharePoint
  - PowerApps
  - PowerPlatform
  - ローコード
private: false
updated_at: '2023-08-17T23:44:30+09:00'
id: ac81baeb7d943ada4a2d
organization_url_name: null
slide: false
ignorePublish: false
---

## ◇はじめに

今回は、Microsoft社Power Platformの中のPower Appsを使い、服薬アプリをつくってみました。  
背景としては、毎日服薬する必要のある薬を飲むにあたり、よく飲んだかどうか記憶が曖昧になることがあったため、どうせならアプリを作って対策してみようと思ったためです。

## ◇開発環境等

- OS：
  - Windows 11 Home（Ver:22H2）
- 契約ライセンス：
  - Microsoft 365 Business Basic
- 使用ツール：
  - Power Apps
  - SharePointリスト

## ◇最終的にできたアプリ

以下のようなアプリケーションになります。

- 薬を飲んだときに登録
- 服薬した履歴表示
- 過去のデータの削除、修正

といった最低限の機能になっています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/921b7905-3fa6-5fd2-fa7e-50696bcbe7df.png" width="250"> 
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/40aa3d0b-c1d3-8b38-b5c4-ad6d3d7e4163.png" width="250">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b5602470-cb43-ddd5-8830-10f6e7415512.png" width="250">

## ◇やりたかったこと

アプリを作る上で必要な機能を以下にまとめました。

1. 薬を飲んだ日時、薬の種類をアプリから登録する
2. 日時は現在日時をデフォルト値にしておく
3. 薬の種類は選択肢から選択可能にする
4. 登録したデータの履歴を確認できる
5. 登録データの修正・削除が可能

なお、この内、3については選択肢は事前にデータベース側で準備する形にします。

また、今回は実装していませんが、いずれ以下のような機能も入れたいと考えています。

＜今回は見送った機能＞

- リマインダー機能  
設定した時刻に当日の服薬状況をチェックし、薬の飲み忘れがあった場合はスマホに通知する
- 服薬前後のタイマー設定機能  
薬によっては服薬前後に飲み食い禁止だったりするため、タイマー機能をつける

## ◇実装

### データ保存部（SharePoint）

データの保存先は今回、SharePointリストを使っています。
先にリストを作成し、そのリストをベースにアプリの作成を行っていきます。

テーブル名、列名は以下のような最低限の構成になっています。
なお、「タイトル」列については、現状使いどころが思いつかなかったので、
リストの設定から入力必須を外しています。

- テーブル名
  - MedicationManagement
- 列名
  - MedicationDateTime：服薬日時
  - MedicationType：薬種類
  - 登録者：データ登録ユーザ（元から存在する列）

追加した列の設定は画像のように設定しています。
MedicationDateTimeはデータ種類を「日付と時刻」に設定し、入力必須としています。
MedicationTypeはデータ種類を「選択肢」に設定しており、薬の種類を増やす場合はSharePoint上で追加する仕様としました。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0fc5bf17-bb04-f85e-d090-b2933d5a724c.png" width="250">
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/98d4f5f8-3021-25d3-5dc7-d56aeaf6b7d1.png" width="250">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b6c5fab4-c4a9-c3a8-4c0f-d3986be4f336.png" width="600">

### データ登録UI部（Power Apps）

次に、服薬時にデータを登録するためのアプリケーションをPower Appsでつくっていきます。  
今回は一からの作成ではなく、SharePointリスト上からアプリを作成する方法でアプリの作成を行っています。  
なお、公式サイトの説明は以下に載っています。

https://learn.microsoft.com/ja-jp/power-apps/maker/canvas-apps/app-from-sharepoint#sharepoint-online-%E5%86%85%E3%81%8B%E3%82%89%E3%82%A2%E3%83%97%E3%83%AA%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B

1. SharePoint画面のメニュー上から、「統合」-「Power Apps」-「アプリの作成」を選択
2. アプリの名前を入力して「作成」を押す
3. しばらくするとPower Apps Studioが起動し、SharePointリストに基づいたアプリが作成される

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/86166f56-e33e-87b2-de87-12622700a329.png" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f1221b31-2b9d-fd39-a367-81f3713bb43e.png" width="400">

この時点で、

- 登録データの一覧表示画面
- 登録データの詳細表示画面
- 登録データの修正画面
- 新規データの登録画面
- それらを行き来するためのアイコンボタン

が自動で作成されているため、このまま使っても大きな問題はありませんが、一部画面や機能の追加を行っています。

#### トップメニューの追加

現状、一覧表示画面がアプリの初期画面となっている（初期画面をどれにするかは変更可能です）ため、別途アプリのトップメニュー画面を作りました。

トップメニューには、2つのボタンを配置し、一つが新規データ登録画面、もう一つが登録データの一覧表示画面に遷移するようにしています。  
あわせて、見た目的にわかりやすいよう、画面上にそれっぽいイラストをいくつか載せています。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/921b7905-3fa6-5fd2-fa7e-50696bcbe7df.png" width="400"> 

今回は実装していませんが、直近の登録データ3件程度を画面上に表示するといったことも後々挑戦してみたいなと考えています。

#### 一覧表示画面の日付範囲フィルタの追加

登録が増えてくると、特定の範囲のデータを探すのが難しくなるため、日付範囲フィルタの項目を追加しました。
実装方法については、以下のサイトを参考にさせていただきました。

https://jn.hateblo.jp/entry/2020/12/16/powerappsgalleryfiltering#%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3%EF%BC%92%E3%80%87%E6%9C%88%E3%80%87%E6%97%A5%E6%9C%88%E6%97%A5%E3%81%BE%E3%81%A7%E3%81%AE%E6%83%85%E5%A0%B1%E3%82%92%E8%A1%A8%E7%A4%BA%E3%81%99%E3%82%8B

日付フィルタについては、「X月X日～X月X日」という形式にしています。

実装方法としては、日付入力用の`DatePicker`を2つ用意し、それぞれに入力された値を用いて、`Gallery`の`Items`に以下の式を入力しています。
なお、`SortByColumns()`の箇所は昇順・降順の表示用の式のため、日付範囲フィルタを掛けているのは、`Filter()`関数の部分になります。

```Power Fx
SortByColumns(
    Filter([@MedicationManagement], MedicationDateTime >= DatePicker1.SelectedDate, MedicationDateTime <= DateAdd(DatePicker2.SelectedDate,1)),
    "MedicationDateTime", If(SortDescending1, SortOrder.Ascending, SortOrder.Descending))
```

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/b5602470-cb43-ddd5-8830-10f6e7415512.png" width="400">

#### その他

他に細かいところとして、

- 全体のフォントサイズ等の調整
- 新規データ登録時の現在日時をDefault値に設定する

といった対応を行っています。

## ◇おわりに

最低限の機能ですが、とりあえず自分で使って2か月程度経ちましたが特に問題なく使用できています。  
ただ、何度か飲み忘れが発生したため、やはりリマインダー機能は早めに追加したいなと感じているところです。

# 🔚
