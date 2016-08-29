+++
date = "2016-08-26T15:10:07+08:30"
draft = true 
title = "Go언어 시작하기"
tags = ["beginning", "install"]
categories=["How-to"]
series=["Go 시작하기"]
authors=["Sangbae Yun"]
+++
## Go 언어에 대해서
구글이 2009년에 만든 비교적 새로운 프로그래밍 언어다. 2009년이면 거의 7년 이상된 구닥다리 언어아닌가? 라고 생각 할 수 있겠으나, Ruby(1996년) 나 python(1991년) 과 비교해보면 느낌이 올 것이다. V8 자바스크립트 엔진 개발에 참여했던 Robert Griesemer, UTF-8을 만든 **Rob Pike**, 초창기 유닉스 운영체제를 설계했으며 B언어(C언어의 전신)를 개발한 **Ken Thompson**등 쟁쟁한 개발자들이 만든 언어다. 구글이 개발 했다는 프리미엄과 함께 **도커(Docker)**의 개발 언어라는게 알려지면서 유명세를 타게 됐다.

Python이나 Java와 같은 범용 프로그래밍 언어로 시스템 프로그래밍과 네트워크 프로그램의 개발을 목표로 만들어진 언어다. 비교적 최근에 만들어진 언어답게 C++, Java, Python 언어들의 장점을 상당 부분 수용했다. 이렇게 보면 최신 프로그래밍 언어들의 트랜드를 따를 것 같지만 코드는 **C 언어**와 매우 비슷한 느낌을 준다.

C 언어 처럼 컴파일이 되며, 컴파일 시간에 타입을 체크하는 정적 타입 언어다. 그리고 C 언어처럼 단순하다. 약 25개 정도의 키워드만이 제공되는데, 실제 코드를 만들다 보면 10개 내외의 키워드 만으로 프로그래밍이 가능하다. 아래 go 언어가 제공하는 키워드들이다.  
```
break        default      func         interface    select
case         defer        go           map          struct
chan         else         goto         package      switch
const        fallthrough  if           range        type
continue     for          import       return       var
```
프로그래밍 언어에 대한 경험이 있다면 chan, defer, fallthrough 등을 제외한 키워드들의 이름과 용도를 미루어 짐작할 수 있을 것이다.

키워드가 적은 만큼 복잡한 준비작업 없이 쉽게 시작 할 수 있으며, 몇 개의 키워드로 반복 사용 함으로써 프로그래밍 숙련도를 빠르게 높일 수 있다는 것도 큰 장점이다.

Go 언어는 **동시성(concurrency)**를 잘 지원하는 것으로 유명하다. Go는 고루틴(goroutine)라는 경량스레드(lightweight thread)를 제공하는데, 고루틴간 메시지를 주고 받을 수 있는 채널(channel)을 이용하면, 아주 쉽게(정말 쉽다) 동시성 프로그램을 개발 할 수 있다. 고루틴은 얼랑(Erlang)의 경량 쓰레드와 매우 유사한데, **2k** 정도로 그 크기가 매우 작다. 많은 수의 고루틴을 시스템 부담을 최소화 하면서 만들 수 있다.

Go 언어를 사용하다보면, 웹 애플리케이션을 만들기가 매우 편하다는 느낌을 받게 된다. 특히 **MSA(Microservice Architecture)**와 **REST(Representational State Transfer)** 모델의 애플리케이션을 쉽게 만들 수 있다. 루비나 파이선 같은 언어의 경우 다양한 **웹 프레임워크**중에서 선택을 고민하게 마련인데, Go 언어는 기본으로 제공하는 **net/http** 패키지로 충분하다. 물론 Go 언어도 다양한 마이크로 프레임워크와 풀 프레임워크를 제공하긴 하지만 이런 프레임워크를 쓰면, "왜 프레임워크를 쓰세요 ? 그냥 기본(net/http) 패키지 쓰세요"라는 말을 들을 정도다.

대규모의 분산 시스템을 유지해야 하는 구글의 요구를 위해서 웹 개발 관련 패키지가 강력해진 것 같다.  
## Go 시작하기

