+++
authors = ["Sangbae Yun"]
date = "2016-10-02T00:51:06+09:00"
draft = false
categories = ["How-to"]
tags = ["Object Oriented","struct"]
series=["Go 시작하기"]
title = "Go와 객체지향"
toc = false
+++

## 객체지향 프로그래밍
Go는 클래스(Class)가 없다!! Struct가 Class의 역할을 수행 할 수 있기는 하지만 메서드도 구조체로부터 분리되는 구성을 가지고 있다. 단일 상속도 없고 당연히 다중 상속도 없다. 왠지 객체지향스럽지 않은 언어로 보일 수 있겠지만 **충분히 객체지향적**이다. 그냥 좀 다른 방법으로 객체를 지향하고 있을 따름이다. 

  * struct가 클래스를 대신한다. 다른 OOP에서의 클래스와는 달리 non-virtual(real) 메서드로만 구성된다.
  * receiver로 구조체와 함수를 연결 해서 메서드를 구현한다. 
  * 네임스페이스(namespacing)는 exports로 대신한다.  
  * 인터페이스(interfaces)로 다형성을 구현할 수 있다. 다른 OOP에서는 필드 없이, virtual 메서드로만 구성된 클래스 형태로 구현된다.
  * embedding으로 상속을 대신한다. 객체지향의 composition 모델과 비슷하다.
Go 언어를 이용한 객체지향 프로그래밍 기술에 대해서 살펴보자.

## struct(구조체)와 메서드
Go언어는 struct가 class 키워드를 대신한다. class와의 눈에 보이는 차이점은 real 타입(non-virtual)의 메서드만 올 수 있다는 점이다. Area 메서드를 가지는 **Rectangle** 구조체는 아래와 같이 만들 수 있다.
```go
type Rectangle struct {
    Name    string
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}
```
class 키워드를 가지고 있는 객체지향 언어의 경우 아래와 같은 의사코드로 표현할 것이다.
```
class Rectangle
   field Name: string
   field Width: float64
   field Height: float64
   method Area() 
       return this.Width * this.Height
```
Go는 구조체 내에 메서드를 포함 할 수 없다. 구조체 바깥에 만들어지며, **리시버(receiver)**를 이용해서 어느 구조체의 메서드인지를 정의 할 수 있다. 아래 그럼은 리시버를 이용해서 구조체와 함수가 연결되는 과정을 묘사하고 있다.

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

**StayTheSame**은 Value 리시버이고 **Mutate**는 포인터 리시버다. 포인터 리시버의 경우 구조체의 필드 값을 변경(Mutate)하는 반면 Value 리시버는 스트럭처의 값을 변경하지 않는다. Mutate 하느냐 하지 않느냐가 Value 리시버와 포인터 리시버의 눈에 보이는 차이다. 포인터라는게 데이터가 저장된 주소를 가리킨다는 것을 생각해보면, 포인터 리시버의 **Mutate** 한 성질을 유추 할 수 있을 것이다.

