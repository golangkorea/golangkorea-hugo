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
프로그래밍 언어에 대한 경험이 있다면 chan, defer, fallthrough 등을 제외한 키워드들의 이름과 용도를 미루어 짐작할 수 있을 것이다. 복잡한 준비작업 없이 쉽게 시작 할 수 있다는 이야기다.

Go 언어는 동시성(concurrency)를 잘 지원하는 것으로 유명하다. Go는 고루틴(goroutine)라는 경량스레드(lightweight thread)를 제공하는데, 고루틴간 메시지를 주고 받을 수 있는 채널(channel)을 이용하면, 아주 쉽게(정말 쉽다) 동시성 프로그램을 개발 할 수 있다. 고루틴은 얼랑(Erlang)의 경량 쓰레드와 매우 유사한데, '''2k''' 정도로 그 크기가 매우 작다. 많은 수의 고루틴을 시스템 부담을 최소화 하면서 만들 수 있다.

Go 언어를 사용하다보면, 웹 애플리케이션을 만들기가 매우 편하다는 느낌을 받게 된다. 특히 MSA(Microservice Architecture)와 REST(Representational State Transfer) 모델의 애플리케이션을 쉽게 만들 수 있다. 루비나 파이선 같은 언어의 경우 다양한 웹 프레임워크를 고려하기 마련인데, Go 언어는 기본으로 제공하는 **net/http** 패키지로 충분하다. 물론 Go 언어도 다양한 마이크로 프레임워크와 풀 프레임워크를 제공하긴 하지만 이런 프레임워크를 쓰면, "왜 프레임워크를 쓰세요 ? 그냥 기본(net/http) 패키지 쓰세요"라는 말을 들을 정도다.

아마도 대규모의 분산 시스템을 유지해야 하는 구글의 요구사항이 언어에 반영된 것으로 보인다. 
## Go 시작하기 

### Go 설치 
### Hello World
### 패키지 관리 시스템 
