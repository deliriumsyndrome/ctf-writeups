# [Dreamhack] xss-1 풀이

## Flask

Flask : 파이썬으로 웹 서버를 만들게 해주는 라이브러리이다.

```python
@app.route("/")
def index():
    return "안녕"
```

`@app.route` : 해당 페이지에 접속 했을 때 아래의 코드를 실행하게 해주는 함수
위의 경우에서는 index 함수가 실행되며 화면에 '안녕' 이 뜨게 된다.

쿼리 스트링 : URL 뒤에 `?param=value` 형태로 값을 붙이면 서버에게 데이터를 넘기는 방법
여러 개를 넘기고 싶으면 `&`로 이어붙이면 된다. (`?a=1&b=2`)

`request.args.get("param")` : 쿼리 스트링을 통해 넘어온 값을 서버 코드에서 꺼낼 때 쓰는 함수

GET : 값을 주소(URL)에 실어서 보내는 방식 (request.args 로 받음)
POST : 값을 폼을 통해 제출 하는 방식으로 주소에 나타나지 않고 body에 담겨 전달된다. (request.form 로 받음)

## 코드 분석

### `/vuln` 라우트

```python
@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    return param
```

param = request.args.get("param", "")
위 함수는 param 이라는 이름으로 받은 값을 GET한다.
따라서 'param'에 '<script>alert(1)</script>'와 같은 스크립트를 작성하면
일반적인 글자가 아니라 실제로 실행해야 할 코드(스크립트)로 인식한다.
이렇게 사용자의 입력값이 그대로 실행 가능한 코드가
되어버리는 취약점을 XSS(Cross-Site Scripting)라고 한다.

### 봇을 움직이는 코드

이 문제에는 셀레니움 이라는 도구로 만들어진 봇이 존재한다.
이 봇은 사람이 아니라 코드가 자동으로 크롬 브라우저를 실행해
대신 접속해주는 역할을 한다.

```python
def read_url(url, cookie={"name": "name", "value": "value"}):
    ...
    driver = webdriver.Chrome(service=service, options=options)
    driver.get("http://127.0.0.1:8000/")   # 1. 먼저 메인 페이지에 접속한다
    driver.add_cookie(cookie)              # 2. 쿠키를 심는다
    driver.get(url)                        # 3. 지정된 주소로 이동한다
```

driver.get("http://127.0.0.1:8000/") :사이트에 한 번 접속하는 코드.
쿠키는 현재 접속해 있는 사이트에만 추가할 수 있기 때문

driver.add_cookie(cookie) : 봇의 브라우저에 쿠키 값을 심는 함수.

driver.get(url) : 지정된 주소로 봇을 이동시키는 함수.

쿠키 : 서버가 브라우저에게 "이 값을 기억해뒀다가 다음에 다시 보여달라"고 맡겨두는 작은 데이터.

### `/flag` 라우트

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

GET으로 접속하면 입력 폼 화면(flag.html)을 보여주고, POST로 제출하면 입력한 값을 param으로 받아 check_xss 함수에 넘긴다.
이때 전달하는 쿠키에는 더미 값이 아니라 실제 FLAG 값이 들어간다.
FLAG.strip() : flag를 출력 할 때 앞뒤 필요없는 공백 제거함.
history.go(-1) : 이전에 들어왔던 브라우저로 돌려보냄.

```python
def check_xss(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)
```

`urllib.parse.quote(param)` : 특수문자가 URL에서 깨지지 않도록 안전한 형태로 바꿔주는 함수.
위 함수는 제출된 값을 가지고 `/vuln?param=값` 주소를 만든 뒤, FLAG를
쿠키로 가진 봇이 그 주소를 방문하게 한다.

정리하면 다음과 같은 순서로 동작한다.
1. 내가 `/flag`에 스크립트를 적어서 제출.
2. 서버가 그 스크립트를 `/vuln?param=...` 주소 변환.
3. FLAG 쿠키를 가진 봇이 그 주소로 진입.
4. `/vuln`이 스크립트를 그대로 반환하고, 봇의 브라우저가 이를 실행.

이 시점에서 내가 작성한 스크립트는 FLAG 쿠키를 가진 브라우저 안에서
실행되는 상태가 된다. 따라서 스크립트 안에 document.cookie를 넣으면
FLAG 값을 읽을 수 있다.

`document.cookie` : 현재 브라우저가 가진 쿠키 전체를 문자열로
읽어오는 JavaScript 코드.

### `/memo` 라우트

```python
memo_text = ""

@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text)
```

memo_text : 서버가 실행되는 동안 계속 유지되는 전역 변수.
접속하는 사람과 상관없이 모두가 이 변수 하나를 공유한다.
위 함수는 쿼리 스트링으로 넘어온 memo 값을 memo_text에 누적시키고, 지금까지 쌓인 전체 내용을 화면에 출력한다.
따라서 내가 아무 때나 `/memo`에 접속하면 그동안 쌓인 내용을 전부 확인할 수 있다.

### FLAG를 바로 알려주지 않는 이유

`/flag`의 응답은 "good" 아니면 "wrong??" 뿐이며, FLAG 값 자체를
직접 알려주지는 않는다.
따라서 FLAG를 탈취해서 어딘가에 기록해두고, 내가 나중에 그 기록(쿠키)을 확인하는 별도의 경로가 필요하다.
이 경로의 역할을 하는 것이 `/memo`이다.

## 페이로드

```html
<script>location.href='/memo?memo='+document.cookie</script>
```

이 코드가 동작하는 순서는 다음과 같다.
document.cookie가 실행되어 현재 브라우저가 가진 쿠키 값이 문자열로 변환 (예: "flag=DH{...}").
이 문자열은 바로 주소로 전송되는 것이 아니라, 먼저 '/memo?memo='라는 문자열 뒤에 붙여짐.
그 결과 "/memo?memo=flag=DH{...}"라는 완성된 주소 문자열이 만들어진다.
location.href= : 브라우저에게 해당 주소로 이동하라고 명령, 사용자가 직접 주소창에 입력해서 접속하는 것과 동일한 기능을 한다.
이 명령이 실행되면 브라우저는 완성된 주소로 새로운 GET 요청을 서버에 전송, 서버의 `/memo` 라우트가 이 요청을 받아 memo_text에 값을 누적 및 출력함.

따라서 이 페이로드를 `/flag`에 제출하면 봇이 대신 실행해주며, 그 결과 `/memo` 페이지에 FLAG 값이 기록된다.
