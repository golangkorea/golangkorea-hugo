+++
date = "2016-09-12T13:10:03+09:00"
draft = false
title = "Go의 주요 특징들"
authors = ["Sangbae Yun"]
categories = ["How-to"]
tags = ["beginning"]
series=["Go 시작하기"]
toc = false
+++

## 단순함
Go 언어는 단순함(simplicity)과 실용성(pragmatism)을 지향하는 언어로 이 두가지 철학이 다른 모든 것들 보다 상위에 있다. go 언어에 없는 것들을 보자. 
  
 * 패턴매칭 
 * 함수 프로그래밍 : 어느 정도 특징을 가지고 있기는 하지만 지향점은 아니다.
 * immutable variables 
 * Option types : 값외에 유효한지, 초기화가 됐는지 등의 추가적인 정보를 설정할 수 있다. 
 * 예외(exception)가 없다.
 * 클래스도 없다. 
 * 제너릭(generics)를 지원하지도 않는다.

현대적인 언어들이라면 당연히 가지고 있음직한 굵직한 특성들을 가지고 있지 않다. 심지어 "Go는 40년 동안의 프로그래밍 언어에 대한 연구를 던져버린 유일한 언어"라고 평가를 받기도 한다(제너릭의 경우 지원하려는 움직임이 있는 것 같기는 하다). 그리고 이러한 철학을 그대로 하고 있는데, 1.0 버전이 나온 이후 1.7 까지 문법적인 변화가 거의 없다. 

