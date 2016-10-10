+++
title = "Go에서 애플리케이션 설계하기"
tags = [
  "번역", "아키텍쳐",
]
categories = [
  "번역", "아키텍처",
]
authors = [
  "mingrammer",
]
draft = false
toc = false
date = "2016-10-10T21:11:16+09:00"

+++

> [Structuring Applications in Go](https://medium.com/@benbjohnson/structuring-applications-in-go-3b04be4ff091#.rannpkamk)을 번역한 글입니다.

## 개요

Go를 배울 때 가장 어려웠던 부분은 애플리케이션을 어떻게 설계하는가였다. Go 이전에, 나는 Rails 애플리케이션을 만들었었는데 Rails는 애플리케이션을 특정한 방식으로 설계하도록 한다. "설정보다는 컨벤션"이라는게 그들의 모토였다. 그러나 Go는 그 어떤 프로젝트 구조나 애플리케이션 설계방식을 규정짓고 있지 않으며 Go의 컨벤션은 대개 제각각이다.

나는  Go 애플리케이션의 아키텍쳐를 구성하면서 발견한 정말 많은 도움이 되었던 4가지 패턴을 여러분에게 알려주려고한다. 이들은 공식적인 규칙은 아니며 누군가는 다른 의견을 가질 수 있다고 생각한다. 나는 그런 의견들을 듣고싶다! 제안할만한게 있다면 댓글로 달아줬으면 좋겠다.

## 1. 전역 변수를 사용하지 말라

내가 읽었던 Go의 net/http 예시들은 다음과 같이 항상 함수들을 [http.HandleFunc](https://golang.org/pkg/net/http/#HandleFunc)를 사용하여 등록한다.

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/hello", hello)
    http.ListenAndServe(":8080", nil)
}

func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hi!")
}
```

이 예제는 net/http를 사용하기 위한 쉬운 방법을 제공하지만 이는 나쁜 습관을 가르친다. 함수 핸들러를 사용하게되면, 애플리케이션 상태에 접근하는 유일한 방법은 전역 변수를 사용하는 것이다. 이 때문에, 우리는 전역 데이터베이스 커넥션 또는 전역 설정 변수를 추가할수도 있다. 그러나  유닛 테스트를 작성할 때 이 글로벌 변수들을 사용한다는건 악몽이다.

더 나은 방법은 핸들러에 대해 특정한 타입을 만들어 필요한 변수들을 가질 수 있게 만드는 것이다.

```go
type HelloHandler struct {
    db *sql.DB
}

