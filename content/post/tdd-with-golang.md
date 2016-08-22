+++
date = "2016-08-20T12:15:44+09:00"
draft = false
title = "Golang 프로젝트에 TDD 도입하기"
tags = ["Development", "UnitTest", "TDD"]
categories = ["Development", "UnitTest","TDD"]
series = ["Go Web Dev"]

author = "Yun Sang Bae"
+++
여기에서 사용한 테스트 코드는 [Bitbucket](https://bitbucket.org/dream_yun/handlertest) 에서 다운로드 할 수 있다.
## TDD 
클라우드와 **MSA**와 **REST**의 등장으로 TDD가 재조명 받고 있다. TDD를 제대로 적용하려면 상당히 많은 시간과 노력이 필요하다. 특히 여기 저기 연동되는 라이브러리나 소프트웨어가 많은 경우 테스트가 굉장히 복잡해지는데, 복잡해지는 만큼 테스트의 신뢰성도 함께 떨어진다. 

TDD는 **유닛 테스트**를 기본으로 하는데, 애플리케이션이 복잡해지면 유닛테스트에 간섭하는 객체들이 많아진다. 이렇게 늘어난 객체들에 대해서 테스트를 진행하다 보면 테스트를 위한 설계로 변질되는 경우가 있다. 

데이터베이스, 소켓, UI가 서로 엉켜있는 소프트웨어를 테스트 한다고 생각해보라. 머리 좀 아플 것이다. 물론 TDD가 테스트를 쉽게 할 수 있는 설계를 지향하긴 하지만, 테스트를 쉽게 할 수 있는 설계와 테스트를 위한 설계는 엄연히 다른 것이다.

TDD의 단점은 아래와 같이 정리 할 수 있다.

  1. **개발 기간이 늘어난다.** TDD에 익숙해졌다고 가정 할 경우 약 20% 정도 구현시간이 늘어난다. 복잡한 소프트웨어의 경우 더 구현시간은 더 늘어날 것이다.  
  2. **복잡성 증가.** 테스트시나리오가 길어질 경우, 시나리오 자체를 관리하는 것도 작업이 된다.
  3. **디자인 변경.** 종종 TDD에 어울리지 않는 디자인의 소프트웨어를 개발 해야 하는 경우도 있다. TDD는 **좋은 코드는 테스트하기 좋은 코드다**라고 주장한다. 하지만 항상 그런건 아니다. TDD에 맞추다 보니 디자인이 이상해지는 경우가 종종 생긴다.

요약하자면 실행관점에서 TDD를 위한 기본 요소는 **유닛 테스트**인데, 소프트웨어가 복잡해지면 굉장히 힘들어 지는게 TDD의 문제다.  

달리 생각하면 소프트웨어가 단순해지면 TDD를 하기 좋은 환경이 된다는 이야기가 되겠다.**MSA와 REST** 바로 그런 환경이다. 

MSA는 작업 서비스(애플리케이션)들을 결합해서 하나의 큰 서비스를 만드는 서비스 디자인 스타일이다. 각 MSA 서비스들은 다른 서비스들과 독립적으로 구성되고 단순한 기능을 가지도록 설계되기 때문에 유닛 테스트가 큰 효과를 발휘 할 수 있다. 

Go언어는 범용 시스템언어로 개발이 됐지만 **net/http**와 **gorilla**를 비롯해서 MSA+REST(이하 MSA) 스타일의 웹 애플리케이션을 효과적으로 만들 수 있도록 지원하고 있다. 나는 Go 언어에서 MSA 애플리케이션을 TDD로 개발하고 테스트 하는 방법을 정리 하려 한다. 이 문서에서 다룰 내용은 아래와 같다. 

  1. Go 언어에서 제공하는 유닛테스트 프레임워크를 살펴본다.
  2. HTTP 핸들러 테스트 : HTTP 웹 서버 핸들러를 테스트하려면, 서버가 실행 중이어야 하기 때문에 메서드 단위의 유닛 테스트로는 테스트가 어렵다. **net/http/httptest**패키지를 이용해서 HTTP 핸들러를 테스트할 수 있다.
  3. TDD는 유닛테스만 의미하지 않는다. 개발 에서 배포까지의 전 과정을 **테스트**를 기반으로 통합하는 일련의 과정들이다. 젠킨스(Jenkins)를 이용해서 TDD를 완성해 본다.
  4. **테스트 커버리지는** 유닛테스트가 얼마나 잘 이루어졌는지를 측정하기 위해서 사용한다. 테스트 커버리지를 계산하고 그 결과를 문서로 출력한다. 이 문서를 젠킨스와 통합해보자.

## Go 유닛 테스트 개요
Go 는 테스트 프레임워크를 내장(build-in)하고 있다. **testing**페키지를 이용해서 유닛 테스트 코드를 만들고 **go test**명령으로 테스트를 수행하면 된다. 유닛 테스트를 위한 간단한 예제 코드를 만들었다. 코드의 이름은 math.go 다.
```go
package math

import (
    "errors"
)   

// 값들을 모두 더한다.
func Sum(nums ...int) int {
    total := 0 
    for _, num := range nums {
        total += num
    }
    return total
}   

// a를 b로 나눈다.
func Div(a float64, b float64) (float64, error) {
    if b == 0 {
        return 0.0, errors.New("Can't divide by zero")
    }
    return a / b, nil
}

// 문자열을 count만큼 반복하고 결과를 반화한다.
func StrRept(s string, count int) string {
    b := make([]byte, len(s)*count)
    bp := copy(b, s)
    for bp < len(b) {
        copy(b[bp:], b[bp:])
        bp *= 2
    }
    return string(b)
}   
```

유닛 테스트 파일을 만든다. 파일의 이름은 **math_test.go**다. 참고로 유닛 테스트 파일의 이름은 반드시 **_test.go**로 끝나야 한다.
```go
package math

import (
    "testing"
)

func Test_Sum(t *testing.T) {
    v0 := Sum(1, 2, 3)
    if v0 != 6 {
        t.Fatal("1+2+3 == 6")
    }

    v1 := Sum(6, 5)
    if v1 != 11 {
        t.Fatal("6+5 == 11 ")
    }
}

func Test_Div(t *testing.T) {
    v2, _ := Div(0, 2)
    t.Log("0/2 =",v2)
}

func Test_StrRept(t *testing.T) {
    str := StrRept("a", 3)
    if len(str) != 3 {
        t.Fatal("Repeat fail")
    }
}
```
go test를 실행해보자.
```sh
# go test
PASS
ok  	_/home/yundream/workspace/golang/unitTest	0.001s
```

**-v** 옵션을 주면 자세한 테스트 정보를 확인 할 수 있다. 로그(t.Log) 정보도 함께 출력한다.
```sh
# go test -v
=== RUN   Test_Sum
--- PASS: Test_Sum (0.00s)
=== RUN   Test_Div
--- PASS: Test_Div (0.00s)
	math_test.go:21: 0/2 = 0
=== RUN   Test_StrRept
--- PASS: Test_StrRept (0.00s)
PASS
ok  	_/home/yundream/workspace/golang/unittest	0.002s
```

**Test_StrRept**테스트를 아래와 같이 수정 한 다음에 테스트해보자.
```go
func Test_StrRept(t *testing.T) {
    str := StrRept("ab", 3)
    if len(str) != 3 {
        t.Fatal("Repeat fail")
    }
}
```
테스트 조건을 바꿨는데, 실수로 예상 결과를 수정하지 않았다.

```go
# go test -v
=== RUN   Test_Sum
--- PASS: Test_Sum (0.00s)
=== RUN   Test_Div
--- PASS: Test_Div (0.00s)
	math_test.go:21: 0/2 = 0
=== RUN   Test_StrRept
--- FAIL: Test_StrRept (0.00s)
	math_test.go:27: Repeat fail
FAIL
exit status 1
FAIL	_/home/yundream/workspace/golang/unittest	0.002s
```
테스트 실패를 확인 할 수 있다.

## testing 패키지
함수가 실행 된 결과가 예측한 결과와 맞아 떨어지는 지를 검사하는 방식으로 테스트를 진행 한다. **t.Fatal()**, **t.Fail()**등을 이용해서 테스트를 제어 할 수 있다. 

  * **FailNow()**이 호출되면, 테스트 함수를 즉시 종료하고 다음 테스트 함수를 실행한다.
  * **Fatal()**는 로그를 출력하는 걸 제외하고 **FailNow** 메서드와 같은 일을 한다. 
  * **Fail()**이 호출되면, 테스트가 실패하더라도 함수를 종료하지 않고 다음 코드를 계속 실행한다. 
  * **Error()**는 로그를 출력하는 걸 제외하고 **Fail** 메서드와 같은 일을 한다.
  * **Errorf()** 형식화된 로그를 출력한다. Fila 메서드와 같은 일을 한다.
  * **Log()** 테스트 로그를 출력한다. 
  * **Logf()** 형식화된 테스트 로그를 출력한다.
  * **Failed()** 실패하더라도 레포트하지 않는다.

## Assertion
테스트 코드를 만들다 보면 if 문이 코드의 절반 이상을 차지하는 걸 보게될 것이다. 비교대상도 가지각색이라서 가독성이 떨어진다. assert 함수가 필요하다. 직접 만들어 보고 싶겠지만 그냥 잘 만들어져 있는 테스트 패키지 가져다가 쓰자. 내가 요즘 쓰고 있는 테스트 패키지는 **github.com/stretchr/testify/assert**이다. 패키지를 설치 한 후 아래 코드를 테스트 했다. 
```
package yours

import (
    "github.com/stretchr/testify/assert"
    "testing"
)

func TestSomething(t *testing.T) {
    // assert equality
    assert.Equal(t, 123, 125, "they should be equal")

    // assert inequality
    assert.NotEqual(t, 123, 456, "they should not be equal")
}
}}}
테스트를 돌려보자.
{{{#!plain
# go test
--- FAIL: TestSomething (0.00s)
        Error Trace:    yours_test.go:11
	Error:		Not equal: 123 (expected)
			        != 125 (actual)
	Messages:	they should be equal
		
FAIL
exit status 1
FAIL	_/home/yundream/workspace/golang/mytest	0.003s
```
테스트 코드와 테스트 결과의 가독성 모두 좋아졌다. 이 패키지는 **assert**외에도 **mock**, **http 테스트**, **suite**등 테스트를 위한 다양한 툴들을 지원한다. 

## HTTP 핸들러 테스트
HTTP 핸들러의 경우 웹 서버를 띄워야 하기 때문에, 메서드보다 테스트가 까다롭다. 아래의 방식으로 테스트 할 수 있다. 

  1. **net/http/httptest** 패키지를 이용한 테스트. httptest를 이용하면 루프백(127.0.0.1)에 바인드 되는 서버를 띄울 수 있다. 이후 net/http에서 제공하는 클라이언트 메서드들을 이용하면 **서버 & 클라이언트**모드에서 테스트 할 수 있다.
  2. 아예 빌드하고 실행하고, HTTP 클라이언트를 이용해서 테스트 한다.
각각의 장/단점이 있다. 1의 경우 테스트 커버리지를 확인 할 수 있고, 2의 경우에는 통합된 환경에서의 테스트가 가능하다. 나는 1과 2의 방법을 모두 다 사용하고 있다. 여기에서는 **httptest**를 이용한 테스트를 살펴볼 생각이다.

테스트에 사용한 소스코드 트리다.
```sh
.
├── handler
│   ├── handler.go
│   └── handler_test.go
└── main.go
```

net/http, gorilla, 패키지를 이용해서 개발 했다. GET /ping에 대해서 "pong"를 리턴하는 일을 한다.
```go
package handler

import (
    "encoding/json"
    "fmt"
    "github.com/gorilla/mux"
    "net/http"
)

type Handler struct {
    router *mux.Router
}

func (h Handler) Init() {
    h.router = mux.NewRouter()
    h.router.HandleFunc("/ping", h.Ping).Methods("GET")
    http.Handle("/", h.router)
}
func (h Handler) Ping(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "pong")
}
func (h Handler) Calc(w http.ResponseWriter, r *http.Request) {

}
```

테스트 코드다. 
```go
// handler_test.go
package handler

import (
    "github.com/gorilla/handlers"
    "github.com/stretchr/testify/assert"
    "io/ioutil"
    "net/http"
    "net/http/httptest"
    "os"
    "testing"
)

var (
    server  *httptest.Server
    testUrl string
)

type Response struct {
    Content string
    Code    int
}

func Test_Init(t *testing.T) {
    logfile, err := os.OpenFile("/tmp/test.log", os.O_RDWR|os.O_CREATE|os.O_APPEND, 0755)
    assert.Nil(t, err, "")
    h := Handler{}
    h.Init()
    server = httptest.NewServer(handlers.CombinedLoggingHandler(logfile, http.DefaultServeMux))
    testUrl = server.URL
}

func Test_Ping(t *testing.T) {
    res, err := DoGet(testUrl + "/ping")
    assert.Nil(t, err, "")
    assert.Equal(t, 200, res.Code, "PING API")
    assert.Equal(t, "pong", res.Content, "PONG Message")
}

func DoGet(url string) (*Response, error) {
    response, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer response.Body.Close()
    contents, err := ioutil.ReadAll(response.Body)
    if err != nil {
        return nil, err
    }
    return &Response{Content: string(contents), Code: response.StatusCode}, nil
}
```
**httptest**패키지는 테스트를 위해서 내장된 웹 서버를 실행한다. 따라서 핸들러 등록, 데이터베이스 연결과 같이 서비스를 위해서 필요한 자원들을 초기화 해야 한다. **Test_init**메서드를 이용해서 서비스를 초기화 하고 있다. 테스트 코드에 대한 디버깅은 **testing.T.Log** 계열의 메서드를 이용해서 모니터에 표준출력하는 방식으로 진행하는데, 웹 서버가 실행되는 방식이라서 로그를 표준출력 할 수 없다. 그래서 /tmp/test.log에 access log를 남기기로 했다. 

**httptest.NewServer** 메서드를 실행하면, 웹 서버가 실행된다. 웹 서버의 접근 URL은 server.URL에 저장돼 있다.

**Test_Ping**에서 ping API를 테스트한다. 테스트는 http client를 이용한다. 200 OK와 "pong" 메시지를 검사하고 있다. 테스트결과다.
```go
# go test -v
=== RUN   Test_Init
--- PASS: Test_Init (0.00s)
=== RUN   Test_Ping
--- PASS: Test_Ping (0.00s)
PASS
ok  	bitbucket.org/dream_yun/handlertest/handler	0.003s
```
존재하지 않는 페이지를 요청 할 경우 **404 Page Not Found**를 반환해야 할테다. 이를 테스트하기 위한 코드를 만들었다. 
```go
func Test_APINotFound(t *testing.T) {
    res, err := DoGet(testUrl + "/myfunc")
    assert.Nil(t, err, "")
    assert.Equal(t, 404, res.Code, "Unknown API")
}
```
이렇게 테스트 시나리오에 따라서 테스트 코드를 추가하면 된다.

## 서비스간 연동 테스트
MSA 모델을 따르는 애플리케이션을 만들다 보면, 다른 (REST)애플리케이션과 통신 해야 할 수도 있다.

![MSA에서의 REST API를 이용한 서비스 이용](https://docs.google.com/drawings/d/1VaRfN4dntpTqQb_zOZhXMXDxVa49PzXfIOR9KyPO_Co/pub?w=521&h=255) 

**App02**는 서비스에서 발생한 다양한 데이터들을 관리하는 일을 한다. 유저가 업로드한 이미지, 문서 파일은 App-02로 전달된다. App-02는 이 파일들을 유저 설정에 따라서 S3, DropBox, Google Drive 등으로 전송한다. 

나는 App01 서비스도 유저가 입력한 연산과 그 결과를 **App02**를 이용해서 저장하기로 했다. -- 이게 어떤 쓸모가 있는 기능인지는 묻지도 말고 따지지도 말자 -- **코드의 추가와 추가된 코드에 대한 테스트**가 필요하다. 

App02를 직접 띄운다음 테스트 하는 방법도 있다. 이 방법에 따라 테스트 하려면 App02를 단순 실행하는 것이 아닌, App02가 제대로 실행 할 수 있는 환경을 만들어야 한다. 그러니까 S3, DropBox, Google Drive 등과 연동할 수 있는 환경을 **개발 서버에 만들어야** 한다. 애로 사항이 꽃필 것이다. 최종 연동 테스트에서는 이렇게 해야겠지만, 개발단계에서 이렇게 하기는 쉽지 않다. 

나는 입력과 출력만 검사하는 **블랙 박스 테스트**를 실행하기로 했다. httptest 패키지를 이용해서 **App02 테스트 서버**를 만들었다. 물론 App02 테스트 서버를 만들기 위해서는 App02의 API 명세서와 App02 패키지가 필요하다. 아래는 테스트 서버 코드다. handler 디렉토리 밑에 만들었다.
```go
//  handler/app02_test_server.go
package handler

import (
    "bitbucket.org/dream_yun/app02"
    "fmt"
    "github.com/gorilla/handlers"
    "github.com/gorilla/mux"
    "net/http"
    "net/http/httptest"
    "os"
)

type TestApiServer struct {
    router *mux.Router
}

// 실행 후 테스트 서버의 URL을 반환한다.
func (api *TestApiServer) Run() string {

    api.router = mux.NewRouter()
    api.router.HandleFunc("/save/{serviceName}", api.Save).Methods("POST")
    api.router.HandleFunc("/save/{serviceName}/{fileName}", api.ReadFile).Methods("GET")

    logfile, _ := os.OpenFile("/tmp/app02_test.log", os.O_RDWR|os.O_CREATE|os.O_APPEND, 0644)
    server := httptest.NewServer(handlers.CombinedLoggingHandler(logfile, api.router))
    return server.URL
}

// Save API다. 여기에 여러가지 테스트 조건들을 코딩하면 된다. 
func (api TestApiServer) Save(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    serviceName := vars["serviceName"]
    if serviceName != "calc" {
        w.WriteHeader(http.StatusBadRequest)
        return
    }
    fmt.Fprintf(w, app02.ServiceOK)
}

// 저장된 파일을 가져온다.
func (api TestApiServer) ReadFile(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    serviceName := vars["serviceName"]
    fileName := vars["fileName"]
    if serviceName != "calc" {
        w.WriteHeader(http.StatusBadRequest)
        return
    }

    if fileName == "my.jpg" {
        fmt.Fprintf(w, app02.ServiceOK)
        return
    }
}
```

연산을 끝낸 후에 Save API를 호출하도록 Div 메서드를 수정했다.
```go
type Handler struct {
    router     *mux.Router
    fileServer string
}

func (h Handler) Init(fileServer string) {
    h.router = mux.NewRouter()
    h.fileServer = fileServer
    h.router.HandleFunc("/ping", h.Ping).Methods("GET")
    h.router.HandleFunc("/div/{a}/{b}", h.Div).Methods("GET")
    http.Handle("/", h.router)
}

func (h Handler) Div(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    a := vars["a"]
    b := vars["b"]

    ai, err := strconv.Atoi(a)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }
    bi, err := strconv.Atoi(b)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }

    if bi == 0 {
        w.WriteHeader(http.StatusNotAcceptable)
        return
    }
    if ai == 0 { 
        w.WriteHeader(http.StatusNotAcceptable)
        return
    }
    
    DoPost(h.fileServer+"/save/calc", "a/b")
    fmt.Fprintf(w, "%d", ai/bi)
}
```
  * Handler 구조체에 fileServer 변수를 추가했다. 여기에는 app02 서버의 주소가 저장된다.
  * 매개변수로 app02 서버를 받도록 Handler.Init() 메서드를 수정했다.

Test 코드도 수정했다.
```go
func Test_Init(t *testing.T) {
    logfile, err := os.OpenFile("/tmp/test.log", os.O_RDWR|os.O_CREATE|os.O_APPEND, 0755)
    assert.Nil(t, err, "")

    FileServer := TestApiServer{}
    fileServerAddr := FileServer.Run()
    h := Handler{}
    h.Init(fileServerAddr)
    server = httptest.NewServer(handlers.CombinedLoggingHandler(logfile, http.DefaultServeMux))
    testUrl = server.URL
}
```
TestApiServer의 바인드 주소를 읽어서 Handler.Init() 메서드에 넘기도록 테스트 코드를 수정했다. 이제 테스트를 실행하면 TestAPIServer가 실행되고, Div 메서드가 TestAPIServer의 save api를 호출하는 것을 볼 수 있을 거다. 

이 테스트는 완전하지 않다. Div의 DoPost 호출 부분을 충분히 테스트하지 않았기 때문이다. 테스트 커버리지를 높이려면, DoPost를 호출하는 별도의 메서드를 만들어서 메서드의 입/출력을 테스트 할 수 있도록 해야 한다.  

여기에서 중요한 점은 **테스트를 쉽게 하기 위해서 메서드들을 수정**했다는 점이다. TDD에서는 코드에 맞는 테스트를 하는게 아니고, 테스트에 맞는 코드를 만든다.

## 목업 vs 직접 구성
예제로 삼았던 ping API 서버는 외부 소프트웨어의 도움 없이 작동한다. 하지만 현실에서 이런 코드를 찾기는 어렵다. 마이에스큐엘(Mysql), 몽고디비(Mongodb), 주키퍼(zookeeper), 레디스(Redis) 등 수많은 다른 애플리케이션들과 통신을 한다. 어떻게 테스트 해야 할까.

연동 애플리케이션과 서버를 모두 구축해서 테스트 하는 방법이 있다. 마이에스큐엘, 몽고디비, 레디스.. 등등을 모두 설치해서 테스트 하는 거다. 이 방법의 단점은 상당히 귀찮다는 것이다. 혼자 하는 개발하는 하고 있다면 좀 귀찮아도 해볼만 하지만, 여럿이 개발한다면 애로사항이 꽃필 것이다. 이외에도 데이터베이스 오류 상황에서, 소프트웨어가 어떻게 작동할지를 테스트하기가 쉽지 않다는 것도 문제다. 

이 문제는 **mocks/stubs**으로 모의 객체를 만들어서 테스트하는 것으로 테스트 커버리지는 늘리면서도 테스트 시간을 줄일 수 있다.

결론부터 말하자면 난 목업을 이용하지 않고 있다. 작동하는 소프트웨어들과 직접 연동해서 테스트 한다. 개발/테스트 환경 구축의 번거로움은.. 글쎄 나는 (데이터베이스를 설치하고 설정하는) 정도의 번거로움은 감수해야 하고, 감수한 만큼 개발자에게 이득이 있다고 생각하는 입장이다. 그리고 세상이 좋아졌다. VM, Container, Vagrant 등을 이용하면 개발환경을 손쉽게 구성하고 배포, 공유 할 수 있다.

## 테스트 커버리지
테스트에 대한 품질은 테스트 커버리지로 측정 할 수 있다.
```sh
# go test -cover
PASS
coverage: 100.0% of statements
ok  	bitbucket.org/dream_yun/handlertest/handler	0.004s
```

모든 코드를 완전히 테스트 하고 있다. 이 예제로는 테스트 커버리지를 확인하기가 애매모호해서, API를 추가했다. 
```go
// package handler
func (h Handler) Div(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(request)
    a := vars["a"]
    b := vars["b"]

    if b == 0 {
        w.WriteHeader(http.StatusNotAcceptable)
        return
    }
    if a == 0 {
        w.WriteHeader(http.StatusNotAcceptable)
        return
    }
    ia, err := strconv.Atoi(a)
    ib, err := strconv.Atoi(b)
    fmt.Fprint(w)
}
```
지금은 코드를 먼저 만들었지만 TDD의 원칙을 정확히 따르려면, 테스트 코드를 먼저 만들고 나서 코드를 만들어야 할 것이다. 다만 MSA의 경우에는 API단위로 하는 일이 특정되기 때문에, 코드를 먼저 만들고 테스트 코드를 만드는 것도 괜찮은 방법이라고 생각한다. **TDD를 위한 TDD**가 문제다라는 주장이 나오는 이유를 생각해보자. 완전한 방법, 완전한 툴은 없다. 자신의 역량과 환경에 적절하게 응용해서 사용해야 한다.  

유닛 테스트를 돌려보자. 테스트 커버리지가 떨어진 걸 확인 할 수 있을 것이다.
```sh
# go test -cover
PASS
coverage: 21.7% of statements
ok  	bitbucket.org/dream_yun/handlertest/handler	0.004s
```

Div 메서드에 대한 테스트 코드를 만들어서 커버리지를 올리기로 했다.
```go
func Test_Div(t *testing.T) {
    res, err := DoGet(testUrl + "/div/4/2")
    assert.Nil(t, err, "")
    assert.Equal(t, http.StatusOK, res.Code)
    assert.Equal(t, "2", res.Content)
    
    res, err = DoGet(testUrl + "/div/4/a")
    assert.Nil(t, err, "")
    assert.Equal(t, http.StatusInternalServerError, res.Code, "Invalide argument")
    
    res, err = DoGet(testUrl + "/div/0/4")
    assert.Nil(t, err, "")
    assert.Equal(t, http.StatusNotAcceptable, res.Code, "Invalide argument")
    
    res, err = DoGet(testUrl + "/div/4/0")
    assert.Nil(t, err, "")
    assert.Equal(t, http.StatusNotAcceptable, res.Code, "Invalide argument")
}   
```go
```sh
# go test -cover
PASS
coverage: 91.3% of statements
ok  	bitbucket.org/dream_yun/handlertest/handler	0.005s
}}}
```
테스트 커버리지는 "이 소프트웨어는 적어도 이 정도의 코드 영역에 대해서 테스트 하고 있다"는 것을 알려준다. 특히 코드에 대한 리펙토링이나 기능 추가시 필요한 품질을 측정하는데 매우 좋은 자료가 된다. 예를 들어 현재 릴리즈된 소프트웨어의 커버리지가 90%일 경우, 수정된 코드의 커버리지를 90%로 맞춘다면 적어도 이전에 테스트했던 내용들은 모두 테스트 했으며, 이전 수준에서의 품질을 유지 하고 있다고 예상 할 수 있을 것 이다.

스타트업의 경우 서비스의 품질보다 출시 시기가 중요한 경우가 많다. 이 경우 소위 **기술부채**라는 명목으로 품질을 희생하는 경우가 많은데,  나중에 기술부채를 제거하기 위해서 엄청난 시간과 노력을 투입해야 할 수 있다. 개발 환경을 갖추지 못한 상태에서 급하게 기술부채를 제거 할 경우 곤욕을 치를 수 있고, 서비스의 발목을 잡을 수도 있다. 

테스트 코드는 리펙토링과 설계변경을 쉽게 할 수 있도록 도와준다. 테스트 커버리지를 관리하는 것으로 일관성 있는 품질을 달성 할 수 있다. 유닛 테스트를 이용해서 기술 부채를 관리 할 수 있다.

테스트 커버리지의 목표를 90%로 잡았다고 가정해 보자. 91.3 % 이니 이 정도면 충분하다고 생각 할 수 있겠으나 그렇지 않다. **서비스의 보안수준은 가장 약한 보안 고리에 의해서 결정된다.** 서비스 품질 역시 마찬가지로 가장 약한 고리가 서비스의 전체 품질을 결정 한다. 따라서 테스트 하지 않은 부분이 서비스에 중요한 영향을 미칠 수 있는 지 점검 해야 한다. 위의 정보로는 어느 부분이 테스트가 안됐는지를 확인 할 수 없다.

테스트 커버리지 레포트를 만들어 보자.
```go
# go test -coverprofile=coverage.out 
# go tool cover -html=coverage.out
```
**-coverprofile** 옵션을 이용하면 테스트한 코드 영역에 대한 레포트가 만들어진다. 일반 텍스트 파일인데, **tool cover** 명령을 이용해서 html 파일로 변환 할 수 있다. 

![go cover html 화면](https://ryudmq.by3302.livefilestore.com/y3mJYLhYTZWyMa4UE1wfN3H-D-RKFA6M7rnil_bUWKMDXdP86xNzJ77MkFXFGN32oS68Pjw5yDOVed1st1toTzjOtip0WaCuWJhEhT5pN8B7b3s3YVrUT9T2ixkHFWHBpM8IgHKykDGGRg5bo_7KA5p-8Ju9AGn4AX4vE0T8xv3vME?width=660&height=432&cropmode=none)

**strconv.Atoi()** 메서드에 대한 테스트가 빠져있음을 알 수 있다. 이에 대해서  "Div API는 클라이언트가 숫자(0-9)가 아닌 다른 값을 보낼 수도 있으므로, 에러 체크가 필요하다. 그리고 변수 a, b에 대한 타당성(숫자인지, int64 범위의 값인지 등)을 검사하는 코드도 추가해야 한다"라는 평가를 할 수 있을 것이다. 

## gocov
go에서 제공하는 기본 툴도 쓸만하긴 하지만, 레포팅 기능이 썩 맘에 들지 않는다. 그래서 **gocov**라는 툴을 이용해서 레포트를 만들기로 했다.
```sh
# go get github.com/axw/gocov/gocov
# go get github.com/matm/gocov-html
```

**gocov test**를 이용해서 커버리지 데이터파일을 만든다.
```sh
# gocov test ./ > handler.json
ok  	bitbucket.org/dream_yun/handlertest/handler	0.005s	coverage: 91.3% of statements
```

handler.json을 html 파일로 변환한다.
```sh
# gocov-html handler.json > handler.html
```

브라우저로 읽어보자. 

![gocov 이미지 1](https://ryscvq.by3302.livefilestore.com/y3miXS8j97x5BezCReIZYH2BNOnlXESm33jhcUAzj_wBD2JzF_k12xyAOv0aKwUj1EhFHkdVZ4bDYFR9Jgvk3TxacSU-W-TU0Xg2MUm1C_i4F-rtt0OqVQr2aDwtRToqAXndvtyN8u6aQHEN8DWVFneg32NSTz71iH8RhcDyMmmqZU?width=660&height=477&cropmode=none)

![gocov 이미지 2](https://rytnsa.by3302.livefilestore.com/y3mdq3P4WAWJGSzJnwg1NuQ0Ck4zue-ttulK2zZ2fPppUVuAe-AMqOLrWK4YVUXoFbQ_rQxH3kL1LEJgSFH8MZP68u57aqPCnxJ3AY9AATwCjtCHHBDc_fV3n6BgAZy_SdNCC57KNiavbNIMchRgDbi0_CbpZYrCMso5989qPFUJSU?width=660&height=477&cropmode=none)

훨씬 보기 좋아졌다. 
## 젠킨스와의 통합
이미 젠킨스를 통해서 테스트를 자동화 하고 있다. 여기에 레포트만 추가하면 된다. 

젠킨스에 웹 서버를 설치하고 gocov test, gocov-html 과정에서 나온 html 결과물을 웹 서버 디렉토리에 저장해서 레포팅 하는 방법도 있다. 하지만 레포팅 결과물이 젠킨스 대시보드와 분리된다는 점이 썩 맘에 들지 않는다. 그리고 gocov-html은 **현재 상태**만 보여준다는 문제가 있다. 테스트 결과를 평가 하기 위해서는 이전 테스트 결과도 함께 볼 수 있어야 한다. 그래서 젠킨스의 코드 테스트 커버리지 레포팅 플러그인인 [Cobertura](http://cobertura.github.io/cobertura)를 사용하기로 했다. Cobertura는 자바코드의 커버리지를 측정하기 위해서 만들어진 툴이지만 XML 포멧만 맞춘다면 다른 언어에도 문제없이 사용 할 수 있다. 

gocov의 결과를 xml로 출력하기 위해서 gocov-xml을 설치했다.
```sh
# go get github.com/AlekSi/gocov-xml
```

아래와 같이 테스트커버리지 결과를 xml 문서로 출력할 수 있다.
```sh
# gocov test ./ | gocov-xml > coverage.xml
```

젠킨스에 cobertura 플러그인을 설치하는 과정은 [Go언어와 Jenkins](http://www.joinc.co.kr/w/man/12/jenkins)문서를 참고하자. 아래는 적용 결과다.

![coberatura 레포팅](http://marcelog.github.io/articles/cobertura_example_covertool.png)
	
## TDD 통합 프로세스
MSA 모델을 따르는 소프트웨어의 개발에서 배포 단계까지의 테스트 방식을 정리해 보자.

![TDD 통합 프로세스](https://docs.google.com/drawings/d/18iqVYcMa8_en8Ro2OjwhnTIujD7cdPCPwxtoNNtNHy0/pub?w=882&h=469)

개발 단계에서는 **화이트 박스 테스트**와 **블랙 박스 테스트**를 함께 사용한다. 직접 제어하고 테스트 할 수 있는 코드들은 화이트 박스 테스트의 대상이다. 애플리케이션을 구성하고 있는 핸들러와 핸들러에서 호출하는 메서드들이다.  

다른 애플리케이션과 (REST API로) 연결된 코드의 경우에는 블랙 박스 테스트를 진행한다. 해당 애플리케이션 개발자로 부터 API 규격을 받아서, 입력과 출력을 테스트 하는 방식이다. **httptesting**을 이용해서 블랙 박스 테스트를 위한 웹 서버를 띄우면 된다. 입력과 출력의 사양은 연동 애플리케이션의 패키지를 그대로 사용할테니, 문서의 내용과 코드가 맞지 않는다고 해도 문제될게 없다. 그냥 패키지를 참고해서 개발해도 된다. 연동 애플리케이션의 규격이 변경될 경우 테스트에러가 떨어질테니 개발단계에서 문제를 해결 할 수 있게 된다.

통합 단계에서는 연동 테스트까지 진행한다. 최신 버전의 애플리케이션을 실행 하고, 직접 API를 전송해서 테스트 하는 방식이다. 
