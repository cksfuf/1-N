# 1:N 관계 설정

### 파일설정
- n-1 폴더 만들고 code 열기
- `python -m venv venv`
- `source venv/Scripts/activate`
- `pip install django`
- .gitignore 파일 만들고 gitignore.io 붙여넣기
- `git init`

- 장고프로젝트만들기
`django-admin startproject board .`

- app 만들기
`django-admin startapp articles`

- app 등록하기
`board > settings.py` 의 33번 줄 `INSTALLED_APPS` 에 `'articles',` 등록

- 서버 실행 `python manage.py runserver`

- 최상단에 `templates` 만들고 안에 `base.html` 만들기
    - ` base.html` 파일 기본구조 잡아주고 바디에 블럭 잡아주기.
        ```html
        <body>
            {% block body %}
            {% endblock %}
        </body>
        ```

- 최상단에 만든 `templates`를 장고가 인식할 수 있게 만들기
`board > settings.py` 의 55번줄` TEMPLATES` 안에 `'DIRS': [BASE_DIR / 'templates']` 코드 수정

- `class article(models.Model)`과 `class Comment(models.Model)` 모델링 구조잡기
    ```python
    class article(models.Model):
        title = models.CharField(max_length=100)
        content = models.TextField()
    ```
- 마이그레이션
    - sql 번역작업
        ```bash
        python manage.py makemigrations
        ```

    - sql 적용
        ```bash
        python manage.py migrate
        ```

- `admin`에 `Article` 모델 등록
    - admin.py에 모델 불러오고 지정
        ```python
        from .models import Article

        admin.site.register(Article)
        ```

- 관리자 아이디 만들기
    ```bash
    python manage.py createsuperuser
    ```

## 오늘은 댓글 달아보기
- 댓글을 달기위한 앱을 만들어야하나? 댓글과 게시물은 1:n 관계.
- 댓글 기능은 articles 안에 만들거임. articles 가 있어야 댓글도 존재하는 관계 이기에.

### 1. Read(All)기능구현
- `articles/templates/index.html` 생성
    ```html
    {% extends 'base.html' %}

    {% block body %}
        {% for article in articles %}
            <p>{{article.title}}</p>
            <hr>
        {% endfor %}
    {% endblock %}
    ```
- `articles/urls.py` 생성
    ```python
    from django.urls import path
    from . import views

    app_name = 'articles'

    urlpatterns = [
        path('', views.index, name='index'),
    ]
    ```
- `articles/views.py` 코드 추가
    ```python
    from django.shortcuts import render
    from .models import Article # 추가

    def index(request): # 추가
        articles = Article.objects.all()

        context = {
            'articles': articles,
        }

        return render(request, 'index.html', context)
    ```
- `board/urls.py` 코드 추가
    ```python
    from django.contrib import admin
    from django.urls import path, include # include 추가

    urlpatterns = [
        path('admin/', admin.site.urls),
        path('articles/', include('articles.urls')), # 추가
    ]
    ```

### 2. Read(1) 기능구현
- `articles/templates/detail.html` 생성
    ```html
    {% extends 'base.html' %}

    {% block body %}
        <h1>{{article.title}}</h1>
        <p>{{article.content}}</p>

    {% endblock %}
    ```
- `articles/templates/index.html` 코드 추가
    ```html
    {% block body %}
        {% for article in articles %}
            <p>{{article.title}}</p>
            <a href="{% url 'articles:detail' id=article.id %}">detail</a> <!-- 추가 -->
            <hr>
        {% endfor %}
    {% endblock %}
    ```
- `articles/urls.py` path 추가
    ```python
    urlpatterns = [
        path('', views.index, name='index'),
        path('<int:id>/', views.detail, name='detail'), # 추가
    ]
    ```
- `articles/views.py` 함수 추가
    ```python
    def detail(request, id):
        article = Article.objects.get(id=id)

        context = {
            'article': article,
        }

        return render(request, 'detail.html', context)
    ```

### 3. Create 기능구현
- `articles/forms.py` 생성
    ```python
    from django import forms
    from .models import Article

    class ArticleForm(forms.ModelForm):
        class Meta():
            model = Article
            fields = '__all__'
    ```
- `articles/templates/form.html` 생성
    ```html
    {% extends 'base.html' %}

    {% block body %}
        <form action="" method="POST">
            {% csrf_token %}
            {{form}}
            <input type="submit">
        </form>
    {% endblock %}
    ```
