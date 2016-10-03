+++
authors = ["Sangbae Yun"]
date = "2016-10-02T00:51:06+09:00"
draft = false 
categories = ["How-to"]
tags = ["Object Oriented","struct"]
series=["Go 시작하기"]
title = "object oriented"
toc = false
+++

## 객체지향 프로그래밍
Go는 클래스(Class)와 객체(Object) 이 아닌 타입(types)과 값(values)를 가지고 있다. Struct가 Class 비슷한 역할을 하지만 함수도 Struct로부터 분리되는 이상한 구조를 가지고 있다. 다중상속은[[FootNote(파이슨이나 루비 같은 언어도 다중 상속을 제공하지는 않는다.)]] 말 할 필요도 없고, 단일 상속도 없다. 뭔가 굉장히 객체지향 스럽지 않은 언어로 보일 수 있겠지만 '''충분히 객체지향적'''이다. 그냥 좀 다른 방법으로 객체를 지향하고 있을 따름이다.

  * 네임스페이스(namespacing)는 exports로 대신한다.  
  * struct가 클래스를 대신한다. 다른 OOP에서의 클래스와는 달리 non-virtual 메서드로만 구성할 수 있다.
  * receiver로 메서드를 만들 수 있다.
  * 인터페이스(interfaces)로 다형성을 구현할 수 있다. 다른 OOP에서는 필드 없이, virtual 메서드로만 구성된 클래스 형태로 구현된다.
  * embedding으로 상속을 대신한다. 객체지향의 composition 모델과 비슷하다.
Go 언어를 이용한 객체지향 프로그래밍 기술에 대해서 살펴보자.

## struct와 메서드
Go언어는 struct가 class 키워드를 대신한다. class와의 눈에 보이는 차이점은 필드에는 real 타입(non-virtual)의 데이터형만 올 수 있다는 점이다. Area 메서드를 가지는 '''Rectangle''' 스트럭처는 아래와 같이 만들 수 있다.
```go
type Rectangle struct {
	Name	string
	Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}
```
class 키워드를 가지고 있는 객체지향 언어의 경우 아래와 같은 의사코드로 표현할 수 있을 것이다.
```
class Rectangle
   field Name: string
   field Width: float64
   field Height: float64
   method Area() 
       return this.Width * this.Height
```
그리고 struct 내에 메서드를 포함 할 수 없다. 스트럭처 바깥에 만들어지며, 리시버(receiver)를 이용해서 어느 스트럭처의 메서드인지를 나타낸다. 아래 그럼처럼 묘사 할 수 있다.

![receiver](https://docs.google.com/drawings/d/1rBOgYujGOIy9EL6U040nCSCC61pwzFe-BhoWNkVoksU/pub?w=756&h=205)

리시버는 ***Value 리시버***와 ***포인터 리시버*** 두 가지 타입이 있다. 아래 코드를 보자. 
```go
package main

import "fmt"

type Mutatable struct {
    a int
    b int
}

func (m Mutatable) StayTheSame() {
    m.a = 5
    m.b = 7
}

func (m *Mutatable) Mutate() {
    m.a = 5
    m.b = 7
}

func main() {
    m := &Mutatable{0, 0}
    fmt.Println(m)
    m.StayTheSame()
    fmt.Println(m)
    m.Mutate()
    fmt.Println(m)
}
```
[코드 실행](https://play.golang.org/p/RY6m5sE2H-)

StayTheSame은 Value 리시버이고 Mutate는 포인터 리시버다. 포인터 리시버의 경우 스트럭처의 값을 변경(Mutate)하는 반면 Value 리시버는 스트럭처의 값을 변경하지 않는다는 것을 알 수 있다. Mutate 하느냐 하지 않느냐가 Value 리시버와 포인터 리시버의 눈에 보이는 결정적인 차이다. 포인터라는게 데이터가 저장된 주소를 가리킨다는 것을 생각해보면, 포인터 리시버의 **Mutate** 한 성질을 유추해 낼 수 있을 것이다.

Go는 생성자를 지원하지 않는다. 하지만 쉽게 생성자(처럼 작동하도록)를 구현 할 수 있다. 보통 두 가지 방법을 사용한다. 첫 번째 방법은 패키지에 객체를 반환하는 **New** 같은 함수를 만드는 것이다. 빌더 패턴(builder pattern)의 응용이다.
```go
```

## Composition 모델 
  * Composition 모델에 대한 일반적인 설명
  * composition 예제 

## Interface
  * interface를 이용한 다형성
  * empty interface

## 참고
  * [golang oop](https://github.com/luciotato/golang-notes/blob/master/OOP.md)
