---
title: WEBアプリの勉強を兼ねてDjangoで備忘録登録アプリを作ってみる
tags:
  - Python
  - Django
  - Webアプリケーション
  - 備忘録
  - 初投稿
private: false
updated_at: '2023-04-08T20:56:45+09:00'
id: 304f58273fb676f51f7a
organization_url_name: null
slide: false
---
# ◇この記事について
今回、初めての投稿となります（ホントは平成の間に初投稿といきたかったのですが、よくある日程遅延で令和までもつれ込みました・・）。
私自身プログラミング経験はあったんですが、WEBアプリ関係の知識がほとんどなかったため、知識・スキルを広げることを目的にこのアプリを作成してみました。
まずは、このアプリをベースにWEBアプリケーションに関する知識を少しずつ深めていければと思います。

# ◇記事投稿順（2019/06/29追記）
今回は、ベースのWEBアプリを1.の記事で作成し、2.以降の記事で機能の追加や改善を行っています。必要に応じてほかの記事も参照ください。
1. 【本記事】WEBアプリの勉強を兼ねてDjangoで備忘録登録アプリを作ってみる
2. [Djangoで作った備忘録登録WEBアプリの高機能化①（タイトル・タグ・本文検索機能の付加）](https://qiita.com/yankee/items/ce4e38346de880f7eabe)
3. [Djangoで作った備忘録登録WEBアプリの高機能化②（タグ一覧表示、タグ別記事の追加など）](https://qiita.com/yankee/items/4e387b8ec4c5a4053c89)


# ◇開発環境
+ OS : Ubuntu 18.04.2 LTS(Windows Subsystem for Linux)
+ 言語 : Python 3.6.7
+ Webアプリフレームワーク : Django (2.2)
+ DB : SQLite3

# ◇事前知識
+ Python : 勉強始めて１年程度
+ HTML/CSS/JavaScript : 勉強始めて２～３か月程度（ 主に[paizaラーニング](https://paiza.jp/works)、[ドットインストール](https://dotinstall.com/)を使って学習 ）
+ Django : Djangoドキュメントの[はじめての Django アプリ作成](https://docs.djangoproject.com/ja/2.2/intro/tutorial01/)をさらっと試した程度

# ◇実装内容
### ◆初期設定関係
Djangoのインストール手順は、[はじめての Django アプリ作成](https://docs.djangoproject.com/ja/2.2/intro/tutorial01/)に沿って進めていきました。細かい部分は省略しますが、以下のコマンドで、プロジェクトおよび備忘録用アプリを新規作成します。

```:Terminal
$ django-admin startproject mysite
$ python manage.py startapp memorandum
```

次に、自動作成されたファイルの中にある`settings.py`を修正します。具体的には、

+ `INSTALLED_APPS`内への`memorandum.apps.MemorandumConfig`の追加
+ `LANGUAGE_CODE`を`ja`に修正
+ `TIME_ZONE`を`Asia/Tokyo`に修正

を行っています。

```Python:mysite/mysite/settings.py

---略---

# Application definition

INSTALLED_APPS = [
    'memorandum.apps.MemorandumConfig',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

---略---

# Internationalization
# https://docs.djangoproject.com/en/2.1/topics/i18n/

# LANGUAGE_CODE = 'en-us'
LANGUAGE_CODE = 'ja'

# TIME_ZONE = 'UTC'
TIME_ZONE = 'Asia/Tokyo'

USE_I18N = True

USE_L10N = True

USE_TZ = True

---略---

```



> + `INSTALLED_APPS`内への`memorandum.apps.MemorandumConfig`の追加

ここでは、`mysite/memorandum/apps.py`内にある`class MemorandumConfig`をプロジェクトに登録しています。
コード中にもあるように、`name = 'memorandum'`としているため、この名前（memorandum）で登録しても問題なさそう（実際、そうやっている説明サイトもある）でしたが、今回は参考にしたドキュメントに倣ってクラス名自体を記述しています。

```Python:mysite/memorandum/apps.py

from django.apps import AppConfig


class MemorandumConfig(AppConfig):
    name = 'memorandum'

```

> + `TIME_ZONE`を`Asia/Tokyo`に修正

については、最初は修正をしてなかったんですが、一通りアプリができた後の検証作業でデータの追加日時がずれるという問題が発生したため、原因を調査して追加で修正しました。
この修正を行うことによって、日時が日本時刻で表示されるようになります。
※`USE_TZ = True`をしておくと、内部的に保存されるデータはUTCで保存される仕様のようです。

```
USE_TZ = True
TIME_ZONE = 'Asia/Tokyo'
```

この設定の組み合わせで、

+ 内部で保存されるデータはUTC
+ 表示される日時データは日本時刻に自動変換される

という形になります。

_参考URL_：[Djangoで、タイムゾーンの変換](https://narito.ninja/blog/detail/82/)


### ◆ファイル構成

`tree`コマンド実行時のターミナル出力結果（`__pycache__`下のファイルなどは削ってます）を下に載せます。自分で明示的に追加・修正した部分については、`<-`でコメントを追記してます。

```:Terminal
$ tree mysite/
mysite/
├── db.sqlite3
├── manage.py
├── memorandum
│   ├── __init__.py
│   ├── __pycache__
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   ├── 0001_initial.py
│   │   ├── __init__.py
│   │   └── __pycache__
│   ├── models.py                    <-データベース内容定義等
│   ├── static
│   │   └── memorandum
│   │       └── css
│   │           └── memorandum.css   <-スタイルシート用ファイル
│   ├── templates
│   │   └── memorandum
│   │       ├── base-layout.html     <-レイアウト用ファイル
│   │       ├── create-tag.html      <-新規タグ作成ページ
│   │       ├── create.html          <-新規記事作成ページ
│   │       ├── delete.html          <-記事削除ページ
│   │       ├── detail.html          <-記事詳細表示ページ
│   │       ├── index.html           <-トップページ（記事一覧表示）
│   │       └── update.html          <-記事内容修正ページ
│   ├── tests.py
│   ├── urls.py                      <-memorandum内のルーティング定義等
│   └── views.py                     <-各関数のメイン処理定義等
└── mysite
    ├── __init__.py
    ├── __pycache__
    ├── settings.py                  <-言語設定、タイムゾーン設定等の定義（前述のとおり）
    ├── urls.py                      <-mysiteプロジェクト全体のルーティング定義等
    └── wsgi.py

```

### ◆データベースのモデル定義（models.py）

モデルの定義として、`Article`クラスと`Tag`クラスの２つに分けて作成しました。
`Tag`クラスを分けたのは、一つの記事に複数のタグが登録できるようにしたいという考えからです。
`Tag`クラスでは、タグ名（`name`）のみを保存し、
メインのデータベースとなる`Article`クラスでは、

+ タイトル（`title`）
+ タグ名（`tag`）
+ 本文（`body`）
+ 作成日（`published_date`）
+ 更新日（`modified_date`）

を定義し、タグ名（`tag`）については、`Tag`クラスを`ManyToManyField`を付けて参照しています。
ManyToManyField を付けることによって、多対多の関係で紐づけが可能になります。

_参考URL_：[Django2.0 　ManyToMany（多対多）リレーションを使う](http://www.goodmadeused.com/entry/django_manytomany)

また、作成日（`published_date`）は`auto_now_add=True`、更新日（`modified_date`）は`auto_now=True`のオプションを付けています。

+ `auto_now_add=True`：データが新規作成される際に、その時の日付が自動で設定されます。
+ `auto_now=True`：データが新規作成または__更新__される際に、その時の日付が自動で設定されます。

これらのオプションを付けておくと、新規記事作成ページなどで作成日（`published_date`）と更新日（`modified_date`）のフォームを追加せずに自動で値が設定されます。

_参考URL_：[モデルフィールドリファレンス](https://docs.djangoproject.com/ja/2.2/ref/models/fields/#datefield)

```Python:models.py
from django.db import models

# Create your models here.
class Tag(models.Model):
    name = models.CharField(max_length=128, unique=True)

    def __str__(self):
        return self.name

class Article(models.Model):
    title = models.CharField(max_length=255)
    tag = models.ManyToManyField(Tag)
    body = models.TextField()
    published_date = models.DateField(auto_now_add=True)
    modified_date = models.DateField(auto_now=True)
    
    #これ入れないと、リスト一覧でタイトル名が表示されない
    def __str__(self):
        return self.title
```

`models.py`の修正が終わったら、以下コマンドを実行し、修正したモデル定義をプロジェクトに反映させます。

```:Terminal
$ python manage.py makemigrations
$ python manage.py migrate 
```

### ◆URLルーティングの定義（urls.py）

まず、`mysite`下の`urls.py`で、全体のURLルーティングを設定します。

```Python:mysite/urls.py
"""mysite URL Configuration

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/2.1/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  path('', views.home, name='home')
Class-based views
    1. Add an import:  from other_app.views import Home
    2. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
Including another URLconf
    1. Import the include() function: from django.urls import include, path
    2. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
"""
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('memorandum/', include('memorandum.urls')),
    path('admin/', admin.site.urls),
]
```

`urlpatterns`に`path('memorandum/', include('memorandum.urls')),`を追加することにより、WEBアプリの`memorandum/`下にアクセスが来た場合、`memorandum`下の`urls.py`のルールが適用されるようになります。

次に、`memorandum/`のURLルーティングを設定します。

```Python:memorandum/urls.py
from django.urls import path

from . import views

app_name = "memorandum"
urlpatterns = [
    # path("", views.index, name="index"),
    path("", views.ArticleListView.as_view(), name="index"),
    path("create/", views.ArticleCreateView.as_view(), name="create"),
    path("<int:pk>/detail/", views.ArticleDetailView.as_view(), name="detail"),
    path("<int:pk>/update/", views.ArticleUpdateView.as_view(), name="update"),
    path("<int:pk>/delete/", views.ArticleDeleteView.as_view(), name="delete"),
    path("tag/create/", views.TagCreateView.as_view(), name="create_tag"),
]
```

この記載をすることにより、例えば、
`memorandum/create/`にアクセスがあった場合、`views.py`内の`ArticleCreateView.as_view()`関数が呼び出されるといった動作になります。
`name="create"`で名前を付けておくことによって、HTML内のhrefなどでその名前を使ってリンクさせることが可能になります。

### ◆ビュー関数の定義（views.py）

views.pyで、それぞれのページアクセス時にどのデータを表示するか記述していきます。
ちなみに、今回はDjangoでサポートしているクラスベースビューというものを使用しています。
※このクラスベースビュー機能は、ListViewやCreateViewなど汎用的なビュークラスについてあらかじめDjnagoで定義されており、これらのクラスを使うことにより、実装作業を効率化できるというものです。

_参考URL_：[Djangoにおけるクラスベース汎用ビューの入門と使い方サンプル](https://qiita.com/felyce/items/7d0187485cad4418c073)

```Python:views.py
from django.shortcuts import render,redirect
from django.urls import reverse_lazy
from django.http import HttpResponse
from django.views import generic

from .models import Tag, Article 

class ArticleListView(generic.ListView):
    model = Article

    #参照するhtmlファイルを指定
    template_name = "memorandum/index.html"

class ArticleDetailView(generic.DetailView):
    model = Article
    #参照するhtmlファイルを指定
    template_name = "memorandum/detail.html"

class ArticleCreateView(generic.edit.CreateView):
    model = Article
    fields = ["title","tag","body"]
    #参照するhtmlファイルを指定
    template_name = "memorandum/create.html"
    success_url = reverse_lazy('memorandum:index')  #POSTが正しく行われた際に飛ばすURL

class ArticleUpdateView(generic.edit.UpdateView):
    model = Article
    fields = ["title","tag","body"]
    #参照するhtmlファイルを指定
    template_name = "memorandum/update.html"
    success_url = reverse_lazy('memorandum:index')  #POSTが正しく行われた際に飛ばすURL

class ArticleDeleteView(generic.edit.DeleteView):
    model = Article
    #参照するhtmlファイルを指定
    template_name = "memorandum/delete.html"
    success_url = reverse_lazy('memorandum:index')  #POSTが正しく行われた際に飛ばすURL

class TagCreateView(generic.edit.CreateView):
    model = Tag
    fields = ["name"]
    #参照するhtmlファイルを指定
    template_name = "memorandum/create-tag.html"
    success_url = reverse_lazy('memorandum:index')  #POSTが正しく行われた際に飛ばすURL
```

それぞれのクラスの最初の`model = Article`or`model = Tag`で使用するモデル定義を選択しており、最低限、これだけでリスト等の表示が行えますが、必要に応じてデフォルト設定から書き換えたい部分については、書き換えていく形になります。

+ `template_name`はページアクセス時に返すhtmlファイルの__ローカル側__のパスになります（デフォルトでは`クラス名_list.html`のような名前のhtmlファイルを探してしまうようなので、基本的に明示的に設定してあげた方がいいかと思います）。
+ `success_url`は`UpdateView`や`CreateView`の際に使用されるもので、保存実行時に正常に終了した場合に遷移させるページのURLになるようです。
+ `reverse_lazy`は、`'memorandum:index'`などで定義したルーティング情報をURLに変換する関数のようです（完全に理解できてない・・）。<br>
参考サイトを読んだ限りでは、`success_url`で設定する場合、`reverse`関数ではなく、`reverse_lazy`関数を使う必要があるとのことだったので、それに倣っています。

_参考URL_：[[Django] success_urlとget_success_urlおよびreverseとreverse_lazyの使い分け](https://btj0.com/%E3%83%96%E3%83%AD%E3%82%B0/django/django-success_url%E3%81%A8get_success_url%E3%81%8A%E3%82%88%E3%81%B3reverse%E3%81%A8reverse_lazy%E3%81%AE%E4%BD%BF%E3%81%84%E5%88%86%E3%81%91/)

### ◆テンプレート（HTMLファイル）の作成

最後に、HTMLファイルとCSSファイルの作成を行います。全てのHTMLページを載せると記事が冗長になるため、いくつかかいつまんで載せていきます。

#### □`base-layout.html`の作成

ページタイトル、ロゴ画面については、共通テンプレートを使用して記述しています。

```Django:base-layout.html
{% load static %}
<!DOCTYPE html>
<html lang="ja">
  <head>
    <link rel="stylesheet" href="{% static 'memorandum/css/memorandum.css' %}">
    <title>{% block title %}Memo Site{% endblock title %}</title>
    {% block head %}{% endblock head %}
  </head>

  <body>
    <header>
      <div class="top-title">
        <a class="home-link-text" href="{% url 'memorandum:index' %}">備忘録登録サイト</a>
      </div>
      {% block header %}
      {% endblock header %}
    </header>
    <div class="main-content">
      {% block content %}
      {% endblock content %}
    </div>
  </body>
</html>
```

#### □トップページ（記事一覧表示）`index.html`の作成
`object_list`内にモデル定義したデータが入っているため、forループで中のアイテムを一つづつ取り出し、そのデータを表示しています。
それぞれの要素については、`models.py`で定義した変数名（titleなど）を`{{ item.title }}`という形で指定すればそのデータが表示されます。

```Django:index.html
{% extends 'memorandum/base-layout.html' %}

{% block content %}
  <div class="local-nav">
    <ul class="local-nav-list">
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create' %}">新規記事</a></li>
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:create_tag' %}">新規TAG</a></li>
    </ul>
  </div>
  <ul class="article-list">
    {% for item in object_list %}
      <li class="article-list-item">
        <a class="link-text" href="{% url 'memorandum:detail' item.pk %}">{{ item.title }}</a>
        <div class="article-date">作成日: {{ item.published_date|date:"Y年m月d日(D)" }}<br>更新日: {{ item.modified_date|date:"Y年m月d日(D)" }}</div>
      </li>
    {% endfor %}
  </ul>
{% endblock content %}
```

![index.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/d9e7f040-49f0-935d-0f10-3c6cc25d2988.png)

#### □記事詳細表示ページ`detail.html`の作成

基本的には、`index.html`同様、`{{ object.title }}`といった形で表示したいデータを指定します。
今回は、__一つ__の記事の詳細を表示する形なのでforループは不要です。
また、`{{ object.body | linebreaks }}`のように、`linebreaks`を入れることにより、フォームで入力した改行コードがHTML表示に反映されます。

_参考URL_：[Django: 組み込みタグとフィルタの一覧](https://qiita.com/nachashin/items/d3f9cd637a9cecbda72c)


```Django:detail.html
{% extends 'memorandum/base-layout.html' %}

{% block content %}
  <div class="local-nav">
    <ul class="local-nav-list">
      <li class="local-nav-item"><a class="local-nav-link-text item-normal" href="{% url 'memorandum:update' object.pk %}">編集</a></li>
      <li class="local-nav-item"><a class="local-nav-link-text item-caution" href="{% url 'memorandum:delete' object.pk %}">削除</a></li>
    </ul>
  </div>
  <div class="main-article">
    <h1 class="article-title">{{ object.title }}</h3>
    <ul class="article-tag-group">
      タグ:
      {% for tag in object.tag.all %}
        <li>
          <div class="article-tag">{{ tag }}</div>
        </li>
      {% endfor %}
    </ul>
    <div class="article-date">
      <span class="">作成日: {{ object.published_date|date:"Y年m月d日(D)" }}</span>
      <span class="">更新日: {{ object.modified_date|date:"Y年m月d日(D)" }}</span>
    </div>
    <p class="article-body">{{ object.body | linebreaks }}</p>
  </div>
{% endblock content %}
```

![detail.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4f185aa7-d94d-c394-c88f-66aea34085d1.png)

#### □新規記事作成ページ`create.html`の作成

`create.html`では、フォームを使用してデータの追加をしていきます。
`{{ form.title }}`という形でタイトル用の入力フォームを表示できます。
また、`{{ form.title.label_tag }}`というようにlabel_tag を付けると、その要素の名前が表示されます。

```Django:create.html
{% extends 'memorandum/base-layout.html' %}

{% block content %}
  <div class="main-article">
    <form method="post">{% csrf_token %}
        <div class="input-form">
          <div class="input-form-position partition-bottom-line">
            <span class="input-label-text">{{ form.title.label_tag }}</span>
            <div class="input-form-title">{{ form.title }}</div>
          </div>
          <div class="input-form-tag-group partition-bottom-line">
            <div class="input-form-position">
              <span class="input-label-text">{{ form.tag.label_tag }}</span>
              <div class="input-form-tag">{{ form.tag }}</div>
            </div>
            <a class="update-inner-link-text item-normal" href="{% url 'memorandum:create_tag' %}">Add New TAG</a>
          </div>
          <div class="input-form-position partition-bottom-line">
            <span class="input-label-text">{{ form.body.label_tag }}</span>
            <div class="input-form-body">{{ form.body }}</div>
          </div>
        </div>
        <!-- {{ form.as_p }} -->
        <input class="button-common" type="submit" value="Save">
    </form>
  </div>
{% endblock content %}
```

![create.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/6d2d52f8-970e-041f-f7ad-8759ac1c12f7.png)


update、deleteについては今回は割愛します。
ここまでの作業で備忘録の新規追加、一覧表示、詳細表示、記事修正、削除の機能をもつWEBアプリを作成することができました。
※TAGについては、現状createしかできず、TAGを削除する場合はadminサイトから行う必要があるため、今後直していければと思います


# ◇その他、参考にした記事
+ [Djangoの 汎用クラスビューをまとめて、実装について言及する](https://qiita.com/renjikari/items/af3e8958d2653e6f8d46)
+ [[Django]NoReverseMatch ‘hoge’ is not a valid view function or pattern name.とエラーが出た時の確認事項](https://code-schools.com/django-noreversematch/)

# ◇今後追加したい機能
+ タイトル検索、タグ検索機能の実装（現状、タグが役に立っていない・・）。
-> [こちら](https://qiita.com/yankee/items/ce4e38346de880f7eabe)に追加記事を記載しました。（2019/06/02追記）

+ タグの管理メニュー（現状追加しかできない・・）。
+ 新規タグ追加画面からタグを追加した場合にtopページに飛んでしまうため、その点の修正。
-> [こちら](https://qiita.com/yankee/items/4e387b8ec4c5a4053c89)に追加記事を記載しました。（2019/06/29追記）
+ フォームの改善（forms.pyの作成）。
+ CSSフレームワーク（Bootstrapなど）の導入（今回のアプリレベルでcssファイルが雑多になってきたため・・）<br>
⇒フレームワークどーせ入れるならSass（SCSS or SASS）も試してみたい。

# ◇最後に
+ 今回、初めての投稿となったわけですが、記事作成に想定よりも時間がかかるというのを改めて痛感しました（結局、GW最終日の夜中まで掛かった）。。
+ 私よりわかりやすい記事を書いている方々の凄さを実感するとともに、そんな記事を載せてくれる人達のありがたみがを再認識できるGWとなりました。
+ Djangoについては、Qiita内でも多くの記事が既に出ていますので、そちらの方が参考になるかとは思いますが、少しでも参考になる方がいれば幸いです。
