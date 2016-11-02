+++

title = "Go 둘러보기 - bytes + strings 패키지"
draft = false
date = "2016-11-02T18:30:33+09:00"

tags = ["Golang", "Bytes", "Strings", "Package"]
categories = ["번역", "Bytes", "Strings"]
series = ["Go  Walkthrough"]
authors = ["mingrammer"]

toc = true

+++


> [Go Walkthrough](https://medium.com/go-walkthrough) 시리즈의 [Go Walkthrough: bytes + strings packages](https://medium.com/go-walkthrough/go-walkthrough-bytes-strings-packages-499be9f4b5bd#.4jbor0c85)를 번역한 글입니다.

우린 [지난번 포스트](https://medium.com/go-walkthrough/go-walkthrough-io-package-8ac5e95a9fbd#.vjz46d3ss)에서 바이트 스트림을 다뤄봤는데 가끔은 제한적인 범위에서, 인메모리 바이트 슬라이스를 가지고 작업해야 할 수도 있다. 바이트의 리스트로 작업 하는 것은 충분히 간단해 보일 수 있지만, [bytes](https://golang.org/pkg/bytes/) 패키지를 유용하게 활용할 수 있는 많은 엣지 케이스와 일반적인 연산들이 있다. 우리는 또한 [strings](https://golang.org/pkg/strings/) 패키지는 문자열 작업을 위한것임에도 불구하고 이 API와 거의 동일하기 때문에 이 포스트에서는 이 [strings]([strings](https://golang.org/pkg/strings/)) 패키지에 대해 파헤쳐 볼 것이다.

이 포스트는 표준 라이브러리를  이해하는데 도움을 주기위한 Go 둘러보기 시리즈의 일부이다. 기존에 생성된 문서(
자동으로 생성된 Go 문서)는 많은 정보를 제공하지만, 이는 패키지를 실제 상황에서 이해하기에는 어려울 수 있다.
이 시리즈는 일상적으로 사용되는 애플리케이션에서 표준 패키지들이 어떻게 사용되는지에 대한 컨텍스트를 제공할
수 있도록 도와준다. 질문이나 코멘트가 있다면 트위터에서 [@benbjohnson](https://twitter.com/benbjohnson)로 찾
아오면 된다.

<br>

## 바이트와 스트링의 간단한 비교

Rob Pike는 [strings, bytes, runes, and characters](https://blog.golang.org/strings)에 대한 훌륭한 포스트를 가지고 있지만 이 포스트에서는 애플리케이션 개발자의 관점에서 좀 더 간결한 정의를 제공하려고 한다.

바이트 슬라이스는 변형 가능하고, 크기 조절이 가능한 연속적인 바이트 리스트를 나타낸다. 그럼 이제 이게 무슨 뜻인지를 알아보자.

바이트 슬라이스가 하나 주어졌다:

```go
buf := []byte{1,2,3,4}
```
이는 변형 가능하므로 원소 갱신이 가능하다:

```go
buf[3] = 5  // []byte{1,2,3,5}
```
이는 크기 조절이 가능하므로 축소 또는 확장이 가능하다:

```go
buf = buf[:2]           // []byte{1,2}           
buf = append(buf, 100)  // []byte{1,2,100}
```

그리고 이는 연속적이므로 메모리상에서 각 바이트는 다른 바이트의 바로 뒤에 위치한다:

```go
1|2|3|4
```

반면에, 문자열은 변형 불가능하고, 고정된 크기의 연속적인 바이트 리스트를 나타낸다. 이는 문자열은 갱신이 불가능함을 의미한다 - 새로운 문자열을 생성하는 수 밖에 없다. 이는 성능 관점에서 중요하다. 고성능 코드에서, 계속해서 새로운 문자열을 생성하면 가비지 컬렉터에 많은 부하가 추가된다.

애플리케이션 개발 관점에서, 문자열은 UTF-8 데이터로 작업 할 때 사용하기 쉬우며 바이트 슬라이스를 사용할 수 없는 맵(map)의 키값으로도 쓰일 수 있다. 그리고 대다수의 API는 문자 데이터를 포함하는 인자에 문자열을 사용한다. 반면에, 바이트 슬라이스는 바이트 스트림을 처리하는 등의 로우(raw) 바이트를 다룰 때 사용하기 좋다. 이는 또한 새로 할당을 하지 않고 데이터를 재사용 해야할 경우에 유용하다.

<br>

## 스트림을 위한 문자열&슬라이스 개조하기

바이트와 문자열 패키지의 가장 중요한 특징중 하나는 io.[Reader](https://golang.org/pkg/io/#Reader)와 io.[Writer](https://golang.org/pkg/io/#Writer)로 인메모리 바이트 슬라이스와 문자열을 인터페이싱할 수 있는 방법을 제공한다는 것이다.

### 인메모리 Reader

Go 표준 라이브러리에서 가장 많이 사용되는 툴 두 가지는 bytes.[NewReader](https://golang.org/pkg/bytes/#NewReader)와 strings.[NewReader](https://golang.org/pkg/strings/#NewReader) 함수이다:

```go
func NewReader(b []byte) *Reader
func NewREader(s string) *Reader
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), (byte)Reader : [Reader](https://golang.org/pkg/bytes/#Reader), string : [string](https://golang.org/pkg/builtin/#string), (string)Reader : [Reader](https://golang.org/pkg/strings/#Reader)

이 함수들은 인메모리 바이트 슬라이스 또는 문자열을 래핑하는 io.[Reader](https://golang.org/pkg/io/#Reader) 구현체를 반환한다. 그러나 이들은 단순한 Reader가 아니다. 이들은 [io](https://golang.org/pkg/io/)의 io.[ReaderAt](https://golang.org/pkg/io/#ReaderAt), io.[WriterTo](https://golang.org/pkg/io/#WriterTo), io.[ByteReader](https://golang.org/pkg/io/#ByteReader), io.[ByteScanner](https://golang.org/pkg/io/#ByteScanner), io.[RuneReader](https://golang.org/pkg/io/#RuneReader), io.[RuneScanner](https://golang.org/pkg/io/#RuneScanner), & io.[Seeker](https://golang.org/pkg/io/#Seeker)를 포함하는 읽기와 관련된 모든 인터페이스를 구현하고 있다.

나는 바이트 슬라이스 또는 문자열이 bytes.[Buffer](https://golang.org/pkg/bytes/#Buffer)에 쓰여지고 이 버퍼가 Reader로써 사용되는 코드를 자주 본다:

```go
var buf bytes.Buffer
buf.WriteString("foo")
http.Post("http://example.com/", "text/plain", &buf)
```

그러나, 이 방법은 힙 메모리 할당을 발생시켜 느려지고 추가적인 메모리를 사용하게 된다. 더 좋은 방안은 strings.[Reader](https://golang.org/pkg/strings/#Reader)을 사용하는 것이다:

```go
r := strings.NewReader("foobar")
http.Post("http://example.com", "text/plain", r)
```

이 방법은 io.MultiReader를 사용함으로써 여러개의 문자열 또는 바이트 슬라이스를 가지고 있을 때에도 잘 동작한다:

```go
r := io.MultiReader(
        strings.NewReader("HEADER"),
        bytes.NewReader([]byte{0,1,2,3,4}),
        myFile,
        strings.NewReader("FOOBAR"),
)
```

### 인메모리 Writer

바이트 패키지는 [Buffer](https://golang.org/pkg/bytes/#Buffer)라고 부르는 io.[Writer](https://golang.org/pkg/io/#Writer)의 인메모리 구현체를 포함하고 있다. 이는 io.[Closer](https://golang.org/pkg/io/#Closer)와 io.[Seeker](https://golang.org/pkg/io/#Seeker)를 제외한 [io](https://golang.org/pkg/io/) 인터페이스의 거의 모든 것을 구현하고 있다. 여기엔 버퍼의 끝에 문자열을 쓰기위한 [WriteString](https://golang.org/pkg/bytes/#Buffer.WriteString)()이라는 헬퍼 메서드도 있다.

나는 유닛 테스트에서 서비스에서 나오는 로그들을 캡쳐하기위해 [Buffer](https://golang.org/pkg/bytes/#Buffer)를 광범위하게 사용한다. 여러분은 이를 log.[New](https://golang.org/pkg/log/#New)()의 인자로써 전달할 수 있으며 나중에 출력값들을 검증할 수 있다:

```go
var buf bytes.Buffer
myService.Logger = log.New(&buf, "", log.LstdFlags)
myService.Run()

if !strings.Contains(buf.String(), "service failed") {
        t.Fatal("expected log message")
}
```
> Buffer : [Buffer](https://golang.org/pkg/bytes/#Buffer), LstdFlags : [LstdFlags](https://golang.org/pkg/log/#LstdFlags), Contains : [Contains](https://golang.org/pkg/strings/#Contains), String : [String](https://golang.org/pkg/bytes/#Buffer.String), Fatal : [Fatal](https://golang.org/pkg/testing/#T.Fatal)

그러나, 나는 프로덕션 코드에서는 [Buffer](https://golang.org/pkg/bytes/#Buffer)를 거의 사용하지 않는다. 이름이 Buffer임에도 불구하고, 해당 목적에 좀 더 특화된 [bufio](https://golang.org/pkg/bufio/) 패키지가 있기 때문에 Buffer를 버퍼 읽기와 쓰기에 사용하지 않는다.

<br>

## 패키지 구조

바이트와 문자열 패키지의 첫인상은 큰 패키지처럼 보이지만 이는 단지 그냥 간단한 헬퍼 함수들의 컬렉션이다. 우리는 이들을 몇 개의 카테고리로 그룹핑 할 수 있다.

* 비교 함수 (Comparison functions)
* 검사 함수 (Inspection functions)
* 접두/접미 함수 (Prefix/suffix functions)
* 교체 함수 (Replacement functions)
* 분할 & 결합 함수 (Splitting & joining functions)

함수들이 어떻게 함께 그룹핑 되는지를 이해한다면, 대형 API에 접근하기가 훨씬 쉬워질 것이다.

<br>

## 비교 함수

두 개의 바이트 슬라이스 또는 문자열을 가지고 있을 때 다음 두 질문중 하나를 질문해야 할 수도 있다. 첫째, 두 객체가 동일한가? 둘째, 정렬시 어떤게 앞쪽에 위치하는가?

### 동일성

[Equal](https://golang.org/pkg/bytes/#Equal)() 함수는 첫번째 질문에 대한 답을 준다:

```go
func Equal(a, b []byte) bool
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), bool : [bool](https://golang.org/pkg/builtin/#bool)

이 함수는 바이트 패키지에만 존재하는데 문자열은 == 연산자로 비교가 가능하기 때문이다.

동일함을 확인하는건 쉬워 보일 수 있지만, 한 가지 일반적으로 나타나는 실수는 대소문자에 무관하게 동일함을 확인하기 위해 strings.[ToUpper](https://golang.org/pkg/strings/#ToUpper)()를 사용하는 것이다.

```go
if strings.ToUpper(a) == strings.ToUpper(b) {
    return True
}
```

이는 새로운 문자열에 대한 2번의 할당이 필요하기 때문에 결점이 있다. 좀 더 나은 방법은 [EqualFold](https://golang.org/pkg/bytes/#EqualFold)()를 사용하는 것이다:

```go
func EqualFold(s, t []byte) bool
func EqualFold(s, t string) bool
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string), bool : [bool](https://golang.org/pkg/builtin/#bool)

"Fold"라는 용어는 [Unicode case-folding](https://www.w3.org/International/wiki/Case_folding)을 말한다. 이는 A-Z에 대한 일반적인 대소문자 규칙뿐만 아니라 ϕ의 φ로의 변환과 같은 다른 언어에 대한 규칙까지 포함한다.

### 비교

우리는 두 개의 바이트 슬라이스 또는 문자열의 정렬 순서를 결정하기 위해 [Compare](https://golang.org/pkg/bytes/#Compare)()를 사용할 것이다.

```go
func Compare(a, b []byte) int
func Compare(a, b string) int
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string), int : [int](https://golang.org/pkg/builtin/#int)

이 함수는 *a* 가 *b* 보다 작을 땐 -1, *a* 가 *b* 보다 클 땐 1, 그리고 *a* 와 *b* 가 같을 땐 0을 반환한다. 이 함수는 [strings](https://golang.org/pkg/strings/) 패키지에도 존재하는데 이는 단지 바이트 패키지와의 대칭을 이루기 위해서이다. Russ Cox도 함수의 주석에 써있는 "[기본적으로 아무도 strings.Compare를 사용해선 안된다](https://golang.org/src/strings/compare.go?s=697:741)"라는 말을 언급하며, 내장 연산자인 < 와 >를 대신 사용한다.

*"기본적으로 아무도 strings.Compare를 사용해선 안된다", Russ Cox*
{: style="text-align: center; font-size: 18px;"}

일반적으로, 정렬을 하기 위해 바이트 슬라이스가 다른 바이트 슬라이스보다 작은지에 대한 여부를 알고 싶을 것이다. sort.[Interface](https://golang.org/pkg/sort/#Interface)는 Less() 함수를 위해 이를 필요로한다. Less()는 [Compare](https://golang.org/pkg/bytes/#Compare)()의 세 가지 값을 갖는 반환값의 부울값으로의 변환을 필요로한다. 우리는 단순히 이 값이 -1과 같은지를 확인한다:

```go
type ByteSlices [][]byte

func (p ByteSlices) Less(i, j int) bool {
        return bytes.Compare(p[i], p[j]) == -1
}
```

<br>

## 검사(Inspection) 함수

바이트와 문자열 패키지는 바이트 슬라이스와 문자열내에서 데이터를 찾기 위한 몇 가지 방법들을 제공한다.

### 카운팅

만약 여러분이 유저의 입력의 유효성을 검증하고 있다면, 특정 바이트가 존재하는지 (또는 존재하지 않는지)를 검증하는건 중요하다. 하나 이상의 부분 슬라이스 또는 부분 문자열의 존재를 확인하기 위해 [Contains](https://golang.org/pkg/bytes/#Contains)() 함수를 사용할 수 있다:

```go
func Contains(b, subslice []byte) bool
func Contains(s, substr string) bool
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string), bool : [bool](https://golang.org/pkg/builtin/#bool)

예를 들면, 여러분이 부적절한 특정 단어들을 가진 입력을 허용하지 않을 수도 있다:

```go
if strings.Contains(input, "darn") {
        return errors.New("inappropriate input")
}
```

부분 슬라이스나 부분 문자열이 정확히 몇 번 사용되었는지를 알기 위해선, [Count](https://golang.org/pkg/bytes/#Count)()를 사용할 수 있다:

```go
func Count(s, sep []byte) int
func Count(s, sep string) int
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string), int : [int](https://golang.org/pkg/builtin/#int)

[Count](https://golang.org/pkg/bytes/#Count)()의 또 다른 사용은 문자열에 있는 룬의 갯수를 반환하는 것이다. *sep* 인자에 빈 슬라이스나 빈 문자열을 전달하면 [Count](https://golang.org/pkg/bytes/#Count)()는 룬의 갯수 + 1을 반환할 것이다. 이는 바이트 수를 반환하는 [len](https://golang.org/pkg/builtin/#len)()과는 다르다. 이 차이점은 멀티바이트 유니코드 문자들을 다룰 때 중요하다:

```go
strings.Count("I ❤ ☃", "")  // 6
len("I ❤ ☃")                // 9
```

첫번째 줄은 룬이 5개만 있기 때문에 이상하게 보일 수 있지만 [Count](https://golang.org/pkg/bytes/#Count)()는 룬 갯수 + 1을 반환한다는 걸 기억하라.

### 인덱싱

내용을 단언하는건 중요하지만 가끔은 부분 슬라이스나 부분 문자열의 정확한 위치를 찾아낼 필요가 있다. 인덱스 함수를 사용하면 가능하다:

```go
Index(s, sep []byte) int
IndexAny(s []byte, chars string) int
IndexByte(s []byte, c byte) int
IndexFunc(s []byte, f func(r rune) bool) int
IndexRune(s []byte, r rune) int
```

각기 다른 상황을 위한 여러개의 인덱스 함수들이 있다. [Index](https://golang.org/pkg/bytes/#Index)()는 멀티바이트 부분 슬라이스를 찾는다. [IndexByte](https://golang.org/pkg/bytes/#IndexByte)()는 슬라이스에서 단일 바이트를 찾는다. [IndexRune](https://golang.org/pkg/bytes/#IndexRune)()은 UTF-8로 해석된 바이트 슬라이스 내에서 유니코드 코드 포인트를 찾는다. [IndexAny](https://golang.org/pkg/bytes/#IndexAny)()는 [IndexRune](https://golang.org/pkg/bytes/#IndexRune)()처럼 동작하지만 한 번에 여러 개의 코드 포인트를 검색한다. 마지막으로, [IndexFunc](https://golang.org/pkg/bytes/#IndexFunc)()는 바이트 슬라이스에서 매칭될 때까지 각각의 룬을 검사하는 커스텀 함수를 전달할 수 있도록 해준다.

또한 바이트 슬라이스 또는 문자열의 끝의 첫번째 인스턴스를 검색하기 위한 함수의 매칭 세트도 있다.

```go
LastIndex(s, sep []byte) int
LastIndexAny(s []byte, chars string) int
LastIndexByte(s []byte, c byte) int
LastIndexFunc(s []byte, f func(r rune) bool) int
```

나는 인덱스 함수를 많이 사용하지 않는다, 왜냐하면 나는 보통 파서와 같은 좀 더 복잡한 것을 만들어야 하기 때문이다.

<br>

## 접두사, 접미사 & 트리밍(Trimming)

바이트 슬라이스 또는 문자열의 처음이나 끝부분의 내용을 가지고 작업하는 일은 검사의 특수한 경우지만 이는 한 섹션으로 다룰만큼 중요하다.

### 접두사 & 접미사 확인하기

접두사는 프로그래밍에서 자주 등장한다. 예를 들면, HTTP 경로는 보통 공통 접두사를 갖는 기능별로 그룹핑된다. 또 다른 예는 유저를 멘션하기 위한 "@"와 같은 문자열 첫부분의 특수 문자들이다.

[HasPrefix](https://golang.org/pkg/bytes/#HasPrefix)()와 [HasSuffix](https://golang.org/pkg/bytes/#HasSuffix)() 함수로 이러한 상황을 확인할 수 있다:

```go
func HasPrefix(s, prefix []byte) bool
func HasPrefix(s, prefix string) bool

func HasSuffix(s, prefix []byte) bool
func HasSuffix(s, suffix string) bool
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string), bool : [bool](https://golang.org/pkg/builtin/#bool)

이 함수들은 너무 간단해 보일 수 있지만 내가 본 한가지 흔한 실수는 개발자들이 길이가 0인 값을 확인하는 것을 잊어버릴 때이다:

```go
if str[0] == '@' {
        return true
}
```

이 코드는 간단해 보이지만 만약 *str*이 빈 문자열이라면 프로그램은 패닉을 발생 시킬 것이다. [HasPrefix](https://golang.org/pkg/bytes/#HasPrefix)() 함수는 여러분을 위해 이 유효성 검증을 포함하고 있다:

```
if strings.HasPrefix(str, "@") {
        return true
}
```
> HasPrefix : [HasPrefix](https://golang.org/pkg/bytes/#HasPrefix)()

### 트리밍(Trimming)

[bytes](https://golang.org/pkg/builtin/#byte)와 [strings](https://golang.org/pkg/builtin/#string) 패키지에서의 "trimming"이라는 용어는 바이트 슬라스나 문자열의 첫부분 그리고/또는 끝부분에 있는 바이트 또는 룬들을 제거한다는 의미이다. 이를 위한 가장 일반적인 함수는 [Trim](https://golang.org/pkg/bytes/#Trim)()이다:

```go
func Trim(s []byte, cutset string) []byte
func Trim(s string, cutset string) string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

이는 문자열의 첫부분과 끝부분에서 *cutset*의 모든 룬을 제거할 것이다. 여러분은 또한 [TrimLeft](https://golang.org/pkg/bytes/#TrimLeft)()와 [TrimRight](https://golang.org/pkg/bytes/#TrimRight)()를 사용하면 각각 문자열의 첫부분에서만 또는 끝부분에서만 룬을 제거할 수도 있다:

그러나 일반적인 트리밍은 정말 흔하지 않다. 대다수의 경우는 공백 문자를 없애는 일이며 이를 위해 [TrimSpace](https://golang.org/pkg/bytes/#TrimSpace)()를 사용할 수 있다:

```
func TrimSpace(s []byte) []byte
func TrimSpace(s string) string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

"\n\t" cutset을 가지고 트리밍을 하는걸로도 충분하다고 생각할 수 있지만 [TrimSpace](https://golang.org/pkg/bytes/#TrimSpace)()는 모든 유니코드로 정의된 공백을 트리밍 한다. 이는 스페이스, 개행, 그리고 탭 문자 뿐만 아니라 [thin space](https://en.wikipedia.org/wiki/Thin_space) 또는 [hair space](http://www.fileformat.info/info/unicode/char/200a/index.htm)와 같은 특이한 공백도 포함한다,

[TrimSpace](https://golang.org/pkg/bytes/#TrimSpace)()는 그냥 트리밍을 위해 전행(leading)과 후행(trailing) 룬들을 평가하는 함수인 [TrimFunc](https://golang.org/pkg/bytes/#TrimFunc)()의 [씬 래퍼](https://golang.org/src/strings/strings.go#L614)이다.

```go
func TrimSpace(s string) string {
        return TrimFunc(s, unicode.IsSpace)
}
```

이는 여러분이 자체적으로 후행 문자만을 위한 공백 트리머를 생성하는걸 쉽게 만들어준다.

```go
TrimRightFunc(s, unicode.IsSpace)
```
> TrimRightFunc : [TrimRightFunc](https://golang.org/pkg/bytes/#TrimRightFunc)

마지막으로, 문자셋 대신 접두사나 접미사만을 트리밍 하고싶은 경우엔 [TrimPrefix](https://golang.org/pkg/bytes/#TrimPrefix)()와 [TrimSuffix](https://golang.org/pkg/bytes/#TrimSuffix)() 함수를 사용하면 된다:

```go
func TrimPrefix(s, prefix []byte) []byte
func TrimPrefix(s, prefix string) string

func TrimSuffix(s, suffix []byte) []byte
func TrimSuffix(s, suffix string) string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

이들은 만약 여러분이 접두사나 접미어를 교체하고 싶은 경우 [HasPrefix](https://golang.org/pkg/bytes/#HasPrefix)()와 [HasSuffix](https://golang.org/pkg/bytes/#HasSuffix)() 함수와 함께 사용할 수 있다. 예를 들면, 나는 나의 설정 파일 경로에 대한 배쉬 스타일의 홈 디렉토리 완성을 구현하기 위해 이를 사용한다.

```go
// 유저의 홈 디렉토리를 찾는다.
u, err := user.Current()
if err != nil {
    return err
} else if u.HomeDir == "" {
    return errors.New("hoem directory does not exist")
}

// 물결표 대시 접두사를 홈 디렉토리로 교체.
if strings.HasPrefix(path, "~/") {
    path = filepath.Join(u.HomeDir, strings.TrimPrefix(path, "~/"))
}
```

<br>

## 교체 함수

### 기본 교체

부분 슬라이스나 부분 문자열을 교체하는건 때때로 필수적이다. 대부분의 단순한 경우엔, [Replace](https://golang.org/pkg/bytes/#Replace)() 함수면 다 된다:

```go
func Replace(s, old, new []byte, n int) []byte
func Replace(s, old, new string, n int) string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

이는 문자열에서 *old* 에 해당하는 모든 인스턴스를 *new* 로 바꿀 것이다. 교체의 횟수를 제한하려면 *n* 을 음이 아닌 정수로 지정하면 된다. 이 함수는 사용자 정의 템플릿에서 간단한 플레이스홀더를 가지고 있을 때 유용하다. 예를 들면, 유저가 "$NOW"를 지정하면 이를 현재 시각으로 교체하도록 할 수 있다;

```go
now := time.Now().Format(time.Kitchen)
println(strings.Replace(data, "$NOW", now, -1))
```
> Now : [Now](https://golang.org/pkg/time/#Now), Format : [Format](https://golang.org/pkg/time/#Time.Format), Kitchen : [Kitchen](https://golang.org/pkg/time/#Kitchen), Replace : [Replace](https://golang.org/pkg/strings/#Replace)

만약 다중 매핑을 가지고 있다면 strings.[Replacer](https://golang.org/pkg/strings/#Replacer)를 사용해야 할 것이다. 이는 strings.[NewReplacer](https://golang.org/pkg/strings/#NewReplacer)()에 old/new 쌍을 지정하여 사용한다.

```go
r := strings.NewReplacer("$NOW", now, "$USER", "mary")
println(r.Replace("Hello $USER, it is $NOW"))

// 출력값: Hello mary, it is 3:04PM
```
> NewReplacer : [NewReplacer](https://golang.org/pkg/strings/#NewReplacer), Replace : [Replace](https://golang.org/pkg/bytes/#Replace)

### 대소문자 교체

여러분은 대소문자 케이싱(casing: 문자 케이스 캐스팅)이 간단하다고 생각할 수도 있다. 그러나 Go는 유니코드를 가지고 동작하며 유니코드는 결코 간단하지 않다. 문자 케이싱은 3가지 타입이 있다: 대문자(upper case), 소문자(lower case) 그리고 타이틀 케이스(title case).

대문자와 소문자는 대다수의 언어에서 간단하며 여러분은 [ToUpper](https://golang.org/pkg/bytes/#ToUpper)()와 [ToLower](https://golang.org/pkg/bytes/#ToLower)() 함수를 사용할 수 있다:

```go
func ToUpper(s []byte) []byte
func ToUpper(s string) string

func ToLower(s []byte) []byte
func ToLower(s string) string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

그러나 몇몇 언어에서는 케이싱의 룰이 다르다. 예를 들면, 터키어는 **i**의 대문자가 **İ**이다. 이러한 특수한 언어들을 위한 함수들의 특수한 버전이 존재한다.

```go
strings.ToUpperSpecial(unicode.TurkishCase, "i")
```
> ToUpperSpecial : [ToUpperSpecial](https://golang.org/pkg/strings/#ToUpperSpecial)

다음으로 타이틀 케이스는 [ToTitle](https://golang.org/pkg/bytes/#ToTitle)() 함수가 있다:

```go
func ToTitle(s []byte) []byte
func ToTitle(s string) string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

그러나, 여러분이 [ToTitle](https://golang.org/pkg/bytes/#ToTitle)()를 사용하게 되면 놀랄 것이다. 모든 문자열이 대문자가 되어버렸다:

```go
println(strings.ToTitle("the count of monte cristo"))

// 출력값 : THE COUNT OF MONTE CRISTO
```
> ToTitle : [ToTitle](https://golang.org/pkg/bytes/#ToTitle)

왜냐하면 유니코드에서, 타이틀 케이스는 케이싱의 특정한 유형이며 각 단어의 첫번째 문자를 대문자화 시키는 방법이 아니다. 대부분의 경우, 타이틀 케이스와 대문자화는 동일하지만 차이점을 갖는 몇 개의 코드 포인트들이 존재한다. 예를 들면, **[lj](https://www.compart.com/en/unicode/U+01C9)** 코드 포인트 (맞다, 이는 하나의 코드 포인트이다)는 대문자화가 되면 **LJ**가 되지만 타이틀 케이스는 **Lj**이다.

여러분이 아마 찾고 있는 것은 [Title](https://golang.org/pkg/bytes/#Title)()이다:

```go
func Title(s []byte) []byte
func Title(s string) string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

이 함수의 출력값은 예상했던 결과값이다:

```go
println(strings.Title("the count of monte cristo"))

// 출력값 : The Count Of Monte Cristo
```
> Title : [Title](https://golang.org/pkg/strings/#Title)

### 룬 매핑하기

바이트 슬라이스 또는 문자열에서 데이터를 교체하기 위한 또 다른 함수는 [Map](https://golang.org/pkg/bytes/#Map)()이다:

```go
func Map(mapping func(r rune) rune, s []byte) []byte
func Map(mapping func(r rune) rune, s string) string
```
> rune : [rune](https://golang.org/pkg/builtin/#rune), byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

이 함수는 모든 룬을 평가하고 이를 교체하는 함수를 전달하도록 한다. 인정하건데, 나는 이 포스트를 쓰기 시작할때까지 이 함수의 존재조차 모르고 있었기에 그 어떤 개인적인 일화도 알려줄 수가 없다.

<br>

## 분할(Splitting) & 결합(Joining) 함수

여러 번 우리는 서로 분할시킬 필요가 있는 문자열들을 구분하고있다. 예를 들면, 유닉스에서 경로들은 콜론으로 합쳐져 있으며 CSV 파일 포맷은 기본적으로 데이터 필드를 콤마로 구분하고 있다.

### 부분 문자열 분할

간단한 부분 슬라이스와 부분 문자열 분할을 위해, 우리는 Split() 함수를 사용한다:

```go
func Split(s, sep []byte) [][]byte
func SplitAfter(s, sep []byte) [][]byte
func SplitAfterN(s, sep []byte, n int) [][]byte
func SplitN(s, sep []byte, n int) [][]byte

func Split(s, sep string) []string
func SplitAfter(s, sep string) []string
func SplitAfterN(s, sep string, n int) []string
func SplitN(s, sep string, n int) []string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

이렇게 구분자로 바이트 슬라이스와 문자열을 쪼개면 부분 슬라이스들과 문자열들이 반환된다. "After" 함수는 서브 문자열의 끝에 구분자를 포함한다. "N" 함수는 발생할 수 있는 분할 횟수를 제한한다:

```go
strings.Split("a:b:c", ":")       // ["a", "b", "c"]
strings.SplitAfter("a:b:c", ":")  // ["a:", "b:", "c"]
strings.SplitN("a:b:c", ":", 2)   // ["a", "b:c"]
```

데이터를 분할하는 것은 매우 일반적인 작업이다. 하지만, 이는 보통 CSV와 같은 파일 포맷에서의 컨텍스트나 경로 분할의 컨텍스트에서 수행된다. 이 작업들을 위해, 나는 [encoding/csv](https://golang.org/pkg/encoding/csv/) 또는 [path](https://golang.org/pkg/path/) 패키지를 대신 사용한다.

### 카테고리성 분할

가끔 구분자로 룬의 열 대신 룬의 집합을 지정하고 싶을 때가 있다. 이 상황의 가장 좋은 예시는 가변적인 공백이 있는 상태에서 단어들을 쪼개는 것이다. 공백 구분 함수로 단순히 [Split](https://golang.org/pkg/bytes/#Split)()을 사용하게되면 연속적인 다중 공백이 있을 경우 빈 부분 문자열도 반환할 것이다. 대신 Fields() 함수를 사용할 수 있다:

```go
func Fields(s []byte) [][]byte
```
> byte : [byte](https://golang.org/pkg/builtin/#byte)

이는 연속된 공백 문자들을 하나의 구분자로 여긴다:

```go
func FieldsFunc(s []byte, f func(rune) bool) [][]byte
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), rune : [rune](https://golang.org/pkg/builtin/#rune)

### 문자열 결합

구분된 데이터를 쪼개는 대신, Join() 함수를 이용해 이를 하나로 합칠 수도 있다:

```go
func Join(s [][]byte, sep []byte) []byte
func Join(a []string, sep string) string
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), string : [string](https://golang.org/pkg/builtin/#string)

개발자들이 join 함수를 직접 구현할 때 내가 봤던 한가지 일반적인 실수가 있는데 이는 다음과 같다:

```go
var output string
for i, s := range a {
        output += s
        if i < len(a) - 1 {
            output += ","
        }
}
return output
```

이 코드의 결점은 많은 수의 할당을 생성하고 있다는 것이다. 문자열은 변형 불가능하기 때문에, 각 반복은 각 문자열 추가를 위해 새로운 문자열을 생성하고 있다. 반면에, strings.[Join](https://golang.org/pkg/strings/#Join)() 함수는 생성시에는 바이트 슬라이스 버퍼를 사용하며 이를 반환할 때 문자열로 변환한다. 이는 힙 메모리 할당을 최소화한다.

<br>

## 이외 함수들

카테고리를 찾지 못한 두 개의 함수가 있는 이는 여기서 다룬다. 첫째, [Repeat](https://golang.org/pkg/bytes/#Repeat)() 함수는 반복된 바이트 슬라이스 또는 문자열을 생성할 수 있도록 해준다. 솔직히, 나는 이를 터미널에서 내용을 구분하기 위해 라인을 만들때 사용했던것만 기억난다:

```go
println(strings.Repeat("-", 80))
```

또 다른 함수는 UTF-8으로 해석된 바이트 슬라이스 또는 문자열에 있는 모든 룬들의 슬라이스를 반환하는 [Runes](https://golang.org/pkg/bytes/#Runes)()이다. 나는 이 함수의 필요성을 전혀 느낀적이  없는데 이는 새로운 할당 없이 *string* 위에서 *for loop* 를 돌려 같은 일을 할 수 있기 때문이다.

<br>

## 결론

바이트 슬라이스와 문자열은 Go에서 가장 기본적인 요소이다. 이들은 바이트와 룬의 열을 위한 인메모리 표현이다. 바이트와 문자열 패키지는 io.Reader와 io.Writer 인터페이스의 개조형뿐만 아니라 수 많은 유용한 헬퍼 함수들을 제공한다.

API의 사이즈가 크지 않기 때문에 이 패키지들의 많은 유용한 툴들을 살펴보는일은 쉽지만 나는 이 포스트가 여러분이 이 패키지가 제공해줄 수 있는 모든 것들을 이해하는데 도움이 되었으면 한다.