이제 구조체로 부터 객체를 만들어 보자. 몇 가지 방법이 있는데, 첫 번째 방법은 패키지에 객체를 반환하는 **New** 같은 함수를 만드는 것이다. **빌더 패턴(builder pattern)**의 응용이다.
```go
package main

import (
    "fmt"
)

type Rectangle struct {
    Name          string
    Width, Height float64
}

// Rectangle 를 반환하는 함수를 만들었다.
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

New() 함수를 실행해서 Rectangle 객체를 만들었다. 

두 번째로 구조체의 초기화 문법을 이용해서 직접 객체를 만드는 방법이 있다.
```go
func main() {
    yourRectangle := Rectangle{}
    myRectangle := Rectangle{Name:"Rect-A", Width:12.5, Height:13.5}
}
```
구조체 초기화 문법을 이용 하면, 필드의 값을 초기화 할 수 있다. 초기화 하지 않는 필드들은 **기본 값(zero-value)**으로 초기화 된다. float64는 0, string은 "", 포인터는 nil로 초기화 된다. 예를 들어서 위 코드의 yourRectangle의 경우 Height와 Width가 0으로 초기화 되기 때문에 yourRectangle.Area()는 0을 반환 할 것이다.  

마지막으로 Go의 new()내장 함수를 이용하는 방법이 있다. 
```go
func new(Type) *Type
```
new 함수를 호출하고 나면, 메모리를 할당하고 포인터를 반환한다. 필드의 값들은 기본 값으로 초기화 된다. c++의 new와 사용방법이 비슷하다. 
```go
func main() {
    myRectangle := new(Rectangle)
    // 출력 : 0
    fmt.Println(myRectangle.Area())

    myRectangle.SetWidth(52.2)
    myRectangle.SetHeight(30.3)

    // 출력 : 1581.66
    fmt.Println(myRectangle.Area())
}
```

Go 언어는 생성자가 없다. 하지만 팩토리 패턴(factory pattern)을 이용해서 구현 할 수 있다. 예제에서 다루었던 **New()**함수가 팩토리 패턴을 이용하고 있다. 아래와 같이 수정해서 생성자를 구현했다.
```go
package main

import (
    "fmt"
)

type Rectangle struct {
    Name          string
    Width, Height float64
}

// 팩토리 패턴을 이용 생성자를 구현했다.
func New(name string, width float64, height float64) *Rectangle {
    return &Rectangle{Name: name, Width: width, Height: height}
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
    myRectangle := New("Rect-A", 12.3, 10.9)

    fmt.Println(myRectangle.Area())

    myRectangle.SetWidth(52.2)
    myRectangle.SetHeight(30.3)

    fmt.Println(myRectangle.Area())
}
```
[코드 실행](https://play.golang.org/p/h1DrKbzEUE)

혹은 **Rectangle{Width:11, Height:12}**와 같은 초기화 문법을 이용하면 된다.

## Exports
Go언어는 패키지로 네임스페이스를 구현하고 있다. 그리고 대문자로 시작하는지에 따라서 export 여부가 결정된다. 소문자로 시작할 경우 패키지 안에서만 사용 할 수 있고, 대문자로 사용 할 경우 패키지 바깥에서 사용 할 수 있다.
```go
type Part struct {
    name        string
    description string
    needsSpare  bool
}
```
이제 아래와 같이 export setter과 getter 메서드를 만들 수 있다.
```go
func (p Part)Name string {
    return part.name
}

func (part *Part) SetName(name string) {
    part.name = name
}
```
이렇게 대/소문자만으로 public 메서드와 private(internal 필드 혹은 메서드)를 결정 할 수 있다. 

## 상속(Inheritance)과 composition
객체지향 디자인은 클래스 간의 관계를 구성하는 것에서 시작한다. 클래스간의 관계는 **상속(Inheritance)**과 **컴포지션(composition)** 두 가지 방법으로 구성 할 수 있다. 

상속은 아래 처럼 묘사 할 수 있다.

![OOP에서의 상속](https://docs.google.com/drawings/d/1AZ2UiAJt44aWogpg6th_v3u0C0CeOK7g2Pofi1iWBQ8/pub?w=631&h=420)

이 그림은 **원형 클래스의 특징을 물려 받은 하위 원형 클래스를 만들 겠다**라는 의미를 내포하고 있다. 이때 상위 클래스를 부모 클래스, 하위 클래스를 자식 클래스라고 부른다. 이렇게 **계층적(hierarchy)**으로 클래스 간에 관계를 구성하는 방법은 다른 분야에서도 널리 사용한다. 생물학 시간에 배웠을 **종 > 속 > 과 > 목 > 강 > 문 > 계**가 전형적인 형태다. 

컴포지션은 부모클래스를 호출해서 포함 하는, 즉 **embedding**하는 방식으로 클래스 관계를 구성한다. 다중 상속의 경우 composition 모델은 두 개 이상의 부모 클래스를 embedding 하면 된다. 상속은 **IS - A Relationship**, 컴포지션은 **Has a Relationship** 으로 그 관계를 묘사 할 수 있다. 아래 그림은 composition 모델을 그리고 있다.

![Has a relicationship](https://docs.google.com/drawings/d/1xorVqrumZRT-EqCbW5cdBiD0Lj0yyZkp8sPGh3JeVEw/pub?w=631&h=420)

MountainBike, RoadBike, TandemBike가 Bicycle 스크럭처를 embedding 하는 방식으로 클래스 관계를 구성한 코드다. 
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

이제 Multiple embedding을 이용해서 다중 상속을 구현해 보자.
```go
package main

