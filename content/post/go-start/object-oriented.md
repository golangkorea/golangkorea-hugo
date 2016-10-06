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
Go는 클래스(Class)와 객체(Object) 이 아닌 타입(types)과 값(values)를 가지고 있다. Struct가 Class 비슷한 역할을 하지만 함수도 Struct로부터 분리되는 이상한 구조를 가지고 있다. 다중상속은 말 할 필요도 없고, 단일 상속도 없다. 뭔가 굉장히 객체지향 스럽지 않은 언어로 보일 수 있겠지만 **충분히 객체지향적**이다. 그냥 좀 다른 방법으로 객체를 지향하고 있을 따름이다.

  * 네임스페이스(namespacing)는 exports로 대신한다.  
  * struct가 클래스를 대신한다. 다른 OOP에서의 클래스와는 달리 non-virtual 메서드로만 구성할 수 있다.
  * receiver로 메서드를 만들 수 있다.
  * 인터페이스(interfaces)로 다형성을 구현할 수 있다. 다른 OOP에서는 필드 없이, virtual 메서드로만 구성된 클래스 형태로 구현된다.
  * embedding으로 상속을 대신한다. 객체지향의 composition 모델과 비슷하다.
Go 언어를 이용한 객체지향 프로그래밍 기술에 대해서 살펴보자.

## struct와 메서드
Go언어는 struct가 class 키워드를 대신한다. class와의 눈에 보이는 차이점은 필드에는 real 타입(non-virtual)의 데이터형만 올 수 있다는 점이다. Area 메서드를 가지는 **Rectangle** 스트럭처는 아래와 같이 만들 수 있다.
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
package main

import (
	"fmt"
)

type Rectangle struct {
	Name          string
	Width, Height float64
}

func New(name string) *Rectangle {
	return &Rectangle{Name: name}
}

func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}

func (r *Rectangle) SetWidth(width float64) {
	r.Width = width
}
func (r *Rectangle) SetHeight(height float64) {
	r.Height = height
}

func main() {
	myRectangle := New("Rect-A")
	// 출력 : 0
	fmt.Println(myRectangle.Area())

	myRectangle.SetWidth(52.2)
	myRectangle.SetHeight(30.3)

	// 출력 : 1581.66
	fmt.Println(myRectangle.Area())
}
```
[코드 실행](https://play.golang.org/p/xPeyGF7wIc)

New() 함수를 실행해서 Rectangle 객체를 만들었다. Go 언어는 생성자가 없다.각 데이터 타입의 **기본 값(zero-value)**으로 채워지는 **초기화**가 있을 뿐이다. float64는 0.0, string은 "", 포인터는 nil이 할당된다. 따라서 Rectangle 객체를 만든 후 Area() 메서드를 실행하면 **0**이 출력된다.

두 번째 방법으로 구조체로 부터 직접 객체를 만드는 방법이 있다.
```go
func main() {
	myRectangle := Rectangle{Name:"Rect-A", Width:12.5, Height:13.5}
}
```

SetWidth()와 SetHeight() 메서드는 **포인터형 리시버**를 사용하고 있다. **Value형 리시버**로 바꾼 다음에 테스트 해보자. 모든 출력이 0 인 것을 확인 할 수 있을 것이다.
```go
func (r Rectangle) SetWidth(width float64) {
	r.Width = width
}
func (r Rectangle) SetHeight(height float64) {
	r.Height = height
}
```

## embedding를 이용한 상속의 구현 
객체지향 언어에서의 상속은 아래 처럼 묘사 할 수 있다.
![OOP에서의 상속](https://docs.google.com/drawings/d/1AZ2UiAJt44aWogpg6th_v3u0C0CeOK7g2Pofi1iWBQ8/pub?w=631&h=420)
이 (UML 스러운)그림이 내포하는 의미는 **원형 클래스의 특징을 물려 받은 하위 원형 클래스를 만들 겠다**라는 것이다. 상위 클래스를 부모 클래스, 하위 클래스를 자식 클래스라고 부른다. 이런 식의 분류법은 널리 사용한다. 생물학 시간에 배웠을 **종 > 속 > 과 > 목 > 강 > 문 > 계**가 이런 계층적인 형태를 생각하면 된다.

시각을 바꿔보자. 상위 클래스로 부터 특징을 물려받는다 대신 **상위 클래스의 특징을 포함(embedding)하는**식으로 접근하면 어떨까 ? 접근 방식은 다르지만 상속과 마찬가지의 효과를 누릴 수 있다. 다중상속은 ? 두 개 이상의 클래스의 특징을 포함하면 된다. 

계층적으로 이루어지는 상속 관계를 **IS - A Relationship**, 포함하는 식으로 이루어지는 관계를 **Has a Relationship**이라고 한다. Go 언어는 Has a Relationship으로 상속 관계를 나타낼 수 있다. 위의 Bicycle를 Has a Relationship으로 나타내보자. 
![Has a relicationship](https://docs.google.com/drawings/d/1xorVqrumZRT-EqCbW5cdBiD0Lj0yyZkp8sPGh3JeVEw/pub?w=631&h=420)

MountainBike, RoadBike, TandemBike가 Bicycle 스크럭처를 가지는 방식으로 상속을 구현하고 있다. 아래 코드를 보자.
```go
package main

import (
    "fmt"
)

type Bicycle struct {
}

func (b Bicycle) Spare() {
    fmt.Println("Bicycle's Spare")
}

func (b Bicycle) Run() {
    fmt.Println("Bicycle Run")
}

type MountainBike struct {
    Bicycle
}

func (m MountainBike) Run() {
    fmt.Println("Mountain Bicycle Run")
}
func (m MountainBike) Jump() {
    fmt.Println("Mountain Bicycle Jump")
}

type RoadBike struct {
    Bicycle
}

func main() {
    rBike := RoadBike{}
    rBike.Run()
    mBike := MountainBike{}
    mBike.Run()
    mBike.Jump()
}
```
[코드 실행](https://play.golang.org/p/A5K7RfPQyS)

MountainBike 구조체와 RoadBike 구조체에 Bicycle 구조체를 **embeded** 했다. Bicycle 구조체의 메서드들을 마치 자신의 메서드인 것처럼 사용 할 수 있으며, **오버라이딩**도 할 수 있다.

## Interface
인터페이스(interface)는 메서드만을 정의한다. 메서드들에 대한 구현은  
  * interface를 이용한 다형성
  * empty interface

## 참고
  * [golang oop](https://github.com/luciotato/golang-notes/blob/master/OOP.md)
