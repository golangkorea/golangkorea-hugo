+++

title = "Go 둘러보기 - encoding/json 패키지"
draft = false
date = "2016-12-03T01:02:26+09:00"

tags = ["Golang", "Encoding"]
categories = ["번역", "Encoding"]
series = ["Go  Walkthrough"]
authors = ["mingrammer"]

toc = true

+++

> [Go Walkthrough](https://medium.com/go-walkthrough) 시리즈의 [Go Walkthrough: encoding/json package](https://medium.com/go-walkthrough/go-walkthrough-encoding-json-package-9681d1d37a8f#.eszbi1cjw)를 번역한 글입니다.

좋든 나쁘든, JSON은 인터넷의 인코딩이다. 이것의 공식적인 정의는 냅킨 뒷면에 쓸 수 있을 정도로 간단하지만 이는 문자열, 숫자, 부울, 널(nulls), 맵(maps) 그리고 배열을 인코딩 할 수 있다. 이러한 간결함 덕에, 모든 언어는 JSON 파서를 가지고 있다. 

Go에서의 구현체는 [encoding/json](https://golang.org/pkg/encoding/json/)이라고 하는 패키지이며 이는 Go 객체에 대한 JSON 인코딩을 원활하게 추가할 수 있도록 해준다. 그러나, 광범위하게 리플렉션을 사용함으로써, *encoding/json* 은 가장 많이 사용되는 패키지중 하나임에도 불구하고 이해하기 어려운 패키지중 하나이다. 우리는 이 패키지가 어떻게 동작하는지에 대해 자세히 살펴볼 볼 것이다. 패키지의 사용법뿐만 아니라 내부 함수들이 어떻게 동작하는지도 살펴볼 것이다.

이 포스트는 표준 라이브러리를 이해하는데 도움을 주기위한 Go 둘러보기 시리즈의 일부이다. 기존에 생성된 문서(자동으로 생성된 Go 문서)는 많은 정보를 제공하지만, 이는 패키지를 실제 상황에서 이해하기에는 어려울 수 있다. 이 시리즈는 일상적으로 사용되는 애플리케이션에서 표준 패키지들이 어떻게 사용되는지에 대한 컨텍스트를 제공할 수 있도록 도와준다. 질문이나 코멘트가 있다면 트위터에서 [@benbjohnson](https://twitter.com/benbjohnson)로 찾아오면 된다.

<br>

## JSON이란 무엇인가?

JSON은 *JavaScript Object Notation* 의 약자로 객체 리터럴을 정의하는 자바스크립트의 하위 집합이다. 자바스크립트는 정적 선언 타이핑이 없기 때문에 언어 리터럴은 암시적 타입을 가져야 한다. 문자열은 쌍 따옴표로 감싸고, 배열은 괄호로 감싸며, 맵은 중괄호로 감싼다.

```javascript
{"name": "mary", "friends":  ["stu", "becky"], age: 30}
```

이 느슨한 타입 정보들은 자바스크립트 개발자들에겐 저주지만, 이는 데이터를 매우 쉽고 간결하게 표현하는 방법을 제공한다.

<br>

## JSON 사용의 트레이드 오프

JSON은 사용하기는 쉽지만, 몇가지 문제가 발생할 수 있다. 사람이 쉽게 읽을 수 있는 포맷들은 일반적으로 컴퓨터가 파싱하기에는 느리다. 예를 들면, 내 맥북 프로에서 *encoding/json* 를 벤치마킹하면 인코딩과 디코딩 속도가 각각 100 MB/sec과 27 MB/sec가 나온다.

```
$ go test -bench=. encoding/json
BenchmarkCodeEncoder-4      106.26 MB/s
BenchmarkCodeDecoder-4      27.76 MB/s
```

그러나 일반적으로 바이너리 디코더는 데이터를 몇 배 더 빠르게 파싱할 수 있다. 이 문제는 데이터를 읽는 방식 때문에 발생한다. "123.45"와 같은 JSON 숫자 리터럴은 두 가지 반복적인 단계로 디코딩 되어야한다.

1. 각 바이트를 읽어 숫자인지 닷(dot, .)인지를 검사한다. 만약 숫자가 아닌 데이터를 읽으면 숫자 리터럴 스캐닝을 멈춘다.
2. 10진수 숫자 리터럴을 *int64* 나 IEEE-754 부동 소수점 숫자 표현식과 같은 2진수 포맷으로 변환한다.

이는 들어오는 모든 바이트에 대한 많은 파싱들뿐만 아니라 디코더상의 미리보기 버퍼를 포함한다. 이와는 대조적으로, 바이너리 디코더는 단순히 얼마나 많은 바이트를 파싱해야 하는지(예를 들어 2,4, 또는 8)와 플립할 수 있는 엔디안만 알면된다. 이 바이너리 파싱 연산은 또한 [CPU 파이프라이닝(pipelining)](https://en.wikipedia.org/wiki/Instruction_pipelining)을 늦추는 [분기(branching)](https://en.wikipedia.org/wiki/Branch_%28computer_science%29)가 필요하지 않다.

<br>

## 언제 JSON을 사용해야 하는가?

일반적으로 JSON은 손쉬운 데이터 교환이 가장 큰 목적이고 성능이 크게 중요하지 않을 경우 사용된다. JSON은 사람이 읽기 쉽기 때문에, 뭔가 잘못될 경우 디버깅 하기가 쉽다. 반면에, 바이너리 프로토콜은 분석되기 전에 우선 디코딩 되어야한다.

많은 애플리케이션에서 인코딩/디코딩 성능은 쉽게 수평 확장이 가능하기 때문에 낮은 우선순위를 가진다. 예를 들어, API 엔드포인트를 제공하기 위해 추가적인 서버를 증설하는건 쉽다. 왜냐하면 인코딩은 서로 다른 서버들간의 조정 등이 필요없기 때문이다. 그러나 데이터베이스는 서버를 추가해야할 때 스케일링 하기가 쉽지 않다.

<br>

## 스트림 인코딩

json 패키지에는 값을 JSON으로 인코딩하는 두 가지 방법이 있다. 첫번째는 값을 io.[Writer](https://golang.org/pkg/io/)로 인코딩하는 스트림 기반 json.[Encoder](https://golang.org/pkg/encoding/json/#Encoder)이다:

```go
type Encoder struct {}
func NewEncoder(w io.Writer) *Encoder
func (enc *Encoder) Encode(v interface{}) error
```
> io : [io](https://golang.org/pkg/io/), Writer : [Writer](https://golang.org/pkg/io/), Encode : [Encoder](https://golang.org/pkg/encoding/json/#Encoder), error : [error](https://golang.org/pkg/builtin/#error)

두번째 옵션은 인코딩된 값을 인메모리 바이트 슬라이스로 반환하는 json.[Marshal](https://golang.org/pkg/encoding/json/#Marshal)()이다:

```go
func Marshal(v interface{}) ([]byte, error)
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error: [error](https://golang.org/pkg/builtin/#error)

이 인코더들에 값을 전달하면, JSON 라이브러리는 타입 정의 검사, 인코더 컴파일 그리고 데이터값 재귀 처리의 복잡한 프로세스를 실행한다. 이제 각각에 대해 자세히 알아보자.

### 타입 검사

인코더에 값을 전달하면 가장 먼저 값의 타입 인코더를 검색한다. 타입들은 Go의 [reflect](https://golang.org/pkg/reflect/) 패키지에 의해 검사되며 [json](https://golang.org/pkg/encoding/json/) 패키지는 이 reflect.[Type](https://golang.org/pkg/reflect/#Type) 값들에 대한 내부 매핑을 가지고 있다. [json](https://golang.org/pkg/encoding/json/) 패키지에는 int, string, map, struct 그리고 slice와 같은 내장 타입들에 대한 하드코딩된 구현체들이 있다. 이들은 정말 단순하다 - *stringEncoder* 는 문자열값을 쌍 따옴표로 감싸며 필요한 경우 문자들을 이스케이프하고, *intEncoder* 는 정수를 문자열 포맷으로 변환하고, 등등.

### 인코더 컴파일

내장 타입이 아닌 타입들에 대해선, 인코더가 즉시 생성되고 재사용을 위해 캐싱된다. 우선, 인코더는 해당 타입이 json.[Marshaler](https://golang.org/pkg/encoding/json/#Marshaler)를 구현하고 있는지 확인한다:

```go
type Marshaler interface {
        MarshalJSON() ([]byte, error)
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error: [error](https://golang.org/pkg/builtin/#error)

만약 구현하고 있다면 마샬링은 타입에 따라 결정된다. 이는 타입중 하나가 [json](https://golang.org/pkg/encoding/json/) 패키지의 리플렉션 기반 인코더로 처리할 수 없는 특별한 JSON 표현식을 가진 경우 매우 유용하다.  

다음으로, 인코더는 타입이 encoding.[TextMarshaler](https://golang.org/pkg/encoding/#TextMarshaler)을 구현하고 있는지 확인한다:

```go
type TextMarshaler interface {
        MarshalText() (text []byte, err error)
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error: [error](https://golang.org/pkg/builtin/#error)

만약 구현하고 있다면 이는 해당 함수로부터 값을 생성하고 결과값을 JSON 문자열로 인코딩 할 것이다. 이는 time.[Time](https://golang.org/pkg/time/#Time)을 사용할 때 항상 볼 수 있다. 왜냐하면 time.[Time](https://golang.org/pkg/time/#Time)은 [MarshalText](https://golang.org/pkg/time/#Time.MarshalText)() 메서드를 가지고 있기 때문이며, JSON 인코더는 time.[Time](https://golang.org/pkg/time/#Time) 값을 [RFC 3339](https://golang.org/pkg/time/#RFC3339) 포맷 문자열로 인코딩 할 것이다.

마지막으로, 두 인터페이스 모두 구현되어 있지 않을 경우엔 기본 인코더를 기반으로 재귀적으로 인코더를 생성한다. 예를 들어, *int* 필드와 *string* 필드를 가지는 *struct* 로 이루어진 타입은 *intEncoder* 와 *stringEncoder* 를 갖는 *structEncoder* 를 생성할 것이다. 다시 말하지만, 인코더 생성은 딱 한 번만 이루어지며 만들어진 인코더는 차후의 사용을 위해 캐싱될 것이다.

### 필드별 옵션

구조체 인코더에 대한 한가지 중요한 점은 이는 인코딩을 위한 필드별 옵션을 결정하기 위해  필드 태그를 읽는다는 것이다. 태그는 구조체의 끝에서 가끔 볼 수 있는 역 따옴표(`)로 감싸진 문자열이다.

예시:

```go
type User struct {
        Name    string `json:"name"`
        Age     int    `json:"age,omitempty"`
        Zipcode int    `json:"zipcode,string"`
}
```

이 옵션은 다음을 포함한다:

* 필드 키 이름을 바꾼다. 많은 JSON 키는 카멜케이스이므로 이에 일치하도록 이름을 바꾸는것은 중요할 수 있다.
* *omitempty* 플래그는 빈 값을 갖는 비구조체 필드들을 하도록 설정할 수 있다.
* *string* 플래그는 필드가 문자열로 인코딩 되도록 강제하는데 사용될 수 있다. 예를 들면, 정수가 문자열로 인코딩 되도록 강제할 수 있다. 

### 재귀 처리

마지막으로, 인코딩이 수행될 때 이는 *encodeState* 라고 하는 내장 버퍼에 기록된다. 이 객체는 값이 필요로하는 각 인코더로 전달되어 인코더가 바이트를 추가할 수 있도록 한다. json.[Marshal](https://golang.org/pkg/encoding/json/#Marshal)이 호출되면, 이 버퍼의 바이트에 대한 참조가 반환된다.

json.[Encoder](https://golang.org/pkg/encoding/json/#Encoder)를 사용할 때, *encodeState* 버퍼를 재사용하기 위해 내부적으로 sync.[Pool](https://golang.org/pkg/sync/#Pool)가 사용된다. 이는 인코더가 필요로 하는 힙 메모리 할당 횟수를 최소화하므로 스트림 처리는 항상 json.[Encoder](https://golang.org/pkg/encoding/json/#Encoder)를 사용한다.

<br>

## 스트림 디코딩

JSON으로 인코딩된 바이트를 다시 객체로 변환하는 것은 인코딩 프로세스의 역과 비슷하지만 중요한 차이점이 있다.

바이트에서 JSON을 디코딩하는 방법은 두 가지가 있다. 첫번째는 io.[Reader](https://golang.org/pkg/io/#Reader)로부터 디코딩 할 수 있는 스트림 기반 json.[Decoder](https://golang.org/pkg/io/#Reader)이다:

```go
type Decoder strcut {}
func NewDecoder(r io.Reader) *Decoder
func (dec *Decoder) Decode(v interface{}) error
```
> io : [io](https://golang.org/pkg/io/), Reader : [Reader](https://golang.org/pkg/io/#Reader), Decoder : [Decoder](https://golang.org/pkg/io/#Reader), error : [error](https://golang.org/pkg/builtin/#error) 

또는 json.[Unmarshal]() 함수를 사용해 바이트 슬라이스로부터 디코딩 할 수 있다:

```go
func Unmarshal(data []byte, v interface{}) error
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error : [error](https://golang.org/pkg/builtin/#error)

이 디코더들은 두 부분으로 동작한다. 우선 *scanner* 가 입력된 바이트를 토큰화하고 *decodeState* 가 토큰들을 Go 객체로 변환한다.

### JSON 스캐닝

*scanner* 는 JSON을 파싱하는데 쓰이는 내부 상태 머신(state machine)이다. 이는 여러 단계로 동작한다. 첫번째로, 이는 파싱을 위한 토큰의 타입을 결정하기 위해 값의 첫번째 바이트를 검사한다. 만약 그게 "{"라면 객체를 파싱해야하고, "["라면 배열을 파싱해야한다. 이는 단순한 값에도 똑같이 적용된다. 쌍 따옴표는 문자열의 시작점을 나타내고, *"t"* 나 *"f"* 는 부울값의 시작을 나타내며, *0-9* 는 숫자의 시작을 나타낸다.

스캐닝 타입 결정이 끝나면, 이는 타입별 함수 (문자열 스캔, 숫자 스캔 등)로 전달된다. 맵이나 배열같은 복잡한 객체들에 대해선, 닫는 중괄호를 추적하는데 스택이 사용된다.

### 버퍼 미리보기

스캐닝의 흥미로운 부분은 버퍼 미리보기이다. JSON은 "LL(1)으로 파싱가능"하며 이는 스캐닝하는데 딱 하나의 바이트 버퍼만 필요하다는 의미이다. 이 버퍼는 다음 바이트를 미리 보는데 사용된다. 

예를 들면, 숫자 스캐닝 함수는 숫자가 아닌 문자를 찾을 때까지 바이트를 계속 읽을 것이다. 그러나, 스트림으로부터 문자를 이미 읽었기 때문에 다른 스캐닝 함수가 사용할 수 있도록 이를 버퍼에서 빼줘야 한다. 이게 바로 버퍼 미리보기가 필요한 이유이다.

*파서 작성에 관심이 있다면, 내가 Gopher Academy에 쓴 [Handwriting Parsers & Lexers in Go](https://blog.gopheracademy.com/advent-2014/parsers-lexers/)를 보라*

### 토큰 디코딩

토큰이 스캔되면 이제 해석해야한다. 이는 *decodeState* 의 일이다. 이 단계에서 디코딩될 입력값들은 처리될 각 토큰과 일치한다.

예를 들면, 만약 구조체 타입을 전달하면 디코더는 *"{"* 값을 기대할 것이다. 다른 토큰들이 들어오면 디코딩은 에러를 반환할 것이다. 토큰들을 값들과 일치시키는 이 단계는 [reflect](https://golang.org/pkg/reflect/) 패키지를 많이 사용하지만 디코더는 이를 캐싱하지 않으므로 매 디코딩마다 리플렉션이 이루어진다.

여러분은 또한 Decoder.[Token](https://golang.org/pkg/encoding/json/#Decoder.Token)()과 Decoder.[More](https://golang.org/pkg/encoding/json/#Decoder.More)() 메서드를 사용해 토큰들을 스트림으로 처리할 수도 있다. 나는 이 메서드들을 사용해본 적은 없지만, 이런것들을 사용할 수 있다는걸 알아두면 좋다.

### 커스텀 언마샬링(Unmarshaling)

인코딩과 마찬가지로, 디코딩 커스텀 구현체도 만들 수 있다. 디코더는 먼저 타입이 json.[Unmarshaler](https://golang.org/pkg/encoding/json/#Unmarshaler) 를 구현하고 있는지를 검사한다:

```go
type Unmarshaler interface {
        UnmarshalJSON([]byte) error
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error : [error](https://golang.org/pkg/builtin/#error)

이는 타입이 한 타입에 대한 JSON 값 전체를 받도록하며 타입 자체를 파싱할 수 있다. 이는 자체적으로 최적화 구현체를 구현하고싶을때 유용하다.

다음으로 디코더는 타입이 encoding.[TextUnmarshaler](https://golang.org/pkg/encoding/#TextUnmarshaler)를 구현하고 있는지를 검사한다:

```go
type TextUnmarshaler interface {
        UnmarshalText(text []byte) error
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error : [error](https://golang.org/pkg/builtin/#error)

이는 사용하고 싶은 타입의 문자열 표현을 가지고 있을때 유용하다. 이의 한 예시는 내부적으로 정수로 표현되는 enum 타입을 문자열로써 인코딩/디코딩하는 경우이다.

### 지연 처리

json.[Unmarshaler](https://golang.org/pkg/encoding/json/#Unmarshaler)의 대체제는 json.[RawMessage] 타입이다. [RawMessage](https://golang.org/pkg/encoding/json/#RawMessage)를 사용하면, 원본 JSON 표현이 언마샬링이 완료된 후에 처리할 수 있는 필드에 저장된다. 이는 JSON 객체의 "type" 필드를 해석하고 값을 기반으로 JSON 파싱을 변경해야 할 때 유용하다.

```go
type T struct {
        Type  string          `json:"type"`
        Value json.RawMessage `json:"value"`
}

func (t *T) Val() (interface{}, error) {
        switch t.Type {
        case "foo":
                // "t.Value"를 Foo로 파싱
        case "bar":
                // "t.Value"를 Bar로 파싱
        default:
                return nil, errors.New("invalid type")
        }
}
```

나는 개인적으로 추후 해석을 위해 JSON을 저장해두는것을 좋아하지 않기 때문에 json.[Unmarshaler](https://golang.org/pkg/encoding/json/#Unmarshaler)가 좀 더 유용하다고 생각한다.

또 다른 지연 처리 방법은 JSON 숫자를 사용하는것이다. 왜냐하면 JSON은 정수와 실수를 구분하지 못하기 때문에 디코더는 *interface{}* 필드로 숫자를 디코딩 할 시  *float64* 로 변환한다. 파싱을 지연 시키기위해 json.[Number](https://golang.org/pkg/encoding/json/#Number) 타입을 대신 사용할 수 있다.

```go
type T struct {
        Value json.Number
}

...

if strings.Contains(t.Value, ".") {
        v, err := t.Value.Float64()
        // 실수로 처리
} else {
        v, err := t.Value.Int64()
        // 정수로 처리
}
```

나는 디코딩시 주로 정적 타입을 사용하기 때문에 json.[Number](https://golang.org/pkg/encoding/json/#Number)를 잘 사용하지 않는다.

<br>

## 깔끔한 출력

JSON은 일반적으로 추가 공백 없이 하나의 긴 바이트로 쓰이지만, 이는 읽기가 어렵다. 여러분은 두 가지 방법으로 들여쓰기를 설정할 수 있다. JSON으로 인코딩된 인메모리 바이트 슬라이스의 경우엔 이를 json.[Indent]()() 함수에 전달할 수 있다:

```go
func Indent(dst *bytes.Buffer, src []byte, prefix, indent string) error
```
> bytes : [bytes](https://golang.org/pkg/bytes/), Buffer : [Buffer](https://golang.org/pkg/bytes/#Buffer), byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string), error : [error](https://golang.org/pkg/builtin/#error)

*prefix* 인자는 모든 라인에 쓸 문자를 지정하고 *inednt* 는 들여쓰기에 사용는 문자를 지정한다. 나는 *prefix* 는 많이 사용하지 않지만 *indent* 값으로는 보통 2-스페이스  또는 탭을 사용한다.

json.[Marshal](https://golang.org/pkg/encoding/json/#Marshal)()를 호출한 다음 json.[Indent](https://golang.org/pkg/encoding/json/#Indent)를 호출해주는 json.[MarshalIndent](https://golang.org/pkg/encoding/json/#MarshalIndent)()라는 헬퍼 함수가 있다.

만약 스트림 기반의 json.[Encoder](https://golang.org/pkg/encoding/json/#Encoder.SetIndent)를 사용하고 있다면 [SetIndent](https://golang.org/pkg/encoding/json/#Encoder.SetIndent)() 메서드를 사용해 들여쓰기를 할 수 있다:

```go
func (enc *Encoder) SetIndent(prefix, indent string)
```
> Encoder : [Encoder](https://golang.org/pkg/encoding/json/#Encoder.SetIndent), string : [string](https://golang.org/pkg/builtin/#string)

많은 사람들이 [SetIndent](https://golang.org/pkg/encoding/json/#Encoder.SetIndent)()에 대해 모르고 바이트 슬라이스를 마샬링하고 들여쓴 후 그 결과를 스트림에 쓴다.

들여쓰기 함수의 반대는 [Compact](https://golang.org/pkg/encoding/json/#Compact)() 함수이다:

```go
func Compact(dst *bytes.Buffer, src []byte) error
```
> bytes : [bytes](https://golang.org/pkg/bytes/), Buffer : [Buffer](https://golang.org/pkg/bytes/#Buffer), byte : [byte](https://golang.org/pkg/builtin/#byte), error : [error](https://golang.org/pkg/builtin/#error)

이는 *src* 를 대상 버퍼로 재작성하지만 모든 공백을 지운다.

<br>

## 인코딩/디코딩시 에러 핸들링

json 패키지는 상당수의 에러 타입을 가지고 있다. 아래에 인코딩 또는 디코딩시 마주할 수 있는 에러 리스트가 있다:

* 디코딩을 하기 위해 포인터가 아닌 값을 전달하여 실제로는 값의 복사본을 전달하게되면 디코더는 원래값에 디코딩을 할 수 없다. 디코더는 이를 잡아내고 [InvalidUnmarshalError](https://golang.org/pkg/encoding/json/#InvalidUnmarshalError)를 반환한다.
* 만약 데이터가 잘못된 JSON 값을 포함하고 있으면 잘못된 문자의 바이트 위치와 함께 [SyntaxError](https://golang.org/pkg/encoding/json/#SyntaxError)가 반환된다.
* 만약 에러가 json.[Marshaler](https://golang.org/pkg/encoding/json/#Marshaler)나 encoding.[TextMarshaler](https://golang.org/pkg/encoding/#TextMarshaler)에 의해 반환되면 이는 [MarshalerError](https://golang.org/pkg/encoding/json/#MarshalerError)로 래핑된다.
* 만약 토큰이 대응하는 값으로 언마샬링 될 수 없는 경우 [UnmarshalTypeError](https://golang.org/pkg/encoding/json/#UnmarshalTypeError)가 반환된다.
* *Infinity* 와 *NaN* 의 실수값은 JSON으로 표현할 수 없으며 [UnsupportedValueError](https://golang.org/pkg/encoding/json/#UnsupportedValueError)가 반환된다.
* JSON으로 표현할 수 없는 타입들(예를 들어, 함수, 복소수, 포인터 등등)의 경우 [UnsupportedTypeError](https://golang.org/pkg/encoding/json/#UnsupportedTypeError)가 반환된다.
* Go 1.2 이전에서 잘못된 UTF-8 문자는 [InvalidUTF8Error]()에러를 반환한다. 이후 버전은 단순히 잘못된 문자를 "알 수 없는 문자"를 의미하는 유니코드 문자인 [U+FFFD](http://unicode.org/cldr/utility/character.jsp?a=FFFD)로 변환한다.

에러가 많은 것처럼 보일 수 있지만, 에러를 로깅하고 사람이 직접 개입해서 조작하는것 이외에 코드에서 처리할 수 있는것은 많지 않다. 또한, 이들중 대부분은 유닛 테스트 커버리지가 있다면 개발 도중 잡아낼 수 있다.

<br>

## 대체 구현

몇 년 전 나는 리플렉션을 완전히 피하기 위해 컴파일 시 특정 타입별 인코더와 디코더를 생성해주는 [megajson](https://github.com/benbjohnson/megajson)라는 툴을 개발했었다. 이는 인코딩과 디코딩을 훨씬 빠르게 만들어주었다. 그러나, 이 툴은 개념 증명이었으며 지원의 한계가 있어 결국 버려졌다.

운좋게도, [Paul Querna](http://paul.querna.org/)이 동일한 일을 하지만 훨씬 나은 [ffjson](https://github.com/pquerna/ffjson)이라는 구현체를 만들었다. JSON 인코딩과 디코딩 성능을 향상시키고자 한다면 이 툴을 강력히 추천한다. 

<br>

## 결론

JSON은 빠르게 실행해야하거나 유저에게 간단한 API를 제공해야 할 때 훌륭한 데이터 포맷이 될 수 있다. Go의 구현체는 리플렉션을 사용하여 간단하게 사용할 수 있는 많은 기능들을 제공한다.

우리는 JSON 표현을 포맷팅 하는 방법 뿐만 아니라 JSON의 인코딩과 디코딩 측면의 내부를 살펴보았다. 이 툴들은 밖에선 간단해 보일 수 있지만 내부적으로는 최대한 빠르고 효율적으로 만들기 위해 많은 일들이 일어나고 있다.
