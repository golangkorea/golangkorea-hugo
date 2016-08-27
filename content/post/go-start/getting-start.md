+++
date = "2016-08-26T15:10:07+08:30"
draft = false 
title = "Go언어 시작하기"
tags = ["beginning", "install"]
categories=["How-to"]
series=["Go 시작하기"]
authors=["Sangbae Yun"]
+++
## Go 언어에 대해서
구글이 2009년에 만든 비교적 새로운 프로그래밍 언어다. 2009년이면 거의 7년 이상된 구닥다리 언어아닌가? 라고 생각 할 수 있겠으나, Ruby(1996년) 나 python(1991년) 과 비교해보면 느낌이 올 것이다. V8 자바스크립트 엔진 개발에 참여했던 Robert Griesemer, 켄 톰슨과 UTF-8을 만든 **Rob Pike**, 초창기 유닉스 운영체제를 설계했으며 B언어(C언어의 전신)를 개발한 **Ken Thompson**등 쟁쟁한 개발자들이 만든 언어다. 구글이 개발 했다는 프리미엄과 함께 **도커(Docker)**의 개발 언어라는게 알려지면서 유명세를 타게 됐다.

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