- `articles/urls.py` path 추가
```python
urlpatterns = [
    ...
    path('create/', views.create, name='create'), # 추가
]
```
- `articles/views.py` 함수 추가
    ```python
    from django.shortcuts import render, redirect # redirect 추가
    from .models import Article
    from .forms import ArticleForm # 추가

    def create(request):
        if request.method == 'POST':
            form = ArticleForm(request.POST)
            if form.is_valid():
                article = form.save()
                return redirect('articles:detail', id=article.id)
        else:
            form = ArticleForm()

        context = {
            'form': form,
        }

        return render(request, 'form.html', context)
    ```
- `templates/base.html` 추가
```html
<body>
    <a href="{% url 'articles:index' %}">home</a>
    <a href="{% url 'articles:create' %}">create</a>
    {% block body %}
    {% endblock %}
</body>
```

### 4. Comment 모델링 / 마이그레이션
- `articles/models.py` Comment Class 추가
    ```python
    ...

    class Comment(models.Model):
        content = models.TextField()
        article = models.ForeignKey(Article, on_delete=models.CASCADE)
    ```
- 클래스 지정 후 다시 마이그레이션
    ```bash
    python manage.py migrate
    ```

### 5. admin에 Comment모델 추가
- `articles/admin.py` 코드 추가
    ```python
    from django.contrib import admin
    # from .models import Article
    from .models import Article, Comment

    admin.site.register(Article)
    admin.site.register(Comment)
    ```

### 6. Comment Read 기능구현
- `articles/templates/detail.html` 코드 추가
    ```html
    {% block body %}
        <h1>{{article.title}}</h1>
        <p>{{article.content}}</p>


        <hr>
        {% for comment in article.comment_set.all %}
            <li>{{comment.content}}</li>
        {% endfor %}
    {% endblock %}
    ```

### 7. Comment Create 기능구현
- `articles/forms.py` CommentForm 클래스 추가
    ```python
    from .models import Article, Comment

    class CommentForm(forms.ModelForm):
        class Meta():
            model = Comment
            # fields = '__all__'

            # fields => 추가할 필드 이름 목록
            # fields = ('content', )

            # exclude => 제거할 필드 이름 목록
            exclude = ('article', )
    ```
- `articles/templates/detail.html` `<form>`코드 추가
    ```html
        <p>{{article.content}}</p>

        <hr>
        <form action="{% url 'articles:comment_create' article_id=article.id %}" method="POST">
            {% csrf_token%}
            {{form}}
            <input type="submit">
        </form>
        {% for comment in article.comment_set.all %}
            <li>{{comment.content}}</li>
        {% endfor %}
    ```
- `articles/urls.py` path 추가
    ```python
        ...

        path('<int:article_id>/comments/create/', views.comment_create, name='comment_create'),
    ]
    ```
- `articles/views.py` 함수 추가
    ```python
    from .forms import ArticleForm, CommentForm

    def detail(request, id):
        article = Article.objects.get(id=id)
        form = CommentForm() # 추가

        context = {
            'article': article,
            'form': form, # 추가
        }
        return render(request, 'detail.html', context)

    ...

    def comment_create(request, article_id):
        if request.method == 'POST':
            form = CommentForm(request.POST)
            if form.is_valid():
                comment = form.save(commit=False)

                # 1. 객체를 저장하는 방법
                # article = Article.objects.get(id=article_id)
                # comment.article = article
                # comment.save()

                # 2. integer(숫자)를 저장하는 방법
                comment.article_id = article_id
                comment.save()

                return redirect('articles:detail', id=article_id)

        else:
            return redirect('articles:index')
    ```

### 8. Comment Delete 기능구현
- `articles/templates/detail.html` 코드 추가
    - Delete 버튼 생성
```html
    {% for comment in article.comment_set.all %}
        <li>{{comment.content}}</li>
        <form action="{% url 'articles:comment_delete' article_id=article.id id=comment.id%}" method="POST">
            {% csrf_token %}
            <input type="submit" value="delete">
        </form>
    {% endfor %} 
```
- `articles/urls.py` Delete 버튼를 만들기 위한 경로 추가
```python
urlpatterns = [
    ...

    path('<int:article_id>/comments/<int:id>/delete/', views.comment_delete, name='comment_delete'),
]
```
- `articles/views.py` Delete 버튼을 활성화 시켜주기 위한 함수 작성
```python
def comment_delete(request, article_id, id):
    if request.method == 'POST':
        comment = Comment.objects.get(id=id)
        comment.delete()

    return redirect('articles:detail', id=article_id)
```