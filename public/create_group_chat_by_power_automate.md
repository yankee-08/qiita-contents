---
title: Power Automate上でグループチャットを作成する時にハマった点の備忘録
tags:
  - PowerPlatform
  - PowerAutomate
  - Teams
  - 備忘録
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
---
## ◇はじめに

Microsoft社Power Platformの中のPower Automate上でグループチャットを作成しようとした際にフローが上手く動作せずにしばらくハマったため、その解決策を備忘録も兼ねて記事にしておきます。  
※常に発生する問題かは切り分けできていないので、もし同じ問題にぶつかった方は本記事を参考にしてもらえればと思います。

## ◇開発環境等

- OS：
  - Windows 11 Home（Ver:22H2）
- 契約ライセンス：
  - Microsoft 365 Business Basic
- 使用ツール：
  - Power Automate
  - SharePointリスト

## ◇元々やりたかったこと

1. SharePointリスト上にデータ種類を`ユーザーまたはグループ`とした列を作成する
2. このとき、`複数選択を許可`のチェックボックスを**オン**にする
3. 新規データ追加で、複数ユーザーを登録する
4. Power Automate上でSharePointリスト上からユーザー列のデータを取得する
5. ユーザー列に保存されていたユーザー全員を含めたグループチャットを作成する
6. 作成したグループチャットにメッセージを投稿する

## ◇最初に試したフロー（❌失敗例）

Power Automate上で新たなチャットを作成する場合、`チャットの作成`アクションを選択します。  
`チャットの作成`アクションを確認すると、
> セミコロンで区切ったユーザーIDまたはメールアドレスを入力します

と記載されており、セミコロン区切りの文字列が必要なことが分かります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/5a6bf3da-39a8-84e2-4031-ba38b5d0476c.png" width="600">

セミコロン区切りの文字列を作るには、`結合`アクションを使うことで対応できそうなことが分かったため、フローの構成としては、

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f0bd55d0-6a18-c730-6cc0-f2dd7f4cd1ec.png" width="600">

1. アレイ型の変数を用意
2. SharePointリストのユーザー列の項目からメールアドレスを人数分取得して、要素ごとに格納
3. アレイ型変数をセミコロン区切りの文字列変数に変換

といった流れにしました。

全体のフロー画面を以下に貼っておきます。  
アレイ型の変数は`UserMailList`という名前にしてあります。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/164b567b-0736-1e58-9025-2daa22f1c9c0.png" width="600">

ここまでで、チャットの作成に必要な文字列ができたため、`チャットの作成`、`チャットまたはチャネルでメッセージを投稿する`アクションを並べます。  
なお、とりあえずチャットのタイトルは`guid()`関数でランダム文字列化にしてますが、適宜変更可能です。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/fe20558c-a97f-c17d-0c37-7cc98082600a.png" width="600">

このとき、`チャットの作成`アクションの追加するメンバー欄には、先ほどの`結合`アクションの`出力`ブロックを入力します。  
また、メッセージを投稿するアクションでは、Group chat欄にカスタム値として`会話ID`を選択します。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f7c3929f-6500-f3d5-4b9b-65f589659aae.png" width="600">

これで一通りフローが完成したので、手動テストを行ったところ、`チャットを作成する`アクションでエラー（`BadRequest`）が発生しました。
文字列の結合処理に問題があるかと思い、調べてみましたがその部分は問題なさそうでした。

ネットで色々調べると、フォーラムでも似たような報告は以前からされているようです。  

https://powerusers.microsoft.com/t5/Building-Flows/Teams-Create-a-chat-Bad-Request-400-error/td-p/1011404

既に改善された、いや、まだ問題が発生する  
といったコメントがあって最終的な状態はよく分かりませんでしたが、その中にメールアドレスじゃなくてユーザー名を使うと解決するよといったコメントがあったため、そちらのやり方を試してみました。

## ◇改善したフロー（⭕成功例）

先ほどまでのフローをベースに、UPN名をリスト化するフローを追加しています。

アクションとしては、`ユーザーの検索(V2)`アクションを使用します。  
検索語句は先ほど同様、ユーザーメールアドレスを使用していますが、DisplayNameとかでも問題ありません。
ユーザー検索後、その中からUPN名を取得して配列変数に格納していきます（変数名は`UserMailList`から`UserList`に変更）。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/71cfa393-9ce3-65c1-0b5d-28641506884f.png" width="600">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/9999fba6-e72b-3d37-7311-689bfa5e51a7.png" width="600">

このあとはさっきのフロー同様、`チャットの作成`、`チャットまたはチャネルでメッセージを投稿する`アクションを並べて実行することで問題なくフローが動作するようになりました。

## ◇おわりに

メールアドレスを使った際にエラーが出る原因は現時点でも解決していませんが、UPN名を使うことで対処できることが分かったので、備忘録として残しておきます。

同様の問題が出る場合、出ない場合の条件がありそうですが、もし同様の問題が発生した場合は今回のやり方を試してもらえればと思います。

# 🔚END