import (
    "fmt"
)

type Phone struct {
    Model string
}

func (p Phone) Call(num string) {
    fmt.Println("Ring Ring....", num)
}

type Camera struct {
    Model string
}

func (p Camera) TakePicture() {
    fmt.Println("Click ....")
}

type SmartPhone struct {
    Phone
    Camera
}

func main() {
    myPhone := SmartPhone{}
    myPhone.TakePicture()
    myPhone.Call("101-1111-2222")
    fmt.Println("=================")
    yourPhone := SmartPhone{}
    yourPhone.Call("201-2222-1111")

}
```
Camera와 Phone의 기능을 가진(상속받은) SmartPhone 구조체를 만들었다. 그냥 embeded 한 것으로 상속의 주요 기능들을 구현했다. 이제 코드를 약간 수정해서 Phone과 Camera에 모델명을 설정해 보자. 
```go
func main() {
    myPhone := SmartPhone{
        Phone: Phone{Model: "ioph-0001"},
        Camera: Camera{Model: "huca-0002"},
    }
    fmt.Println(myPhone.Model)
}
```
실행하면 "ambiguous selector myPhone.Model"에러가 출력된다. 어느 구조체의 Model을 선택해야 할지 모호(ambiguous)해서 생기는 문제다. 셀렉터(selector)를 설정하면 된다. SmartPhone.Model 까지 추가한 완전한 예제다.
```go
package main

import (
    "fmt"
)

type Phone struct {
    Model string
}

func (p Phone) Call(num string) {
    fmt.Println("Ring Ring....", num)
}

type Camera struct {
    Model string
}

func (p Camera) TakePicture() {
    fmt.Println("Click ....")
}

type SmartPhone struct {
    Model string
    Phone
    Camera
}

func main() {
    myPhone := SmartPhone{
        Model:  "Android-007",
        Phone:  Phone{Model: "ioph-0001"},
        Camera: Camera{Model: "huca-0002"},
    }
    fmt.Println(myPhone.Model)
    fmt.Println(myPhone.Phone.Model)
    fmt.Println(myPhone.Camera.Model)

    yourPhone := SmartPhone{
        Model:  "iphone-8",
        Phone:  Phone{Model: "Lxg-0001"},
        Camera: Camera{Model: "Apppll-0002"},
    }
    fmt.Println(yourPhone.Model)
    fmt.Println(yourPhone.Phone.Model)
    fmt.Println(yourPhone.Camera.Model)
}
```
[코드 실행](https://play.golang.org/p/NJEhR5cFC5)

## 다중 상속과 다이아몬드 문제
다중 상속은 직관적이고 사용하기 편하지만 **죽음의 다이아몬드**라는 골치아픈 문제가 있기 때문에, 별로 권장하지 않는다. 아래 그림을 보자. 

![죽음의 다이아몬드](https://docs.google.com/drawings/d/1p8IlZrB_xPgfdm-ilM_rU_Vn3CHp2MMspNTk-RE-5e4/pub?w=457&h=464)

Animal로 부터 Tiger과 Lion 클래스가 파생됐다. 그리고 다중 상속을 이용해서 Tiger과 Lion으로 부터 파생된 Liger 클래스가 있다. Liger 객체에서 getWeight()를 호출 할 경우 어느 클래스의 getWeight()를 호출해야 할지 모호 하므로 컴파일 실패한다.

Go언어는 selector를 이용해서 네임스페이스를 설정하는 것으로 문제를 피해갈 수 있다. 아래 예제를 보자.
```go
package main

