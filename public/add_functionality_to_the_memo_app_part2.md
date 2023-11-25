---
title: Djangoで作った備忘録登録WEBアプリの高機能化②（タグ一覧表示、タグ別記事の追加など）
tags:
  - Python
  - JavaScript
  - Django
  - Webアプリケーション
  - 備忘録
private: false
updated_at: '2023-04-08T21:04:57+09:00'
id: 4e387b8ec4c5a4053c89
organization_url_name: null
slide: false
ignorePublish: false
---
# ◇はじめに

過去に投稿した、

1. [WEBアプリの勉強を兼ねてDjangoで備忘録登録アプリを作ってみる](https://qiita.com/yankee/items/304f58273fb676f51f7a)
2. [Djangoで作った備忘録登録WEBアプリの高機能化①（タイトル・タグ・本文検索機能の付加）](https://qiita.com/yankee/items/ce4e38346de880f7eabe)
からの続きになります。

# ◇記事投稿順（2019/06/29追記）

今回は、ベースのWEBアプリを1.の記事で作成し、2.以降の記事で機能の追加や改善を行っています。必要に応じてほかの記事も参照ください。

1. [WEBアプリの勉強を兼ねてDjangoで備忘録登録アプリを作ってみる](https://qiita.com/yankee/items/304f58273fb676f51f7a)
2. [Djangoで作った備忘録登録WEBアプリの高機能化①（タイトル・タグ・本文検索機能の付加）](https://qiita.com/yankee/items/ce4e38346de880f7eabe)
3. 【本記事】Djangoで作った備忘録登録WEBアプリの高機能化②（タグ一覧表示、タグ別記事の追加など）

# ◇今回追加した機能

今回は以下の項目のうち、タグの管理メニュー、タグ追加後の画面遷移処理修正について実装をおこないました。

> + [x] <font color="darkgray"> タイトル検索、タグ検索機能の実装（現状、タグが役に立っていない・・）。</font>  
> -> [こちら](https://qiita.com/yankee/items/ce4e38346de880f7eabe)で実装済み。
> + [x] <font color="tomato"> タグの管理メニュー（現状追加しかできない・・）。</font>
> + [x] <font color="tomato"> 新規タグ追加画面からタグを追加した場合にtopページに飛んでしまうため、その点の修正。</font>
> + [ ] フォームの改善（forms.pyの作成）。
> + [ ] CSSフレームワーク（Bootstrapなど）の導入（今回のアプリレベルでcssファイルが雑多になってきたため・・）  
    ⇒フレームワークどーせ入れるならSass（SCSS or SASS）も試してみたい。

タグ管理としては、タグの一覧表示（使用されている記事が多い順に表示）、記事中のタグをクリックするとそのタグが登録されている記事一覧を表示する（Qiitaの記事のイメージ）といった機能を実装していきます。

# ◇開発環境（前回と同じ）

+ OS : Ubuntu 18.04.2 LTS(Windows Subsystem for Linux)
+ 言語 : Python 3.6.7
+ Webアプリフレームワーク : Django (2.2)
+ DB : SQLite3

# ◇実装内容

今回の記事では、

+ タグ一覧表示ページの作成
+ タグクリック時のタグ登録記事一覧表示対応
+ 新規タグ登録ページの画面遷移処理修正

という流れで記事を書いています。

### ◆タグ一覧表示ページの作成

トップページ（記事一覧表示）のとき同様、クラスベースビューの中の`ListView`を使えばそんなに時間をかけずに実装できそうでしたが、今回はあえて、大元のクラスである`View`を使ってタグ一覧表示ページの作成に挑戦しました。
ちなみに、`View`クラスは`ListView`クラスの親クラスである`BaseListView`の親クラスになり、`CreateView`クラスや`UpdateView`クラスも`View`クラスを継承しています（ソースコード追っていったらそうなってた）。

#### □`urls.py`へのルーティング追加

まず、`urls.py`にルーティングを追加します。`View`クラスでも`as_view()`はサポートされているので、今までと同じ書き方です。

```Python:urls.py
---略---

urlpatterns = [
    ---略---
    path("<int:pk>/update/", views.ArticleUpdateView.as_view(), name="update"),
    path("<int:pk>/delete/", views.ArticleDeleteView.as_view(), name="delete"),
    path("tag/create/", views.TagCreateView.as_view(), name="create_tag"),
    path("tag/list/", views.TagListView.as_view(), name="list_tag"),  <-追記部分
]
```

#### □`views.py`へのビュークラス追加

次に、`views.py`に`TagListView`クラスを定義していきます。
GETリクエスト時の処理をget関数で定義します。get関数では、`Tag`モデルからデータを取得していますが、このときに、

+ __それぞれ__のタグと紐付いている記事数を集計する。（`annotate( **** Count('article'))`部分）
+ その集計した値を`post_count`という名前で指定できるようにする。（`post_count=Count('article')`部分）
+ 紐付いている記事が多い順に並びかえる。（`order_by('-post_count')`部分）
といったことを行っています。

`post_count`の部分はイマイチ理解しきれていませんが、（自分の解釈では）`annotate()`の関数で指定した条件で集計（`Count`、`Avg`、`Max`など）したものを擬似的なフィールドに代入していて、そのフィールド名を`post_count=`で指定しているのかなという理解です。
_参考URL_：[Aggregation](https://docs.djangoproject.com/en/2.2/topics/db/aggregation/)
_参考URL_：[Djangoで、集計処理](https://narito.ninja/blog/detail/84/)
_参考URL_：[Django データベース操作 についてのまとめ](https://qiita.com/okoppe8/items/66a8747cf179a538355b#%E9%96%A2%E4%BF%82%E5%85%83%E3%83%86%E3%83%BC%E3%83%96%E3%83%AB%E3%81%AE%E4%BB%98%E5%B1%9E%E6%83%85%E5%A0%B1%E3%81%A8%E3%81%97%E3%81%A6%E9%96%A2%E4%BF%82%E5%85%88%E3%83%86%E3%83%BC%E3%83%96%E3%83%AB%E3%81%AE%E5%80%A4%E3%82%92%E9%9B%86%E8%A8%88%E3%81%99%E3%82%8B)

そして、その取得したデータ（`tag_list`）を辞書にいれ、`render`でレスポンスを返しています。
_参考URL_：[ショートカット: render()](https://docs.djangoproject.com/ja/2.2/intro/tutorial03/#a-shortcut-render)

```Python:views.py
---略---

class TagListView(generic.View):
    model = Tag

    def get(self, request, *args, **kwargs):
        tag_list = self.model.objects.annotate(post_count=Count('article')).order_by('-post_count')
        for tag in tag_list:
            print(tag.name)
            print(tag.post_count)
        context = {
            "object_list": tag_list,
        }
        return render(
            request=request,
            template_name="memorandum/list-tag.html",
            context=context
        )
```

#### □テンプレートファイル、cssファイルの追加・修正

次に、テンプレートファイル（`list-tag.html`）の新規作成とcssセレクタを追加していきます。
構成は今まであまり変わりませんが、取得したモデルのデータリスト（`object_list`）の項目をforループで取得してタグ名とタグが登録されている記事数を表示しています。
このとき、記事数については先ほどの`post_count`で指定することができます。
cssについては、追加したものだけ以下に載せます（`display: flex;`だらけになってしまったんで、ホントは共通化すべきかも・・・）。

```Django:list-tag.html
{% extends 'memorandum/base-layout.html' %}

{% block content %}
  <ul class="tag-list">
    {% for item in object_list %}
      <li class="tag-list-item">
        <div class="tag-list-text text-reddish">{{ item.name }}</div>
        <div class="tag-list-text background-reddish text-whitish small-circle">{{ item.post_count }}</div>
      </li>
    {% endfor %}
  </ul>
{% endblock content %}
```

```css:memorandum.css
---略---

.text-reddish {
    color: tomato;
}

.text-whitish {
    color: white;
}

.background-reddish {
    background: tomato;
}

.background-whitish {
    background: white;
}

.tag-list {
    align-items: center;
    display: flex;
    flex-wrap: wrap;
    justify-content: flex-start;
    list-style-type: none;
    margin: 5px 0;
    padding: 30px 20px;
}

.tag-list-item {
    align-items: center;
    background: mistyrose;
    color: tomato;
    display: flex;
    flex-wrap: nowrap;
    justify-content: center;
    margin: 20px 10px;
    padding: 5px;
}

.tag-list-text {
    align-items: center;
    display: flex;
    justify-content: center;
    height: 25px;
    padding: 2px;
}

.small-circle {
    border-radius: 50%;
    width: 25px;
}
```

これでタグ一覧表示ページ（`list-tag.html`）ができましたので、トップページ（`index.html`）にTAG一覧ボタンを追加し、タグ一覧表示ページにアクセスできるようにします。

```Django:index.html
---略---

{% block content %}
  <div class="local-nav">
    <ul class="local-nav-list">
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create' %}">新規記事</a></li>
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:list_tag' %}">TAG一覧</a></li>
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create_tag' %}">新規TAG</a></li>

---略---
{% endblock content %}
```

記事数順にタグを表示して、タグ名の後ろに記事数を表示しています。

![005.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/dc46cd85-bdde-9635-aeb4-2a8495a7829b.png)
![004.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/596a31f8-a1dc-8d02-cad5-17c3c8e57ff2.png)

これで、タグ一覧表示ページができましたので、次にそれぞれのタグをクリックした際に記事一覧を表示する仕組みをつくっていきます。

### ◆タグクリック時のタグ登録記事一覧表示対応

つづいて、さきほど作成したタグ一覧表示ページ（`list-tag.html`）と最初の記事で作成した記事詳細表示ページ（`detail.html`）内にあるタグ名をクリックすると、そのタグが登録されている記事の一覧を表示するようにしていきます。

#### □`views.py`の修正

まず、タグ名をクリックした場合の処理の実装です。
今回は、トップページ（記事一覧表示）に処理を追加していく方針としました（クラス名でいうと`ArticleListView`）。
なお、このクラスには、すでにPOST時の処理は`Ajax`用の処理が実装されてしまっているので、今回はGET時にパラメータをつけてタグ名を受け取る形としました。

`ArticleListView`クラスの`get_queryset()`関数をオーバーライドし、まず元（親クラス）の`get_queryset()`を実行します。
その後、GETパラメータ内に`"tag"`という項目名がある場合、その項目の値でフィルタリングし、その結果を返します。
※GETパラメータ内に`"tag"`という項目名がない場合は元の結果を返す（元々の`get_queryset()`関数と同じ挙動になる）ため、単純にトップページにアクセスした場合はすべての記事が表示されます

```Python:views.py
---略---

class ArticleListView(generic.ListView):
    model = Article

    # 参照するhtmlファイルを指定
    template_name = "memorandum/index.html"

    ---略---

    def get_queryset(self):
        query_set = super().get_queryset()
        if "tag" in self.request.GET:
            query_set = query_set.filter(tag__name__icontains=self.request.GET.get("tag"))
            print("get_queryset tag={}".format(self.request.GET.get("tag")))
        else:
            print("get_queryset tag NOT exist")
        return query_set

---略---

```

#### □`list-tag.html`/`detail.html`への`a`タグの追加

つづいて、タグ一覧表示ページ（`list-tag.html`）と最初の記事で作成した記事詳細表示ページ（`detail.html`）それぞれに`a`タグを追加して、クリック時にトップページに飛ぶように変更します。
このとき、
`href="{% url 'memorandum:index' %}?tag={{ item.name }}"`
として、GETパラメータとしてタグ名をつけることで`ArticleListView`クラスでタグ名によるフィルタリングが行われます。

```Django:list-tag.html
{% extends 'memorandum/base-layout.html' %}

{% block content %}
  <ul class="tag-list">
    {% for item in object_list %}
      <a class="tag-link-text" href="{% url 'memorandum:index' %}?tag={{ item.name }}">
        <li class="tag-list-item">
          <div class="tag-list-text text-reddish">{{ item.name }}</div>
          <div class="tag-list-text background-reddish text-whitish small-circle">{{ item.post_count }}</div>
        </li>
      </a>
    {% endfor %}
  </ul>
{% endblock content %}
```

```Django:detail.html
{% extends 'memorandum/base-layout.html' %}

{% block content %}
    ---略---

    <h1 class="article-title">{{ object.title }}</h3>
    <ul class="article-tag-group">
      タグ:
      {% for tag in object.tag.all %}
        <a class="tag-link-text" href="{% url 'memorandum:index' %}?tag={{ tag }}">
          <li>
            <div class="article-tag">{{ tag }}</div>
          </li>
        </a>
      {% endfor %}
    </ul>

    ---略---
{% endblock content %}
```

これで、タグ一覧表示ページ、記事詳細表示ページ中のタグ名をクリックした場合にそのタグを含む記事一覧が表示されるようになりました。
下の画像では、`WSL`のタグをクリックしています（わかりにくいですが、`:hover`を使って`WSL`の表示が太文字になっています）。

![006.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/69aba1f4-f8aa-6188-a1bb-1a6f5b1705ed.png)

![007.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/c9f8e1f1-0270-40fd-2dab-23edc545e9f1.png)

### ◆新規タグ登録ページの画面遷移処理修正

つづいて、新規タグ登録完了時の画面遷移を改善していきます。
現状では、新規タグ追加画面でタグを追加した場合、ページが遷移してtopページに飛んでしまいます。
そこで、今回はDjango管理サイトのように、タグ追加のボタンを押すとポップアップ画面を表示し、タグを登録するとポップアップ画面を閉じる仕組みにしていきます。

ちなみに、このポップアップ画面の実装方法については、以下のサイトに詳しく書いてあります。私の記事より詳細に説明がありますんで、必要に応じてご覧ください。
_参考URL_：[Django、ポップアップでデータを追加](https://narito.ninja/blog/detail/62/)

#### □タグ追加用ポップアップ画面の実装（`common.js`追加）

まず、ボタンクリック時にタグ追加用ポップアップ画面を表示する処理を`JavaScript`で書いていきます。
なお、タグ追加のボタンは`index.html`と`create.html`の２か所あり、`common.js`ファイルを新規追加して、共通に使う関数を実装します。

```JavaScript:common.js
const addNewTagBtnElement = document.getElementById("js-create-tag-btn");

if(addNewTagBtnElement != null)
{
    addNewTagBtnElement.addEventListener("click", function() {
        console.log("hoge");
        window.open("/memorandum/tag/create/", "_blank", "width=640,height=480");
    }, false);
}
```

ID名が`js-create-tag-btn`のエレメントを取得し、その要素がクリックされた際に`window.open()`関数を呼び出してポップアップ画面を表示しています。
なお、`window.open()`関数では色々なオプションが指定できますが、ブラウザによってサポートされていなオプションもあるようなので、使用する際は注意が必要です。

_参考URL_：[window.open - MDN - Mozilla](https://developer.mozilla.org/ja/docs/Web/API/Window/open)
_参考URL_：[[JavaScript] 最新ブラウザではwindow.openのオプションは動かない](https://qiita.com/yun_bow/items/356f21fc376133037d84)

ちなみに、今回追加した`common.js`は複数のDjangoテンプレートから呼び出す予定のため、`js-create-tag-btn`IDが存在しない場合を考慮して、`if(addNewTagBtnElement != null)`でヌルチェックをしています。

#### □テンプレートファイルの修正（`index.html`/`create.html`）

つぎに、新規TAG追加ボタンが配置されている`index.html`/`create.html`に、さきほど定義した`js-create-tag-btn`IDを追加していきます。
また、その際に`a`タグから`button`タグに修正しています。

```Django:index.html
---略---

{% block content %}
  <div class="local-nav">
    <ul class="local-nav-list">
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create' %}">新規記事</a></li>
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:list_tag' %}">TAG一覧</a></li>
      {% comment %} <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create_tag' %}">新規TAG</a></li> {% endcomment %}
      <li class="local-nav-item"><button class="local-nav-link-text item-normal" id="js-create-tag-btn" type="button">新規TAG</button></li>
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

---略---
{% endblock content %}
```

```Django:create.html
---略---

          <div class="input-form-tag-group partition-bottom-line">
            <div class="input-form-position">
              <span class="input-label-text">{{ form.tag.label_tag }}</span>
              <div class="input-form-tag">{{ form.tag }}</div>
            </div>
            {% comment %} <a class="update-inner-link-text item-normal" href="{% url 'memorandum:create_tag' %}">Add New TAG</a> {% endcomment %}
            <button class="update-inner-link-text button-common" id="js-create-tag-btn" type="button">
            New TAG
            </button>
          </div>

---略---
```

#### □タグ追加後のポップアップ画面クローズ処理の追加（`views.py`/`urls.py`）

つづいて、ポップアップ画面でタグの登録が正しく完了した際に、そのポップアップ画面を閉じるようにしていきます。
`views.py`では、`TagCreateView`クラスの修正と新規関数（`close_window()`）の定義をします。

```Python:views.py
---略---

class TagCreateView(generic.edit.CreateView):
    model = Tag
    fields = ["name"]
    # 参照するhtmlファイルを指定
    template_name = "memorandum/create-tag.html"
    success_url = reverse_lazy('memorandum:close')  <- 修正部分

---略---

def close_window(request):
    return render(request=request, template_name="memorandum/close.html")
```

`success_url`に代入する値を修正し、タグ追加完了後に遷移するページを変更します。
（このあと後述する`urls.py`側の修正により、）`close_window()`関数が呼び出され、`memorandum/close.html`が返されます。

なお、`TagCreateView`クラス内で直接`memorandum/close.html`を返したい場合は、`form_valid()`関数をオーバーライドして`render()`関数を呼び出す必要がありそうです。
※`CreateView`クラスの`form_valid()`関数は`HttpResponseRedirect()`を呼び出しており、この関数は内部的に`reverse()`関数を呼び出しているため、URL直書きをするとエラーが発生する
_参考URL_：[はじめての Django アプリ作成、その 4](https://docs.djangoproject.com/ja/2.2/intro/tutorial04/)

`urls.py`ではさきほど`views.py`で追加した`close_window()`関数へのルーティングが正しくとおるようにpathを追加します。

```Python:urls.py
---略---

app_name = "memorandum"
urlpatterns = [
    path("", views.ArticleListView.as_view(), name="index"),
    path("create/", views.ArticleCreateView.as_view(), name="create"),
    path("<int:pk>/detail/", views.ArticleDetailView.as_view(), name="detail"),
    path("<int:pk>/update/", views.ArticleUpdateView.as_view(), name="update"),
    path("<int:pk>/delete/", views.ArticleDeleteView.as_view(), name="delete"),
    path("tag/create/", views.TagCreateView.as_view(), name="create_tag"),
    path("tag/list/", views.TagListView.as_view(), name="list_tag"),
    path("close/", views.close_window, name="close"),　<- 追記部分
]
```

#### □画面クローズ用ファイルの追加（`close.html`/`close.js`）

ここまでの実装で`close.html`への遷移が行われるようになったので、最後にウィンドウを閉じる処理と元ページをリロード（追加したタグを反映させるため）する処理を実装します。

```Django:close.html
{% extends 'memorandum/base-layout.html' %}

{% load static %}

{% block head %}
  <script src="{% static 'memorandum/js/close.js' %}" charset="UTF-8" defer></script>
{% endblock head %}

{% block content %}

{% endblock content %}
```

```JavaScript:close.js
// console.log("hugahuga");

window.opener.location.reload();
window.close();
```

ソースコード見たまんまですが、`close.html`は`close.js`を呼び出すだけです。
`close.js`では、新規タグ登録画面を呼び出した元の親画面をリロードした後、自画面を閉じています。

下のキャプチャ画面は、

+ 新規記事登録ページから「New TAG」ボタンを押して、新規タグ登録ページのポップアップ画面を表示した状態
+ その後、新規タグを登録した後にポップアップ画面がクローズされ、新規記事登録ページに追加したタグが反映されている状態  
の画面です。ここでは、`testd`というタグが追加されています。

![008.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/2f959ac2-ed10-f097-1859-c04ff9585ae1.png)

![009.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f38e88e5-69e3-92f3-557d-3ecdf56eb677.png)

# ◇おわりに

+ [前回の投稿](https://qiita.com/yankee/items/ce4e38346de880f7eabe)の記事検索機能の実装につづき、今回はタグの一覧表示や追加について実装することができました。
+ 備忘録を登録するだけの最低限のWEBアプリですが、これだけでもすでにCSSが煩雑になってきており、もう少しCSS用のクラス名の命名ルールをしっかり決めておく必要があるなーと感じてます。
+ それもあり、つぎはCSS部分の実装を少し改善（CSSフレームワークの導入 or SASSの導入）していければと思います（まずはドットインストールの「Sass/SCSS入門」を見るところからかな・・）。

# 🔚