1.0 이 나온게 2012년이니 5년 동안 변한게 없다는 이야기다. 따라서 개발자는 호환성 문제에서 자유로우며, 기술에 대한 숙련도를 꾸준히 유지 할 수 있다. 단순함을 포기하지 않기 때문에 가능한 일이다. [Go release History](https://golang.org/doc/devel/release.html)에서 버전별 변경점을 찾아 볼 수 있는데, 버그 수정, 지원 플랫폼, 툴 추가, 컴파일러 변경, 가비지 컬랙터 효율화 등 언어 내적인 것들이 대부분이다.

계속 단순함을 유지하면서, 언어적인 발전이 가능 할 것인지에 대한 의구심을 가질 수 있다. 이렇게 생각해보자. 복싱은 주먹을 사용하는 격투기 중 최고로 평가받고 있다. 그런데 복싱이 가지고 있는 기술이라는게 스트레이트, 잽, 어퍼, 훅 4가지 밖에 없다. 기술이 적기 때문에 시작하기가 쉽고 반복훈련을 통해서 빠르게 기량을 높일 수 있다. 그리고 직관적인 만큼 실전에서의 응용이 용이하다. 

Go 언어도 마찬가지다. 1-2주면 언어의 거의 모든 기능에 익숙해질 수 있으며, 반복 훈련을 통해서 빠르게 기량을 높일 수 있다. 코드가 직관적이기 때문에 코드를 만들고 읽는게 쉬우며 그만큼 실전에 빠르게 써먹을 수 있다.

물론 언어의 단순함이 모든 경우에 장점이 될 수는 없을 것이다. Go 언어는 시스템, 네트워크 프로그램 특히 클라우드 환경에서 작동하는 프로그램의 개발에는 강력한 면모를 보여주지만 모바일, 데스크탑 애플리케이션에도 강점을 보여줄지는 의문이다(애초에 이쪽은 별로 신경을 쓰고 있지 않기 때문에 판단하기는 애매모호하긴 하다). 

## 클라우드와 친한 go 언어
단순함과 이로부터 파생되는 특징은 클라우드 환경에 잘 맞는 경향이 있다. 분산환경은 시스템이 분산된다는 의미외에 소프트웨어가 분산된다는 의미도 있다. 이런 환경에서는 소프트웨어들이 많은 기능을 가지고 있을 필요가 없다. 필수적인 기능만 가진 여러 소프트웨어들이 서로 데이터를 주고 받는 식으로 작동을 하는게 더 효율적이다. 이런 소프트웨어 운영 모델은 리눅스에서 찾아볼 수 있다.    
```sh
# ps aux | grep chorm | grep -v grep | awk '{print $2}' | xargs kill 
```
ps로 프로세스 목록을 출력하면 grep으로 chrom 프로세스의 정보만 가져오고, awk를 이용해서 **PID**를 읽어서 kill로 죽이는 일을 하는 스크립트다. 클라우드환경에서 뜨는 **MSA**(go언어를 이용한 MSA 문서를 만들어봐야 겠다.)가 이런 방식으로 작동한다.

클라우드는 컴퓨터와 네트워크, 운영체제를 하나로 통합한다. 이런 환경에서 프로그래밍 언어의 버전, 라이브러리 의존성을 신경쓰면서 애플리케이션을 배포하는 건 굉장히 어려운 일이다. 최근 도커(docker)가 핫한 것도 운영체제 등 주변환경이 어떻든지 간에 자유롭게 배포 할 수 있고, 동일하게 작동 할 것을 보장해 주기 때문이다. 

Go 언어로도 이런 개발 & 배포 환경을 만들 수 있다. 도커와 함께 클라우드를 위한 컨테이너 솔류션을 만들고 싶다면 Go는 최고의 선택이 될 것이다.

## struct를 이용한 객체지향
Go는 클래스와 객체가 없다. 그렇다고 해서 객체지향 언어가 아니라고 하기도 그렇다. 원래 객체지향이라는 것은 프로그래밍 방법론으로 언어와 상관이 있는 것은 아니다. C언어로도 객체지향을 할 수 있고, C++로도 절차지향을 할 수 있다. 다만 얼마나 객체지향 프로그래밍을 잘 지원하느냐에 대한 차이는 있는데, 표면적으로는 클래스와 객체가 있는지를 기준으로 삼는 경우가 많다. 상속역시 지원하지 않는다. Go는 전통적인 의미에서의 객체지향 언어라고 하기는 애매모호 하다.

하지만 메서드를 만들 수 있으며, interface를 이용해서 다형성을 구현 할 수도 있다. **composition**으로 상속을 대신 할 수도 있다. 뭔가 편법을 동원한다는 느낌이 들 수도 있겠지만, 객체지향에 있어서 반드시 무엇을 해야 한다는 어떤 규칙은 없다. Sandi Metz은 이렇게 말하고 있다. "객체지향에 있어서 클래스와 상속은 옵션이며, 한 문제는 다양한 방법으로 풀 수 있다." 

struct는 하나이 상의 필드들로 구성된 데이터 타입으로 레코드 형식의 데이터 그룹을 만들기 위해서 사용한다. 개인 정보를 다루는 애플리케이션을 개발한다면 아래와 같은 **person** 스트럭처를 만들 수 있을 것이다. 
```go
type person struct {
    name string
    age  int
}
```

소프트웨어 공학에서 기본적으로 클래스는 속성과 메서드의 모음으로 표현된다. 파이슨의 경우를 보자. 
```python
class Person:
    minAge = 0
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def Hello(self):
        print("Hello. My name is %s" % self.name)
    def MyAge(self):
        print("My age is %s" % self.age)
```
minAge와 self.name, self.age라는 속성과 Hello, MyAge라는 메서드를 가지는 Person 클래스를 만들었다. 일반적으로 알고 있는 클래스의 모습이다.

반면 go는 struct와 메서드가 서로 분리된다. 위의 파이슨 코드를 go 코드로 만들어 봤다.
```go
package main

import (
    "fmt"
)

type Person struct {
    minAge int
    name   string
    age    int
}

func (p Person) Hello() {
    fmt.Printf("Hello. My name is %s\n", p.name)
}

func (p Person) MyAge() {
    fmt.Printf("My age is %d\n", p.age)
}

func main() {
    yundream := Person{minAge: 0, name: "yundream", age: 33}
    yundream.Hello()
    yundream.MyAge()
}
```
[Playground](https://play.golang.org/p/Slj3hhovW4)

뭔가 굉장히 낯설어 보인다. 일단 캡슐화는 지원한다. 보통 **private, public** 키워드를 이용하는데, Go언어는 그런 거 없다. 대신 대/소 문자로 구분을 한다. 대문자로 시작하면 public, 소문자로 시작하면 private가 되는 식이다.  private 변수나 메서드는 패키지 내에서만 사용 할 수 있다.

메서드가 구조체와 분리되기 때문에, 이 메서드가 어느 구조체에 연결된 것인지를 구분해야 한다. **수신자(receiver - func 키워드와 함수명 사이에 위치한다)**를 이용해서, 연결된 구조체를 확인 할 수 있다. 메서드는 **.** 연산자를 이용해서 호출 할 수 있다.

Go는 생성자가 없다. 예제에서 처럼, 구조체를 생성 할 때 초기값을 할당 하거나 혹은 구조체 객체의 포인터를 반환하는 **New** 함수를 만들어서 사용한다. 
```go
func New(minAge int, name string, int age) *Person {
    return &Person{minAge: minAge, name: name, age: age}
}
```

## 에러 처리
Go는 예외(execption)가 없다. C언어와 같이 반환 값이 에러인지 아닌지를 비교하는 방법으로 에러를 처리한다. 대신 에러만을 전문적으로 처리하는 **error** 타입을 내장하고 있다. Go 프로그램은 error 값을 검사하는 것으로 에러 상태를 확인 할 수 있다.  

또한 go는 두 개 이상의 값을 반환 할 수 있다. 이 특징을 이용하면 실행 반환 값과 에러를 함께 넘기는 방식으로 에러를 처리할 수 있다. 예를 들어 os.Open 함수는 열린 파일의 데이터를 담고 있는 File 구조체와 error를 함께 반환한다. 
```go
func Open(name string) (file *File, err error)
```

코드에서는 error 값이 "nil"인지 아닌지로 에러를 검사한다. 
```go
f, err := os.Open("filename.txt")
if err != nil {
    fmt.Println("File open error : ", err.Error())
    os.Exit(1)
}
// 파일 처리
```

예외 처리가 없기 때문에 C 언어처럼 모든 에러 리턴에 대한 코드를 만들어야 한다. 함수를 만들다 보면 에러처리 코드가 절반이상을 차지하는 것을 심심찮게 볼 수 있다.

개발자는 **errors** 패키지를 이용해서 직접 에러를 만들 수 있다. 
```go
package main

import (
    "errors"
    "fmt"
    "os"
)

func YourLevel(point int) (int, error) {
    if point < 0 {
        return 0, errors.New("Level: 레벨 값은 0보다 커야 합니다.")
    }
    if point > 255 {
        return 0, errors.New("Level: 레벨 값은 255보다 작아야 합니다.")
    }
    return point / 10, nil
}

func main() {
    level, err := YourLevel(25)
    if err != nil {
        fmt.Println("Error ", err.Error())
        os.Exit(1)
    }
    fmt.Printf("당신의 레벨은 %d 입니다.\n", level)
}
```
[Playground](https://play.golang.org/p/zKZHQ7Obm1)

실제 코드에서는 아래와 같이 에러 케이스를 정의해서 사용한다. 위 코드를 약간 수정했다.
```go
var StatusPointUnberZero = errors.New("Level: 레벨 값은 0보다 커야 합니다.")
var StatusPointOverflow = errors.New("Level: 레벨 값은 255보다 작아야 합니다.")

func YourLevel(point int) (int, error) {
    if point < 0 {
        return 0, StatusPointUnberZero
    }   
    if point > 255 {
        return 0, StatusPointOverflow
    }   
    return point / 10, nil
}   
func main() {
    level, err := YourLevel(25)
    switch err {
    case StatusPointUnberZero:
        // 에러처리 코드
    case StatusPointOverflow:
        // 에러처리코드
    default:
        // 에러처리 코드
    }  
    fmt.Printf("당신의 레벨은 %d 입니다.\n", level)
}
```

다른 예제를 이용해서 **error**를 이용한 에러 처리가 가지는 장점을 살펴보자. 아래 프로그램은 입력 값이 양수인지 음수인지를 검사한다.
```go
package main

import "fmt"

// Positive returns true if the number is positive, false if it is negative.
func Positive(n int) bool {
        return n > -1
}

func Check(n int) {
        if Positive(n) {
                fmt.Println(n, "is positive")
        } else {
                fmt.Println(n, "is negative")
        }
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}
```
[Playground](https://play.golang.org/p/oMfSuAqw74)

프로그램의 실행 결과다. 버그를 가지고 있음을 알 수 있다.
```
1 is positive
0 is positive
-1 is negative
```
0은 양수도 아니고 음수도 아니다. 양수, 음수, 0 이렇게 3개의 상태를 가지기 때문에 boolean 으로는 이들 상태를 처리 할 수없다.

Go언어는 2개 이상의 반환 값을 가질 수 있으므로, 0인지를 측정 할 수 있는 값을 반환하면 된다.
```go
// 입력값이 0일 경우 두번째 리턴값으로 false를 반환한다.
func Positive(n int) (bool, bool) {
    if n == 0 {
        return false, false
    }
    return n > -1, true
}

func Check(n int) {
    pos, ok := Positive(n)
    if !ok {
        fmt.Println(n, "is neither")
        return
    }
    if pos {
        fmt.Println(n, "is positive")
    } else {
        fmt.Println(n, "is negative")
    }
}
```
프로그램을 실행해보자. 버그가 잡혔다.
```
1 is positive
0 is neither
-1 is negative
```

문제없이 작동하지만 좋은 코드는 아니다. 일단 직관적이지 않다. 코드를 열어 보기 전에는 두 개의 boolean 반환 값이 무엇을 의미하는지 알 수가 없다. error를 이용해서 0 값을 예외처리하도록 바꿔보자. 
```go
func Positive(n int) (bool, error) {
    if n == 0 {
        return false, errors.New("undefined")
    }
    return n > -1, nil
}

func Check(n int) {
    pos, err := Positive(n)
    if err != nil {
        fmt.Println(n, err)
        return
    }
    if pos {
        fmt.Println(n, "is positive")
    } else {
        fmt.Println(n, "is negative")
    }
}
```
하는 일은 차이가 없으나 코드가 명료해 졌다. 

## 동시성 프로그래밍 
Go에서 제공하는 고루틴이라고 기능을 이용해서 다른 함수를 동시에 실행 할 수 있다. 쓰레드와 비슷하게 작동한다. Go에서 고루틴은 **일급 객체(first class)**로 정수(integer)나 실수(floating point number)와 같은 데이터 타입과 동급으로 취급한다. 일급객체에 대해서는 [wikipedia 문서 ](https://en.wikipedia.org/wiki/First-class_citizen)를 참고하자. 아래 예제 코드를 보자.
```go
package main

import "fmt"

func f(id int) {
    for i := 0; i < 10; i++ {
        fmt.Println(id, ":", i)
    }
}

func main() {
    go f(0)
    var input string
    fmt.Scanln(&input)
}
```

go 키워드 뒤에 동시 실행할 함수를 두면, 해당 함수를 실행하는 고루틴이 만들어진다. 고루틴은 main 함수는 서로 독립적으로 진행이 된다. main 함수가 고루틴 보다 먼저 종료 할 수 있기 때문에 **Scanln** 함수를 이용해서 기다리게 했다.

10개의 고루틴을 만들어보자. 그냥 go를 열번 호출하면 된다.
```go
# go run goroutine.go 
4 : 0
6 : 0
4 : 1
4 : 2
4 : 3
```

채널(channel)은 고루틴들 간에 데이터를 교환하기 위해서 사용한다. Go도 공유 잠금을 지원하기는 하지만 메시지 교환방식을 선호한다. 아래는 고루틴간 ping 메시지를 교환하는 간단한 프로그램이다.
```go
package main

import (
    "fmt"
    "time"
)

func pinger(c chan string) {
    for i := 0; ; i++ {
        c <- "ping"
    }
}

func pingPrinter(c chan string) {
    for {
        msg := <-c
        fmt.Println(msg)
        time.Sleep(time.Second * 1)
    }
}

func main() {
    var c chan string = make(chan string)

    go pinger(c)
    go pingPrinter(c)

    var input string
    fmt.Scanln(&input)
}
```
**chan** 키워드를 이용해서 채널 타입 데이터를 만들 수 있다. 채널은 메시지를 주고 받는 통로 역할을 하는데 struct를 포함한 모든 종류의 데이터들을 주고 받을 수 있다. 코드에서는 string 타입 데이터를 위한 채널을 만들었다.   

**<-** 연산자를 이용해서 채널에 데이터를 쓰거나 읽을 수 있다. **c <- "ping"**는 채널에 "ping"을 쓰겠다는 의미고, **msg := <-c**는 채널에서 읽은 데이터를 msg에 저장하겠다는 의미다.

## 인터페이스
인터페이스는 **메서드**들의 모음으로 간단히 정의 할 수 있다. 또한 그 자체로 하나의 타입이기도 하다. 메서드들의 형태만 정의하고, 구현은 외부에 맡기는 방식으로 유연한 코드를 만들 수 있다.

```go
import (
    "fmt"
    "math"
)

type Shape interface {
    Area() float64
    Type()
}

type Rectangle struct {
    width  float64
    height float64
}

func (r Rectangle) Area() float64 {
    return r.width * r.height
}

func (r Rectangle) Type() {
    fmt.Println("I'm rectangle")
}

type Circle struct {
    radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.radius * c.radius
}

func (c Circle) Type() {
    fmt.Println("I'm circle")
}

func main() {
    rec := []Shape{
        Rectangle{width: 10, height: 20},
        Circle{radius: 12},
    }

    for _, s := range rec {
        s.Type()
        fmt.Println("Area :", s.Area())
        fmt.Println("===========")
    }
} 
```
실행 결과
```
I'm rectangle
Area : 200
===========
I'm circle
Area : 452.3893421169302
===========
```
### 웹 프로그래밍
Go는 특히 MSA모델의 웹 애플리케이션 개발을 잘 지원한다. 기본으로 지원하는 **net/http**와 [gorilla](http://www.gorillatoolkit.org/)만으로도 훌륭하게 작동하는 웹 애플리케이션 서버를 개발 할 수 있다. 다른 프레임워크 사용 할 필요 없다. [그리고 성능도 매우 뛰어나다](http://www.joinc.co.kr/w/man/12/golang/HTTPPerf).

아래는 net/http와 gorilla를 이용해서 만든 간단한 웹 애플리케이션 서버다.
```go
package main

import (
    "fmt"
    "github.com/gorilla/mux"
    "net/http"
    "strconv"
)

func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello world")
}

func sum(w http.ResponseWriter, r *http.Request) {
    v := mux.Vars(r)
    a, _ := strconv.Atoi(v["a"])
    b, _ := strconv.Atoi(v["b"])
    fmt.Fprintf(w, "%d", a+b)
}
func main() {
    h := mux.NewRouter()
    h.HandleFunc("/hello", hello).Methods("GET")
    h.HandleFunc("/sum/{a}/{b}", sum).Methods("GET")
    http.Handle("/", h)
    http.ListenAndServe(":3000", nil)
}
```
군더더기를 찾아볼 수 없는 단순하고 이해하기 쉬운 코드다. 

### 유닛테스트
Go언어는 분산환경에 최적화된 측면이 있다. 분산환경에서는 테스트해야 할 기능이 명확하기 때문에 유닛테스트의 효과를 크게 누릴 수 있다. 특히 Go 언어는 **Simple is best** 철학을 지향하기 때문에, 유닛테스트의 활용이 중요하다. Go가 유닛테스트를 기본으로 제공하는 것도 이런 이유 때문일 것이다. 

유닛 테스트를 위해서 **mymath**라는 간단한 패키지를 만들었다. 이 코드는 [github](https://github.com/yundream/mymath)에서 다운로드 할 수 있다.
```go
package mymath

import (
    "errors"
)

var (
    StatusDivideZero = errors.New("Divide zero")
)

func Div(a float64, b float64) (float64, error) {
    if b == 0 {
        return 0, StatusDivideZero
    }
    return a / b, nil
}
```
아래는 테스트 코드다.
```go
package mymath

import (
    "testing"
)

func Test_Div(t *testing.T) {
    _, err := Div(1, 0)
    if err != StatusDivideZero {
        t.Error("Divide zero")
    }
    v, err := Div(10, 5)
    if v != 2 {
        t.Fatal("10/5 = 2 but ", v)
    }
}
```
**go test** 명령을 실행하면, 현재 패키지 디렉토리에 있는 파일에서 테스트 코드를 찾아서 실행 한다. 함수의 이름이 **Test** 로 시작하고 *testing.T 를 매개변수로 사용하면 테스트 함수인 것으로 간주한다.
```go
# go test
PASS
ok  	github.com/yundream/mymath	0.001s
```
**-cover** 옵션을 이용하면 테스트 커버리지 레포팅도 할 수 있다.
```go
# go test -cover
PASS
coverage: 100.0% of statements
ok  	github.com/yundream/mymath	0.001s
```
Go는 함수 단위의 유닛 테스트 도구 뿐만 아니라 웹 애플리케이션 서버 단위의 테스트 툴도 제공한다. 직접 웹 서버를 실행해서 핸들러들을 테스트하고 커버리지를 측정하는 식으로 작동한다. 웹 애플리케이션 서버 개발 편에서 자세히 다뤄볼 계획이다.

### 문서화
코드 문서화 도구까지 기본 툴로 제공하고 있다.

### 마치며 
여기에서는 Go 언어의 주요 특징들만 간단하게 살펴봤다. 자세한 내용들은 주제별로 따로 다루도록 하겠다.