### Go 설치
[golang.org](https://golang.org/dl/)에서 운영체제별로 Go 언어를 다운로드 할 수 있다. 2016년 8월 현재 최신 버전은 1.7이다. 압축을 푼 다음 **/usr/local** 디렉토리로 복사했다.
```
# wget https://storage.googleapis.com/golang/go1.7.linux-amd64.tar.gz
# tar -xvzf go1.7.linux-amd64.tar.gz
# mv go /usr/local
```
go 실행을 위해서 환경 변수를 설정했다.
```
# export PATH=$PATH:/usr/local/go/bin
# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```
go를 실행해보자.
```
# go version
go version go1.7 linux/amd64
```

이제 작업 디렉토리 '''workspace'''를 만들었다. Go언어에게 작업 디렉토리를 알려주기 위해서 환경변수 **GOPATH**도 설정했다.
```
# mkdir $HOME/workspace
# export GOPATH:$HOME/workspace
```

이것으로 go 언어 개발 환경을 마쳤다. 이제 Hello World 프로그램을 만들어보고 Go 프로그램의 기본적인 특징들을 살펴보자.
### 패키지 관리 시스템
Hello World 프로그램을 만들기 전에 Go 언어의 패키지 관리 시스템을 사펴봐야 할 것같다. 앞서 나는 workspace 라는 작업 디렉토리를 만들었다. 다른 프로그램이라면 workspace 디렉토리 밑에 프로젝트 파일을 만드는 것으로 개발을 시작할 것이다. 예컨데 mkdir workspace/helloworld 디렉토리를 먼저 만들 것이다.

go 언어는 다르다. 우선 go 언어는 **인터넷**을 기본 개발 환경으로 한다. Go로 원할히 개발하기 위해서는 컴퓨터가 인터넷에 연결되어 있어야 하며, 코드를 저장하고 읽기 위한 github, bitbucket 혹은 직접 구성한 git 서버가 있어야 한다. 즉 go 언어에서 패키지는 프로젝트 저장소 단위로 관리가 된다.

예를 들어 **sqlite3**을 사용하는 애플리케이션을 개발한다고 가정해 보자. 이를 위해서 sqlite3 패키지를 go get으로 다운로드 해야 하는데 git 으로 부터 다운로드 한다.
```
go get github.com/mattn/go-sqlite3
```
거의 모든 패키지가 이처럼 인터넷 상에 있는 git으로 관리 되고 있다. workspace를 살펴보자.
```
.
├── pkg
│   └── linux_amd64
│       └── github.com
│           └── mattn
│               └── go-sqlite3.a
└── src
    └── github.com
        └── mattn
            └── go-sqlite3
                ├── backup.go
                ├── backup_test.go
                ├── callback.go
                ├── callback_test.go
```
패키지의 경로가 github 경로인 것을 확인 할 수 있다. 물론 인터넷에 연결하지 않고도 프로젝트를 수행 할 수는 있지만 제대로 go 프로그래밍을 하려면 인터넷과 github 계정이 필요하다.

아래에서 다룰 Hello World 프로젝트도 github 기반으로 진행 할 것이다. github 계정이 없다면 지금 계정을 만들자. 내가 사용하고 있는 github 계정은 **yundream** 이다.

### Hello World
Hello World 프로젝트를 시작하기 위해서 github 계정에 helloworld 프로젝트를 만들었다. 프로젝트를 만들 때 **Initialize this repository with a README** 옵션을 체크하자. 이제 go get 명령을 이용해서 패키지를 다운로드 한다.
```
$ go get github.com/yundream/helloworld
package github.com/yundream/helloworld: no buildable Go source files in /home/yundream/golang/src/github.com/yundream/helloworld
```
지금은 README.md 파일만 있으므로 빌드 할 수 있는 go 파일이 없다는 경고메시지가 뜨지만 무시하자. **GOPATH** 환경에 등록된 /home/yundream/golang 디렉토리 밑에 패키지를 다운로드 해서 설치하는 것을 확인 할 수 있을 것이다. 디렉토리로 이동해서 helloworld.go 파일을 만들어보자.  
```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("Hello World")
}

```
터미널에 "Hello World"를 출력하는 간단한 프로그램이다. go run 명령으로 실행해보자. 
```
$ go run main.go 
Hello World
```
go run은 임시디렉토리에서 코드를 컴파일 하고 실행하는 일을 한다. go가 컴파일 언어임에도 불구하고 유저 입장에서는 인터프리터 언어처럼 사용 할 수 있다. 물론 python 같은 언어에 비해서는 즉시성이 떨어지기는 하지만 왠만한 프로젝트에서는 굳이 컴파일 과정을 거치지 않고도 바로 바로 실행 할 수 있다.

go build 명령으로 소스코드를 컴파일 할 수 있다.
```sh
$ go build
$ ls
README.md  helloworld  main.go
$ ./helloworld 
Hello World
```
이제 소스코드를 살펴보자. C 언어와 매우 비슷하다는 느낌을 받을 것이다.
```go
package main
```
패키지를 선언한다. 모든 go 언어는 패키지 선언으로 시작해야 한다. 이 패키지이름을 이용해서 코드를 조직화하고 재사용 할 수 있다. C언어와 유사하게 go 언어도 실행 프로그램과 라이브러리를 위한 두 개의 코드 타입을 가지고 있다. 실행 프로그램이란 쉘 에서 명령을 내려서 직접 실행 할 수 있는 (우리가 일반적으로 알고 있는)프로그램이고, 라이브러리는 다른 프로그램에서 이용 할 수 있게 패키징된 코드의 모음이다. 실행 프로그램을 만들기 위한 go 코드는 반드시 **package main**을 선언해야 한다.

```go
import (
  "fmt"
)
```

외부 패키지를 import 하기 위해서 사용한다. java의 import와 매우 유사하다. 위에서 go 코드는 실행 프로그램과 라이브러리 타입이 있다고 했던 것을 기억할 것이다. import는 라이브러리 타입의 go 코드를 재사용 하기 위해서 사용한다. 여기에서는 화면과 파일 출력에 관련된 여러 유용한 기능을 담고 있는 **fmt** 패키지를 import했다.

```go
func main() {
    fmt.Println("Hello World")
}
```
go 언어의 기본 구성요소는 함수이며, func 키워드로 정의해서 사용 할 수 있다. 이 함수의 이름은 main 이며, 0개의 매개변수(parameter)과 0개의 반환 값을 가지고 있다. main은 프로그램의 시작 점이 되는 특수한 함수다. 실행 가능한 타입의 go 코드는 반드시 하나의 main 함수를 가지고 있어야 한다.

```go
fmt.Println("Hello World")
```
**fmt**는 패키지 이름으로 해석하자면 fmt 패키지가 가지고 있는 **Println** 함수를 사용해서 "Hello World"를 출력하라는 의미가 된다.