import (
    "fmt"
)

type Animal struct {
}

func (a Animal) GetGene() {
    fmt.Println("동물 유전자")
}

type Tiger struct {
    Animal
}

func (t Tiger) GetGene() {
    fmt.Println("호랑이 유전자")
}

type Lion struct {
    Animal
}

func (l Lion) GetGene() {
    fmt.Println("사자 유전자")
}

type Liger struct {
    Tiger
    Lion
}

func main() {
    fmt.Println("유전자 정보")
    myLiger := Liger{}
    myLiger.Tiger.GetGene()
    myLiger.Lion.GetGene()
    myLiger.Lion.Animal.GetGene()
}
```
[코드 실행](https://play.golang.org/p/tpXfrM8-cw)

셀렉터를 이용 해서 모호함을 없애고 있다. 


## Structs와 Interface
Go의 구조체는 non-virtual 메서드만 가질 수 없다. 가상 메서드(virtual method)를 만들려면 **interface**를 이용해야 한다. Go에서 interface는 오로지  가상 메서드로만 구성 할 수 있다. Go interface의 특징이다.

  * interface는 하나의 타입으로 변수(var)와 매개변수(parameter)로 쓸 수 있다.  
  * interface의 구현은 concret class(struct)에서 이루어진다. concret class는 구상 클래스라고 번역한다.
  * interface는 다른 interface에 상속(embed)할 수 있다.

인터페이스 응용 예제 코드를 만들어 보자.

![인터페이스 예제](https://docs.google.com/drawings/d/1jsBTAaIeUs9jxoa8jOdX0fL3CjIm_8ElmSLJ3eUU3nk/pub?w=378&h=281)

나는 범용 위키애플리케이션을 만들려고 한다. 이 애플리케이션은 MediaWiki의 문법뿐만 아니라 마크다운(MarkDown) 문법도 처리를 할 수 있어야 한다. 필요 할 경우 모니위키(Moniwiki) 등 다른 위키 문법들까지 처리 할 수 있게 만들려고 한다. 플러그인 방식으로 확장을 하게 될테다. 

나는 Wiki interface를 만들고, 위키문서를 HTML로 변환하기 위한 **Parser()**메서드를 등록했다. 개발자는 새로운 위키 파서가 필요할 경우 구조체를 만들고 **Parser()** 메서드만 만들면 된다. 
```go
package main

import "fmt"

// Interface를 만들었다.
type Wiki interface {
    Parser(string) string
}

// MediaWiki 엔진 구현을 위한 구조체
type MediaWiki struct {
    Type string
}

// MediaWiki 파서 실제 구현
func (m MediaWiki) Parser(a string) string {
    return "Moniwiki " + a
}

// 마크다운 엔진 구현을 위한 구조체
type MarkDown struct {
    Type string
}

// 마크다운 파서 실제 구현
func (m MarkDown) Parser(a string) string {
    return "Markdown " + a
}