func (h *HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    var name string
    
    // 쿼리 실행.
    row := h.db.QueryRow("SELECT myname FROM mytable")
    if err := row.Scan(&name); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    // 클라이언트에 전송.
    fmt.Fprintf(w, "hi %s\n", name)
}
```

이제 우리는 데이터베이스를 초기화할 수 있으며 글로벌 변수 없이 핸들러를 등록할 수 있다.

```go
func main() {
    // 데이터베이스 커넥션을 연다.
    db, err := sql.Open("postgres", "...")
    if err != nil {
        log.Fatal(err)
    }
    
    // 핸들러 등록.
    http.Handle("/hello", &HelloHandler{db: db}
    http.ListenAndServe(":8080", nil)
}
```
이 접근법은 또한 핸들러 유닛 테스팅을 자체적으로 할 수 있다는 이점을 가지며 심지어 HTTP 서버도 필요하지않다.

```go
func TestHelloHandler_ServeHTTP(t *testing.T) {
    // 커넥션을 열고 핸들러를 설정한다.
    db, _ := sql.Open("postgres", "...")
    defer db.Close()
    h := HelloHandler{db: db}
    
    // 간단한 버퍼를 가지고 핸들러를 실행.
    rec := httptest.NewRecorder()
    rec.Body = bytes.NewBuffer()
    
}
```

***UPDATE:*** Tomás Senart와 Peter Bourgon의 [트위터 멘션](https://twitter.com/tsenart/status/485920561391239168)에서 언급된 [클로저를 사용한 핸들러 래핑](https://gist.github.com/tsenart/5fc18c659814c078378d)으로 이를 좀 더 간단히 할 수 있다. 이는 핸들러를 쉽게 구성할 수 있게 해준다.

<br>

## 2. 애플리케이션에서 바이너리를 분리하라

나는 누군가 "go get"을 실행하면 내 애플리케이션이 자동으로 설치될 수 있도록 *main.go* 파일을 프로젝트 루트에 넣어 사용한다. 그러나, *main.go* 파일과 애플리케이션 로직을 같은 패키지로 합치게되면 다음의 두 가지 결과를 갖게된다.

1. 애플리케이션을 라이브러리로써 재사용할 수가 없다.
2. 오직 하나의 애플리케이션 바이너리만을 가질 수 있다.

이 문제를 해결하기위해 발견한 가장 좋은 방법은 단순히 프로젝트내에서 각 서브 디렉토리가 하나의 애플리케이션 바이너리인 *"cmd"* 디렉토리를 사용하는 것이다. 나는 이 접근법을 여러 개의 애플리케이션 바이너리를 사용하는 Brad Fitzpatrick의 [Camilstore](https://camlistore.org/) 프로젝트에서 처음으로 발견했다.

```
camlistore/
    cmd/
        camget/
            main.go
        cammount/
            main.go
        camput/
            main.go
        camtool/
            main.go
```

Camilstore가 설치될 때 빌드되는 4개의 분리된 애플리케이션 바이너리(camget, cammount, camput, camtool)가 있다.
  
### 라이브러리 주도 개발 (Library driven development)
 
 *main.go* 파일을 루트 밖으로 옮기는 것은 애플리케이션을 라이브러리의 관점에서 구현할 수 있게 해준다. 애플리케이션 바이너리는 단순히 애플리케이션 라이브러리의 클라이언트이다. 나는 이것이 나의 핵심 로직 코드(라이브러리)가 무엇이고 애플리케이션 실행 코드(애플리케이션 바이너리)가 무엇인지에 대한 추상화를 명확히 하도록 도와준다는걸 알 수 있다. 

애플리케이션 바이너리는 정말 단순히 사용자가 로직과 상호작용하는 방식에 대한 인터페이스이다. 때때로 당신은 유저가 여러 방법으로 상호작용할 수 있도록 여러개의 바이너리를 생성한다. 예를 들어, 두 수를 더하는 "adder"라는 패키지가 있을 때, 커맨드 라인 버전뿐만 아니라 웹 버전을 배포하고 싶을 수도 있다. 프로젝트를 다음과 같이 구성하면 이를 쉽게 만들 수 있다.

```
adder/
    adder.go
    cmd/
        adder/
            main.go
        adder-server/
            main.go
```

유저는 생략 부호("...")를 사용하여 "go get"으로 "adder" 애플리케이션 바이너리를 설치할 수 있다.

```
$ go get github.com/benbjohnson/adder/...
```
짜잔, 사용자는 설치된 *"adder"* 와 *"adder-server"* 를 갖게되었다!

<br>

## 3. 애플리케이션별 컨텍스트를 위한 타입 래핑

내가 발견한 특히 유용한 한 트릭은 애플리케이션 수준의 컨텍스트를 제공하기위해 몇가지 제너릭 타입을 래핑하는 것이다. 한 가지 훌륭한 예는 DB와 Tx (transaction) 타입을 래핑하는것이다. 이 타입들은 database/sql 패키지나 [Bolt](https://github.com/boltdb/bolt)같은 다른 데이터베이스 라이브러리에서 찾을  수 있다.

우리는 이 타입들을 다음과 같이 래핑하면서 시작할 수 있다:

```go
package myapp

import (
    "database/sql"
)

type DB struct {
    *sql.DB
}

type Tx struct {
    *sql.Tx
}
```
이제 우리의 데이터베이스와 트랜젝션을 위해 초기화 함수를 래핑하자:

```go
// Open은 데이터 소스를 위한 DB 레퍼런스를 반환한다.
func Open(dataSourceName string) (*DB, error) {
    db, err := sql.Open("postgres", dataSourceName)
    if err != nil {
        return nil, err
    }
    return &DB{db}, nil
}

// Begin은 새로운 트랜젝션을 반환하기 시작한다.
func (db *DB) Begin() (*Tx, error) {
    tx, err := db.DB.Begin()
    if err != nil {
        reutnr nil, err
    }
    return &Tx{tx}, nil
}
```

예를 들어, 만약  사용자가 생성하기 전에 다른 시스템에 대한 검증이 필요하다거나 다른 테이블들의 업데이트가 필요할 때 이 함수는 더 복잡한 일을 할 수 있다.

### 트랜잭션 구성

이 함수들을 *Tx* 에 추가하는 것의 또 다른 이점은 하나의 트랜잭션에서 여러개의 액션을 구성할 수있다는 것이다. 사용자를 추가해야 하는가? 그냥 *Tx.CreateUser()* 를 한 번 호출하면 된다:

```go
tx, _ := db.Begin()
tx.CreateUser(&User{Name:"susy"})
tx.Commit()
```
기본 데이터 스토어를 추상화 하는것은 새 데이터베이스로 교환하거나 다수의 데이터베이스의 사용을 쉽게 만들어준다. 그들은 애플리케이션의 *DB & Tx* 타입을 호출하는 코드로부터 모두 감춰져있다.

<br>

## 4. 서브패키지로 골머리를 앓지 말라

대다수의 언어는 패키지 구조를 원하는대로 구성할 수 있도록 한다. 나는 모든 클래스들이 다른 패키지에  채워지고 이 패키지들은 서로를  모두 포함하고 있는 Java 코드베이스를 가지고 근무했던적이 있다. 정말 엉망이었다!

Go는 패키지를 위한 요구조건이 딱 한가지 있는데, 순환 의존(cyclic dependencies)을 가질 수 없다는 것이다. 처음엔 이 순환 의존 규칙이 조금 이상하게 느껴졌다. 나는 원래 프로젝트를 각 파일은 하나의 타입을 가지고 패키지에는 파일들이 여러개 있도록 구성하며 새로운 서브패키지를 만들려고 했었다. 그러나 이 서브패키지들은 패키지 "A"를 포함하는 패키지 "C"를 포함하는 패키지 "B"를 포함하는 패키지 "A"를 찾을 수 없었기에 점점 관리하기가 어려워졌다. 이는 순환 의존이 될 것이다. 나는 "너무 많은 파일들"을 갖게 되는것에 대해서는 제외하고 패키지들을 분리시킬만한 이유가 없다는 걸 깨달았다. 

최근 나는 오직 하나의 루트 패키지를 사용하는 방법을 택했다. 보통 내 프로젝트의 타입들은 모두 매우 연관이 많기 때문에 이는 사용성이나 API 관점에서 더 잘 맞았다. 이 타입들은 또한 API를 작고 명확하게 유지하는 사이에 노출되지 않은것들을 호출하는 장점을 가질 수 있다.


여기에 내가 찾아낸 큰 패키지를 만드는데 도움이 되는 몇가지 팁이 있다. 

1. 각 파일에 관련있는 타입과 코드를 함께 그룹핑하라. 타입과 함수들이 잘 구성되어 있다면 그 파일은 200에서 500줄의 소스코드를 가지는 경향이 있다는걸 발견했다. 이는 많은 것처럼 들릴 수도 있지만 이는 탐색하기가 쉽다는걸 알아냈다. 나의 경우 보통 한 파일의 대한 상한은 1000줄이다.
2. 가장 중요한 타입은 파일의 맨 위에 놓고 밑으로 갈수록 중요성이 낮아지는 순서대로 타입을 추가하라.
3. 애플리케이션이 10,000라인을 넘어가기 시작하면 이 프로젝트를 더 작은 프로젝트들로 나눌 순 없는지를 심각하게 고민해보라.

[Bolt](https://github.com/boltdb/bolt)는 이에 대한 좋은 예제이다. 각 파일은 하나의 Bolt 구조체와 관련 있는 타입들로 그룹핑된다:

```
bucket.go
cursor.go
db.go
freelist.go
node.go
page.go
tx.go
```

<br>

## 결론

코드 구성은 소프트웨어 개발에 있어 가장 어려운 주제중 하나이며 이것이 가져오는 가치에 초점을 맞추는 일은 드물다. 전역 변수를 적게 사용하고, 애플리케이션 바이너리 코드를 패키지로 옮기고, 애플리케이션별 컨텍스트를 위해 타입을 래핑하고, 서브패키지는 제한하라. 이들은 단지 Go 코드를 쉽게 작성하고 더 나은 유지보수가 가능하도록 도와주는 몇가지 트릭들이다.

Go 프로젝트를 Ruby, Java 또는 Node.js 프로젝트와 같은 방식으로 작성하게되면 아마 언어와 싸우게 될 것이다.
