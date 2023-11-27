---
title: Djangoで作った備忘録登録WEBアプリの高機能化①（タイトル・タグ・本文検索機能の付加）
tags:
  - Python
  - JavaScript
  - Django
  - Web
  - Ajax
private: false
updated_at: '2023-11-28T00:55:05+09:00'
id: ce4e38346de880f7eabe
organization_url_name: null
slide: false
ignorePublish: false
---
# ◇はじめに

前回のこの記事（[WEBアプリの勉強を兼ねてDjangoで備忘録登録アプリを作ってみる](https://qiita.com/yankee/items/304f58273fb676f51f7a)）からの続きになります。

# ◇記事投稿順（2019/06/29追記）

今回は、ベースのWEBアプリを1.の記事で作成し、2.以降の記事で機能の追加や改善を行っています。必要に応じてほかの記事も参照ください。

1. [WEBアプリの勉強を兼ねてDjangoで備忘録登録アプリを作ってみる](https://qiita.com/yankee/items/304f58273fb676f51f7a)
2. 【本記事】Djangoで作った備忘録登録WEBアプリの高機能化①（タイトル・タグ・本文検索機能の付加）
3. [Djangoで作った備忘録登録WEBアプリの高機能化②（タグ一覧表示、タグ別記事の追加など）](https://qiita.com/yankee/items/4e387b8ec4c5a4053c89)

# ◇今回追加した機能

前回の記事の最後に課題として挙げていた、以下の項目のうち、タイトル検索、タグ検索機能の実装をおこないました。

> + [x] タイトル検索、タグ検索機能の実装（現状、タグが役に立っていない・・）。
> + [ ] タグの管理メニュー（現状追加しかできない・・）。
> + [ ] 新規タグ追加画面からタグを追加した場合にtopページに飛んでしまうため、その点の修正。
> + [ ] フォームの改善（forms.pyの作成）。
> + [ ] CSSフレームワーク（Bootstrapなど）の導入（今回のアプリレベルでcssファイルが雑多になってきたため・・）  
    ⇒フレームワークどーせ入れるならSass（SCSS or SASS）も試してみたい。

当初、タグの管理メニューまで挑戦する予定でしたが、検索機能の実装でいろいろハマったポイントがあり、実装に時間がかかってしまったため、検索機能までとしています。
ハマったポイントについては、途中途中で記載していきます。

# ◇開発環境（前回と同じ）

+ OS : Ubuntu 18.04.2 LTS(Windows Subsystem for Linux)
+ 言語 : Python 3.6.7
+ Webアプリフレームワーク : Django (2.2)
+ DB : SQLite3

# ◇実装内容

今回の記事ではプログラミングした順にのっとり、

+ 外観部分（HTML）の作成　⇒
+ 検索データのPOST処理（JavaScript）　⇒
+ サーバ側での記事検索・該当記事の返答処理（Django）　⇒
+ レスポンスデータによるHTMLの部分更新（JavaScript）  
という流れで記事を書いています。

### ◆検索ボックス作成（HTML部分）

まず、トップページ（記事一覧表示）に検索ボックスを追加していきます。
検索ボックスの構成は、ドロップダウンリストで`タイトル検索`、`タグ検索`、`本文検索`を選択し、テキストボックスに検索ワードを入力して`Search`ボタンを押すと該当する記事のみが表示される流れとしています。

```Django:index.html

{% extends 'memorandum/base-layout.html' %}

{% load static %}

---略---

{% block content %}
  <div class="local-nav">
    <ul class="local-nav-list">
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create' %}">新規記事</a></li>
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create_tag' %}">新規TAG</a></li>
      <li class="local-search-item">
        <select id="js-search-item-list" name="search-item" size="1">
          <option value="title">タイトル検索</option>
          <option value="tag">タグ検索</option>
          <option value="body">本文検索</option>
        </select>
        <input id="js-search-text" type="search" placeholder="検索ワード">
        <button type="button" class="button-common" id="js-search-btn">Search</button>
      </li>
    </ul>
  </div>

---略---

{% endblock content %}

```

![001.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/51896838-40d8-6c16-20e5-2b6024cc654b.png)

これでとりあえず外観はできたので、次に`Search`ボタンを押したときの`JavaScript`の処理を実装していきます。

### ◆検索データPOST処理（JavaScript部分）

検索ボックスに入力したデータのPOST処理ですが、今回`Ajax`でのHTML部分更新（記事リストの部分のみ更新し、検索ボックス部分は更新させない）を行うため、`JavaScript`を追加しています。
そのため、前回のファイル構成から`js/index.js`を追加しています。

```:Terminal
mysite/
├── memorandum
    ├── static
        └── memorandum
            ├── css
            │   └── memorandum.css   <-スタイルシート用ファイル
            ├── js
                └── index.js   <-トップページ用JavaScript
```

`index.js`では、以下の順で処理を行います。

1. `Search`ボタンクリック時のイベントを追加する。
2. クリックされたら、ドロップダウンリストの選択項目（`item`）とテキストフォームの入力値（`value`）を取得する。
3. ２つのデータを`json`化して`fetch`関数を用いてPOSTする。

```JavaScript:index.js（未完成）
const searchItemList = document.getElementById("js-search-item-list");
const searchText = document.getElementById("js-search-text");
const searchBtn = document.getElementById("js-search-btn");

searchBtn.addEventListener("click", function() {
    console.log("hoge");
    const item = searchItemList.value;
    const text = searchText.value;
    postSearchText(item, text);
}, false);

async function postSearchText(searchItem, searchText){
    const postBody = {
        item: searchItem,
        text: searchText,
    };
    console.log(postBody);
    const postData = {
        method: "POST",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify(postBody)
    };
    console.log(postData);
    const res = await fetch("./", postData)
    console.log(res.json());
}

```

上の`index.js`はデータをPOSTするところまでのコードです。
とりあえずこの状態で`Django`側にPOSTデータを受けるだけのコードを追加し、データが正しくPOSTされているかのテストを行いましたが、`Search`ボタンをクリックすると`403 (Forbidden)`エラーが発生しました。

#### □`403 (Forbidden)`の発生（ハマったポイント①）

ドキュメントなどで調べたところ、POSTフォームを__内部 URL__に対して行う際には基本的にCSRFトークンを付けてあげる必要があり、それをつけていないことが原因でした（逆に__外部URL__にPOSTする際はトークンが外部に漏れるためつけてはいけない）。
ちなみに、`Django`のテンプレートからPOSTフォームする場合は以下のよう`{% csrf_token %}`を付けるだけでOKです。
_参考URL_：[クロスサイトリクエストフォージェリ (CSRF) 対策](https://docs.djangoproject.com/ja/2.2/ref/csrf/#ajax)

```Django
<form method="post">{% csrf_token %}
```
  
今回は、`Ajax`でフォームのPOSTを行おうとしているため、その場合のCSRFトークン付加の処理を実装します。
※Django側でCSRFトークンのチェックを無効化する方法もありそうでしたが、根本解決にはならないため、今回は見送りました

#### □`Ajax`使用時のCSRFトークン付加

実装にあたっては、以下のサイトを参考にしました。

_参考URL_：[クロスサイトリクエストフォージェリ (CSRF) 対策](https://docs.djangoproject.com/ja/2.2/ref/csrf/#ajax)
_参考URL_：[DjangoでPOSTメッセージにCSRFtokenを含ませる方法まとめ](https://narito.ninja/blog/detail/88/#csrf)
_参考URL_：[A simple, lightweight JavaScript API for handling browser cookies ](https://github.com/js-cookie/js-cookie/)

> 以下、[クロスサイトリクエストフォージェリ (CSRF) 対策](https://docs.djangoproject.com/ja/2.2/ref/csrf/#ajax) ページからの引用です。
>> 上記のコードは JavaScript Cookie library を使って getCookie を置き換えればシンプルにできます:
>> `var csrftoken = Cookies.get('csrftoken');`

`JavaScript Cookie library`を使うと、CSRFトークンの取得が簡単にできるため、今回はこのライブラリを使用しました。
ライブラリのダウンロードはCDNを用いてダウンロードしていますので、以下のように`index.html`に追記しています。
※npm経由で手動ダウンロードも可能です。それぞれの手順については、[A simple, lightweight JavaScript API for handling browser cookies ](https://github.com/js-cookie/js-cookie/)を参照

```Django:index.html
{% extends 'memorandum/base-layout.html' %}

{% load static %}

{% block head %}
  <script src="https://cdn.jsdelivr.net/npm/js-cookie@2/src/js.cookie.min.js"></script>
  ↑ 追記部分
  <script src="{% static 'memorandum/js/index.js' %}" defer></script>
{% endblock head %}

---略---
```

`index.js`では、CSRFトークンの取得とPOSTのヘッダーにそのトークンを追加する処理を追記しました。

```JavaScript:index.js（未完成）
const csrftoken = Cookies.get('csrftoken');  <- 追記部分

---略---

async function postSearchText(searchItem, searchText){
    const postBody = {
        item: searchItem,
        text: searchText,
    };
    console.log(postBody);
    const postData = {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "X-CSRFToken": csrftoken,  <- 追記部分
        },
        body: JSON.stringify(postBody)
    };
    console.log(postData);
    const res = await fetch("./", postData)
    console.log(res.json());
}
```

これで、`Search`ボタンを押すと、検索用データが`Django`側にPOSTできるようになりました。
つづいて、`Django`側の処理を実装していきます。

### ◆記事検索・該当記事の返答処理（Django）

`views.py`内の`ArticleListView`クラス内にPOSTがきた場合の処理を追加で実装します。
POSTデータの`item`の値によってフィルタをかける項目を分けています。

記事の検索（フィルタリング）については、以下のサイトを参考にしました。

_参考URL_：[Django データベース操作 についてのまとめ](https://qiita.com/okoppe8/items/66a8747cf179a538355b#%E5%AF%BE%EF%BD%8E%E3%83%AA%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E6%93%8D%E4%BD%9C%E5%B0%82%E7%94%A8%E3%83%A1%E3%82%BD%E3%83%83%E3%83%89)
_参考URL_：[ManyToMany フィルターなどのオブジェクト操作一覧](https://www.djangobrothers.com/blogs/many_to_many_objects/)
_参考URL_：[クエリを作成する](https://docs.djangoproject.com/ja/2.1/topics/db/queries/#)

POSTデータの`text`の値を検索ワードとして、検索条件は__部分一致（大文字小文字区別無し）__としています。
※SQLiteの場合は大文字小文字の区別有りの検索条件としても大文字小文字の区別がされないので注意
また、タグ検索については、`Article`モデルと`Tag`モデルを`ManyToManyField`でつなげていますが、この場合も`tag__name`というような書き方で検索をかけることが可能でした。
_参考URL_：[リレーションを横断するルックアップ](https://docs.djangoproject.com/ja/2.1/topics/db/queries/#lookups-that-span-relationships)

最後に、検索に該当する記事のデータを`json`化して、レスポンスを行います。

```Python:views.py
class ArticleListView(generic.ListView):
    model = Article

    # 参照するhtmlファイルを指定
    template_name = "memorandum/index.html"

    def post(self, request, *args, **kwargs):
        json_body = request.body.decode("utf-8")
        body = json.loads(json_body)

        item = body["item"]
        text = body["text"]

        if item == "title":
            print("Search title word={}".format(text))
            model_data = self.model.objects.filter(title__icontains=text)
        elif item == "tag":
            print("Search tag word={}".format(text))
            model_data = self.model.objects.filter(tag__name__icontains=text)
        elif item == "body":
            print("Search body word={}".format(text))
            model_data = self.model.objects.filter(body__icontains=text)
        else:
            print("Search ??? word={}".format(text))
            model_data = self.model.objects.all()

        json_data = serializers.serialize("json", model_data, ensure_ascii=False, indent=2)

        print("json_data:{}".format(type(json_data)))
        print("json_data:{}".format(json_data))

        return JsonResponse(json_data, safe=False)
        
```

ここまでで、`Django`からレスポンスが返され、先ほど実装した`index.js`でそのレスポンスがコンソール出力される状態になったはずでした。
が、実際にコンソールを確認すると、`res.json()`の中身が空っぽになりました。

```JavaScript:index.js（未完成）
    ---略---
    const res = await fetch("./", postData)
    console.log(res.json());
}
```

#### □`Django`からの応答の中身が空っぽ（ハマったポイント②）

この問題については、

+ `Django`側での`JsonResponse()`の引数の指定の仕方が悪かったのか？
+ レスポンスのデータの中身を`res.json()`でうまくデコードできなかったのか？

など色々調べましたが、原因を特定するのに３日ほどかかってしまいました、、

結論としては、（答えがわかれば単純ですが）`res.json()`に__`await`__を付けていなかったことが原因でした。
__`await`__をつけることにより、レスポンスのデータが正しく出力されることが確認できました。

```JavaScript:index.js（未完成）
    ---略---
    const res = await fetch("./", postData)
    const json = await res.json();
    console.log(json);
}
```

<details><summary>原因特定に時間がかかった~~理由~~言い訳・・・（本筋から外れるので、折りたたみにします）</summary><div>

+ `fetch`関数のようなサーバにアクセスするもの（メッセージを送る感じのもの）については、`await`を入れないと応答があるまで待機してくれないとは思っていたが、メソッドを呼び出す場合は常にそのメソッド処理の完了を待つものだと思い込んでいた。
+ 元々C言語から学び始めたため、関数呼び出しの場合はその関数が終わるまで次の処理に進むというイメージがつかみにくかった。  
というのが言い訳です。。  

今回のことをうけて、`async/await`演算子について、もう少し理解する必要があると感じました。
なんとなくの感じで`await`演算子をつけていましたが、どの処理には必要かというのを理解していないと、_とりあえず`await`つけとけ！_という感じになってしまいそうなので・・
_参考URL_：[async/await地獄](https://qiita.com/rana_kualu/items/e6c5c0e4f60b0d18799d)
</div></details>

### ◆レスポンスデータによるHTMLの部分更新（JavaScript）

最後に`Django`からのレスポンスを受け取り、HTMLを更新します。
`index.html`では、要素の更新を行うための基準となる、記事リストの`ul`タグに`id`を追加します。
また、年月日の表示形式を`Y年m月d日(D)`から`Y-m-d`に変更しています。これは、`Ajax`でPOSTした際のレスポンスデータの年月日の表示形式に合わせるためです（`JavaScript`側で年月日の表示形式を合わせることもできそうでしたが、そこまでこだわりがなかったため、簡単に修正できる方法を選択しました）。

```Django:index.html
---略---

{% block content %}

  ---略---

  <ul id="js-article-list" class="article-list">　　<- idの追加
    {% for item in object_list %}
      <li class="article-list-item">
        <a class="link-text" href="{% url 'memorandum:detail' item.pk %}">{{ item.title }}</a>
        {% comment %} <div class="article-date">作成日: {{ item.published_date|date:"Y年m月d日(D)" }}<br>更新日: {{ item.modified_date|date:"Y年m月d日(D)" }}</div> {% endcomment %}
        <div class="article-date">作成日: {{ item.published_date|date:"Y-m-d" }}<br>更新日: {{ item.modified_date|date:"Y-m-d" }}</div>　　<- 年月日の表示フォーマットを変更
      </li>
    {% endfor %}
  </ul>
{% endblock content %}
---略---
```

`index.js`では、必要な要素をつくり、そこにクラスの適用やテキストの代入を行い、最終的に`appendChild`で子要素を追加しています。
なお、記事詳細へのリンクのURL部分は元々`Django`のテンプレート言語で生成されていたため、今回はレスポンスデータの`pk`や`title`を利用して、`JavaScript`側で以下のように生成しています。
`const linkText = "/memorandum/" + String(id) + "/detail/";`  
※URL生成部分については、サーバ側のURLパスの変更にともなってこの部分も変更する必要がでてしまうため、もう少しスマートなやり方がないかとは思いましたが、良案が思いつかなかったため、この実装にしています

+ _参考書籍_：[JavaSccriptコードレシピ集](https://gihyo.jp/book/2019/978-4-297-10368-2)

```JavaScript:index.js

---略---

const articleListElement = document.getElementById("js-article-list");

---略---

async function postSearchText(searchItem, searchText) {
    const postBody = {
        item: searchItem,
        text: searchText,
    };
    const postData = {
        method: "POST",
        headers: {
            // "Content-Type": "application/x-www-form-urlencoded",
            "Content-Type": "application/json",
            "X-CSRFToken": csrftoken,
        },
        body: JSON.stringify(postBody)
    };
    console.log(postData);
    let res = await fetch("./", postData)
    console.log(res.statusText, res.url);

    const json = await res.json();
    // console.log(json);
    const filteredArticles = await JSON.parse(json);
    // console.log(filteredArticles);

    while( articleListElement.firstChild )
    {
        articleListElement.removeChild(articleListElement.firstChild);
    }

    for(let article of filteredArticles)
    {
        console.log("pk:", article.pk);
        console.log("title:", article.fields.title);
        console.log("published_date:", article.fields.published_date);
        console.log("modified_date:", article.fields.modified_date);
        createFilteredElement(article.pk, article.fields.title, article.fields.published_date, article.fields.modified_date);
    }

}

function createFilteredElement(id, title, publishedDate, modifiedDate) {
    // async function createFilteredElement(title, link, publishedDate, modifiedDate) {
    const listItemElement = document.createElement("li");
    listItemElement.classList.add("article-list-item");
    console.log(listItemElement);

    const linkText = "/memorandum/" + String(id) + "/detail/";
    console.log(linkText);
    const listLinkElement = document.createElement("a");
    listLinkElement.classList.add("link-text");
    listLinkElement.setAttribute("href", linkText);
    listLinkElement.textContent = title;
    console.log(listLinkElement);

    const listDateElement = document.createElement("div");
    listDateElement.classList.add("article-date");
    listDateElement.innerHTML = "作成日:"+publishedDate+"<br>更新日:"+modifiedDate;
    console.log(listDateElement);

    listItemElement.appendChild(listLinkElement);
    listItemElement.appendChild(listDateElement);

    articleListElement.appendChild(listItemElement);
}
```

これでひととおりプログラムが完成したので、実際に記事の検索がおこなえるか確認を行いました。
その結果、検索ワードに該当する記事のリスト表示はできましたが、一部文字化けが発生してしまいました。

#### □`Ajax`による部分更新を行った際に文字化けが発生（ハマったポイント③）

当初、HTMLや`JavaScript`の文字コードが`UTF-8`に統一されていないためと思われましたが、問題が解決しません。
その後もいろいろ調べた結果、`Firefox`では文字化けは起きず、`Chrome`だと文字化けが発生することが判明し、それをもとにネット情報を調べると以下のサイトが見つかりました。

_参考URL_：[Chrome 57 からエンコーディングの自動判別がダメになった](http://var.blog.jp/archives/70125676.html)

この記事によると、サーバ側のレスポンスヘッダーに`UTF-8`の文字コードを設定してあげれば解決可能とのことでしたので、実際にコードを修正したところ`Chrome`での文字化けがなくなりました。
（以下は、タグ検索で`Django`を検索ワードとして検索した例）

![003.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/e8ce215e-5591-5088-6697-ef9355ed7d06.png)

### ◆今回修正したファイルのソースコード

<details><summary>ソースコードが小出しになってしまったため、一応今回修正したファイルのコード一覧を折りたたみで載せます</summary><div>

```Django:index.html
{% extends 'memorandum/base-layout.html' %}

{% load static %}

{% block head %}
  <script src="https://cdn.jsdelivr.net/npm/js-cookie@2/src/js.cookie.min.js"></script>
  <script src="{% static 'memorandum/js/index.js' %}" charset="UTF-8" defer></script>
{% endblock head %}

{% block content %}
  <div class="local-nav">
    <ul class="local-nav-list">
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create' %}">新規記事</a></li>
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create_tag' %}">新規TAG</a></li>
      <li class="local-search-item">
        <select id="js-search-item-list" name="search-item" size="1">
          <option value="title">タイトル検索</option>
          <option value="tag">タグ検索</option>
          <option value="body">本文検索</option>
        </select>
        <input id="js-search-text" type="search" placeholder="検索ワード">
        <button id="js-search-btn" type="button" class="button-common">Search</button>
      </li>
    </ul>
  </div>
  <ul id="js-article-list" class="article-list">
    {% for item in object_list %}
      <li class="article-list-item">
        <a class="link-text" href="{% url 'memorandum:detail' item.pk %}">{{ item.title }}</a>
        {% comment %} <div class="article-date">作成日: {{ item.published_date|date:"Y年m月d日(D)" }}<br>更新日: {{ item.modified_date|date:"Y年m月d日(D)" }}</div> {% endcomment %}
        <div class="article-date">作成日: {{ item.published_date|date:"Y-m-d" }}<br>更新日: {{ item.modified_date|date:"Y-m-d" }}</div>
      </li>
    {% endfor %}
  </ul>
{% endblock content %}
```

```JavaScript:index.js

const searchItemListElement = document.getElementById("js-search-item-list");
const searchTextElement = document.getElementById("js-search-text");
const searchBtnElement = document.getElementById("js-search-btn");

const articleListElement = document.getElementById("js-article-list");

const csrftoken = Cookies.get('csrftoken');
console.log(csrftoken);

const x = document.characterSet;
console.log(x)

searchBtnElement.addEventListener("click", function() {
    console.log("hoge");
    const item = searchItemListElement.value;
    const text = searchTextElement.value;
    postSearchText(item, text);
}, false);

async function postSearchText(searchItem, searchText) {
    const postBody = {
        item: searchItem,
        text: searchText,
    };
    const postData = {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "X-CSRFToken": csrftoken,
        },
        body: JSON.stringify(postBody)
    };
    console.log(postData);
    let res = await fetch("./", postData)
    console.log(res.statusText, res.url);

    const json = await res.json();
    // console.log(json);
    const filteredArticles = await JSON.parse(json);
    // console.log(filteredArticles);

    while( articleListElement.firstChild )
    {
        articleListElement.removeChild(articleListElement.firstChild);
    }

    for(let article of filteredArticles)
    {
        console.log("pk:", article.pk);
        console.log("title:", article.fields.title);
        console.log("published_date:", article.fields.published_date);
        console.log("modified_date:", article.fields.modified_date);
        createFilteredElement(article.pk, article.fields.title, article.fields.published_date, article.fields.modified_date);
    }

}

function createFilteredElement(id, title, publishedDate, modifiedDate) {
    const listItemElement = document.createElement("li");
    listItemElement.classList.add("article-list-item");
    console.log(listItemElement);

    const linkText = "/memorandum/" + String(id) + "/detail/";
    console.log(linkText);
    const listLinkElement = document.createElement("a");
    listLinkElement.classList.add("link-text");
    listLinkElement.setAttribute("href", linkText);
    listLinkElement.textContent = title;
    console.log(listLinkElement);

    const listDateElement = document.createElement("div");
    listDateElement.classList.add("article-date");
    listDateElement.innerHTML = "作成日:"+publishedDate+"<br>更新日:"+modifiedDate;
    console.log(listDateElement);

    listItemElement.appendChild(listLinkElement);
    listItemElement.appendChild(listDateElement);

    articleListElement.appendChild(listItemElement);
}

```

```Python:views.py
from django.shortcuts import render, redirect
from django.urls import reverse_lazy
from django.http import HttpResponse, JsonResponse
from django.views import generic
from django.core import serializers
import json
import time

from .models import Tag, Article


class ArticleListView(generic.ListView):
    model = Article

    # 参照するhtmlファイルを指定
    template_name = "memorandum/index.html"

    def post(self, request, *args, **kwargs):
        json_body = request.body.decode("utf-8")
        body = json.loads(json_body)

        item = body["item"]
        text = body["text"]

        if item == "title":
            print("Search title word={}".format(text))
            model_data = self.model.objects.filter(title__icontains=text)
        elif item == "tag":
            print("Search tag word={}".format(text))
            model_data = self.model.objects.filter(tag__name__icontains=text)
        elif item == "body":
            print("Search body word={}".format(text))
            model_data = self.model.objects.filter(body__icontains=text)
        else:
            print("Search ??? word={}".format(text))
            model_data = self.model.objects.all()

        json_data = serializers.serialize("json", model_data, ensure_ascii=False, indent=2)

        print("json_data:{}".format(type(json_data)))
        print("json_data:{}".format(json_data))

        # return JsonResponse(json_data, safe=False)
        return JsonResponse(json_data, content_type="application/json; charset=utf-8", safe=False)


class ArticleDetailView(generic.DetailView):
    model = Article
    # 参照するhtmlファイルを指定
    template_name = "memorandum/detail.html"


class ArticleCreateView(generic.edit.CreateView):
    model = Article
    fields = ["title", "tag", "body"]
    # 参照するhtmlファイルを指定
    template_name = "memorandum/create.html"
    success_url = reverse_lazy('memorandum:index')  # POSTが正しく行われた際に飛ばすURL


class ArticleUpdateView(generic.edit.UpdateView):
    model = Article
    fields = ["title", "tag", "body"]
    # 参照するhtmlファイルを指定
    template_name = "memorandum/update.html"
    success_url = reverse_lazy('memorandum:index')  # POSTが正しく行われた際に飛ばすURL


class ArticleDeleteView(generic.edit.DeleteView):
    model = Article
    # 参照するhtmlファイルを指定
    template_name = "memorandum/delete.html"
    success_url = reverse_lazy('memorandum:index')  # POSTが正しく行われた際に飛ばすURL


class TagCreateView(generic.edit.CreateView):
    model = Tag
    fields = ["name"]
    # 参照するhtmlファイルを指定
    template_name = "memorandum/create-tag.html"
    success_url = reverse_lazy('memorandum:index')  # POSTが正しく行われた際に飛ばすURL

    def form_invalid(self, form):
        print("---form_invalid Called")
        print(form)
        print("form_invalid Called---")
        return super().form_invalid(form)
        # return redirect("memorandum:create_tag")
```

</div></details>

# ◇おわりに

+ [前回の投稿](https://qiita.com/yankee/items/304f58273fb676f51f7a)から、記事検索機能を実装できました。
+ 今回は、前回以上にいろいろとハマってしまい完成までに時間がかかりましたが、今後もマイペースで機能追加していきたいと思います（その際は、今回のようにハマったポイントもあわせて掲載していければと思います）。
+ つぎはタグの管理メニュー（削除、一覧表示など）関係を実装していければと思います。

# 🔚