func main() {
    md := MarkDown{Type: "MarkDown"}
    me := MediaWiki{Type: "MoniWiki"}

    // 위키엔진 저장을 위한 map 자료구조를 만들었다.
    WikiEngine := make(map[string]Wiki, 2)
    WikiEngine["markdown"] = md
    WikiEngine["mediawiki"] = me

    // MarkDown과 MediaWiki는 서로다른 구조체다.
    // 하지만 Wiki interface로 서로 연결이 됐다.
    for _, value := range WikiEngine {
        fmt.Println(value.Parser("Text Data"))
    }
}
```
실제 위키엔진 구현이라면 설정으로 위키 엔진 정보를 읽어온 다음, 맵에 저장할 것이다. 그 후 문서의 타입에 따라서 적당한 Parser()메서드를 호출할 것이다. interface를 이용하면, 같은 이름의 메서드라고 하더라도 다른 구현을 할 수 있다. 즉 객체지향의 **다형성**을 구현할 수 있다.

## empty interface
Go의 interface는 가상 메서드만 가진다고 했다. 가상 메서드도 가지지 않으면 **빈 인터페이스(empty interface: "interface{}")**가 된다. 흔히 빈 인터페이스는 **어떠한 타입이라도 사용 할 수 있다**라고 생각하곤 하는데, 그렇지 않다. 빈 인터페이스에는 인터페이스 타입을 사용해야 한다. 아래 코드를 보자.
```go
func DoSomething(v interface{}) {
    // ...
}
```
DoSomething는 **빈 인터페이스 타입인 v**를 매개변수로 취하고 있다. 여기에 인터페이스 타입의 값을 넘기면, Go언어는 런타임에 형 변환(Type conversion - 언제나 가능한 건 아니다)을 한 후, 정적 타입의 값으로 변경해서 넘긴다. 아래 예제를 보자. 
```go
package main

import (
    "fmt"
)

func PrintAll(vals []interface{}) {
    for _, val := range vals {
        fmt.Println(val)
    }
}

func main() {
    names := []string{"stanley", "david", "oscar"}
    PrintAll(names)
}
```
[코드 실행](https://play.golang.org/p/4DuBoi2hJU)

코드를 실행하려 하면 **main.go:15: cannot use names (type []string) as type []interface {} in argument to PrintAll** 에러가 떨어진다. 타입이 맞지 않기 때문이다. 아래와 같이 타입을 맞춰줘야 한다. 
```go
package main

import (
    "fmt"
)

func PrintAll(vals []interface{}) {
    for _, val := range vals {
        fmt.Println(val)
    }
}

func main() {
    names := []string{"stanley", "david", "oscar"}
    vals := make([]interface{}, len(names))
    for i, v := range names {
        vals[i] = v
    }
    PrintAll(vals)

    age := []int{38, 27, 42}
    vals = make([]interface{}, len(age))
    for i, v := range age {
        vals[i] = v
    }
    PrintAll(vals)
}
```
[코드 실행](https://play.golang.org/p/h7RA5hfMad)

빈 인터페이스를 적절하게 사용하는 메서드로 **fmt.Printf**가 있다. Printf 메서드는 다양한 타입의 값을 매개변수로 받아서 처리해야 하는데 이 때 빈 인터페이스를 사용한다. 
```go
func Printf(format string, a ...interface{}) (n int, err error)
```
빈 인터페이스를 이용해서 하나의 메서드로 여러 데이터를 처리하는 예제 코드다. 
```go
package main

import (
    "fmt"
    "strconv"
)

type Stringer interface {
    String() string
}

func ToString(any interface{}) string {
    if v, ok := any.(Stringer); ok {
        return v.String()
    }

    switch v := any.(type) {
    case int:
        return strconv.Itoa(v)
    case float64:
        return strconv.FormatFloat(v, 'f', 4, 64)
    }
    return "???"
}

type User struct {
    Name string
}

func (u User) String() string {
    return u.Name
}

func main() {
    fmt.Println(ToString(1234))
    fmt.Println(ToString(17.4))
    fmt.Println(ToString("Hello World"))
    fmt.Println(ToString(User{"yundream"}))
}
```
[코드 실행](https://play.golang.org/p/dlzNKsm00V)

매개변수로 넘어온 (빈 인터페이스 타입의)데이터를 연산하기 위해서는, 데이터의 타입을 알아야 한다. **reflect.TypeOf()**혹은 **type assertions**를 이용해서 데이터 타입을 알 수 있다. 예제 코드에서는 type assertion(any.(type))을 이용하고 있다.

## 참고
  * [golang oop](https://github.com/luciotato/golang-notes/blob/master/OOP.md)
  * [How to use interfaces in go](http://jordanorelli.com/post/32665860244/how-to-use-interfaces-in-go)
