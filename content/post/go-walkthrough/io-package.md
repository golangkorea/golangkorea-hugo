+++

title = "Go 둘러보기 - io 패키지"
draft = false
date = "2016-10-26T02:30:33+09:00"

tags = ["Golang", "IO", "Package"]
categories = ["번역", "IO"]
series = ["Go  Walkthrough"]
authors = ["mingrammer"]

toc = true

+++

> [Go Walkthrough](https://medium.com/go-walkthrough) 시리즈의 [Go Walkthrough: io package](https://medium.com/go-walkthrough/go-walkthrough-io-package-8ac5e95a9fbd#.41qxgc1zr)를 번역한 글입니다.

Go는 바이트(bytes)를 사용하여 작업하기 위해 만들어진 프로그래밍 언어이다. 바이트의 리스트, 바이트의 스트림, 또는 단일 바이트중 무얼 가지고 있는지와 상관없이 Go는 이를 쉽게 처리해준다. 이러한 간단한 기본적인 것들로 우리는 추상화 및 서비스를 구축한다.

io 패키지는 표준 라이브러리 내에서 가장 기본적인 패키지 중 하나이다. 이는 바이트 스트림을 가지고 작업을 하기 위한 인터페이스와 헬퍼(함수)의 모음을 제공한다.

이 포스트는 표준 라이브러리를  이해하는데 도움을 주기위한 Go 둘러보기 시리즈의 일부이다. 기존에 생성된 문서(자동으로 생성된 Go 문서)는 많은 정보를 제공하지만, 이는 패키지를 실제 상황에서 이해하기에는 어려울 수 있다. 이 시리즈는 일상적으로 사용되는 애플리케이션에서 표준 패키지들이 어떻게 사용되는지에 대한 컨텍스트를 제공할 수 있도록 도와준다. 질문이나 코멘트가 있다면 트위터에서 [@benbjohnson](https://twitter.com/benbjohnson)로 찾아오면 된다.

<br>

## 바이트 읽기

바이트로 작업을 할 때 사용되는 두 가지 기본적인 연산이 있는데 바로 읽기와 쓰기이다. 우선 바이트 읽기에 대해서 살펴보자.

### Reader 인터페이스

스트림에서 바이트를 읽기 위한 기본 구조체는 [Reader](https://golang.org/pkg/io/#Reader) 인터페이스이다:

```go
type Reader interface {
    Read(p []byte) (n int, err error)    
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), int : [int](https://golang.org/pkg/builtin/#int), error : [error](https://golang.org/pkg/builtin/#error)

이 인터페이스는 [network connections](https://golang.org/pkg/net/#Conn)부터 [files](https://golang.org/pkg/os/#File)와 [wrappers for in-memory slices](https://golang.org/pkg/io/#Reader)에 이르기까지 모든 표준 라이브러리에 걸쳐 구현된다.

Reader는 동일한 바이트를 재사용할 수 있도록 버퍼(p를 말함)를 Read() 메서드에 전달함으로써 동작한다. 만약 Read()가 바이트 슬라이스를 하나의 인자로 받는 대신 이를 반환하게되면 Reader는 Read()를 호출할 때마다 새로운 바이트 슬라이스를 할당해야 할 것이다. 이는 가비지 컬렉터에 안좋은 영향을 끼친다.

Reader 인터페이스의 한 가지 문제점은 애매한 규칙들을 가지고 있다는 것이다. 첫째, 이는 스트림이 완료되면 io.EOF 에러를 정상적인 동작을 하는 것처럼 반환한다. 이는 초보자에게는 혼란을 가져올 수 있다. 둘째, 버퍼가 채워질거라는 보장이 없다. 만약 여러분이 8 바이트 슬라이스를 전달한다면 여러분은 0부터 8바이트 사이의 그 어떤값으로도 돌려받을 수 있다. 부분 읽기를 다루는것은 지저분하고 에러가 발생하기 쉽다. 다행히도 이 문제를 해결하기 위한 헬퍼 함수가 있다.

### Reader의 보장성 개선하기

여러분이 파싱 프로토콜을 가지고있고 Reader로부터 unit64 타입의 값을 8 바이트 읽어야한다고 해보자. 이런 경우엔 고정된 크기만큼 읽어야하기 때문에 io.ReadFull()을 사용하는게 더 적합하다.

```go
func ReadFull(r Reader, buf[] byte) (n int, err error)
```
> Reader : [Reader](https://golang.org/pkg/io/#Reader), byte : [byte](https://golang.org/pkg/builtin/#byte), error : [error](https://golang.org/pkg/builtin/#error)

이 함수는 버퍼가 반환되기 전에 데이터가 완전히 채워짐을 보장한다. 만약 버퍼가 일부만 읽었을 경우 여러분은  io.ErrUnexpectedEOF을 돌려받을 것이다. 만약 버퍼가 아무것도 읽지 않았을 경우엔 io.EOF가 반환된다. 이 간단한 보증은 코드를 매우 단순화시킨다. 8 바이트를 읽기 위해선 다음처럼만 하면된다.

```go
buf := make([]byte, 8)
if _, err := io.ReadFull(r, buf); err == io.EOF {
    return io.ErrUnexpectedEOF
} else if err != nil {
    return err
}
```

Go에는 특정 타입의 파싱을 처리하는 [binary.Read()](https://golang.org/pkg/encoding/binary/#Read)와 같은 고수준의 파서들이 많이 있다. 우리는 이들을 나중에 다른 패키지에서 다룰 것이다.

또 다른 사용빈도가 낮은 헬퍼 함수는 [ReadAtLeast](https://golang.org/pkg/io/#ReadAtLeast)이다:

```go
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error)
```

이 함수는 추가 데이터를 읽을 수 있는 경우 이를 버퍼로 읽어들이지만 항상 최소한의 바이트 수를 반환한다. 개인적으로는 이 함수의 필요성을 찾지 못했지만 Read() 호출을 최소화하고 추가 데이터를 버퍼링하고자 한다면 유용하게 사용할 수 있을 것 같다.

### 스트림 연결

여러개의 Reader를 하나로 결합해야하는 경우를 많이 볼 것이다. [MultiReader](https://golang.org/pkg/io/#MultiReader)를 사용하면 이들을 하나의 Reader로 합칠 수 있다.

```go
func MultiReader(readers ...Reader) Reader
```
> Reader : [Reader](https://golang.org/pkg/io/#Reader)

예를 들면, 디스크에 있는 데이터와 인메모리 헤더가 결합된 HTTP 요청을 보낼 수도 있을 것이다. 많은 사람들이 헤더와 파일을 인메모리 버퍼로 복사하려고 하지만 이는 느리고 많은 메모리를 사용할 수 있다.

다음과 같은 간단한 방법이 있다:

```go
r := io.MultiReader(
    bytes.NewReader([]byte("...my header...")),
    myFile,
)

http.Post("http://example.com", "application/octet-stream", r)
```
> MultiReader : [MultiReader](https://golang.org/pkg/io/#MultiReader), NewReader : [NewReader](https://golang.org/pkg/bytes/#Reader), Post : [Post](https://golang.org/pkg/net/http/#Post)

MultiReader는 [http.Post()](https://golang.org/pkg/net/http/#Post)가 두 개의 Reader를 하나의 연결된 Reader로 간주하도록 한다.

### 스트림 복제

Reader를 사용할 때 맞닥뜨릴 수 있는 한가지 문제는 Reader가 한 번 읽히면, 데이터를 다시 읽을 수 없다는 것이다. 예를 들어, 애플리케이션이 HTTP 요청 파싱을 실패하면 파서는 이미 데이터를 다 사용했기 때문에 디버깅을 할 수 없을 것이다.

[TeeReader](https://golang.org/pkg/io/#TeeReader)는 Reader에서 데이터를 다 읽어버리는것에 방해받지 않으면서 Reader의 데이터를 캡쳐하는데 있어 훌륭한 선택이다.

```go
func TeeReader(r Reader, w Writer) Reader
```
> Reader : [Reader](https://golang.org/pkg/io/#Reader), Writer : [Writer](https://golang.org/pkg/io/#Writer)

이 함수는 여러분의 Reader(r을 말함)를 래핑하는 새로운 Reader를 생성한다. 새로운 Reader에서 읽는것들은 또한 w에 저장될 것이다. 이 Writer는 [in-memory buffer](https://golang.org/pkg/bytes/#Buffer)부터 로그 파일, [STDERR](https://golang.org/pkg/os/#pkg-variables)까지 그 어떤것도 가능하다.

예를 들면, 잘못된 요청은 다음과 같이 캡쳐할 수 있다:

```go
var buf bytes.Buffer
body := io.TeeReader(req.Body, &buf)

// ... process body ...

if err != nil {
    // inspect buf
    return err
}
```

그러나, 메모리가 부족하지 않도록 캡쳐하려는 요청을 제한하는것이 중요하다.

### 스트림 길이 제한

스트림은 제한이 없기 때문에 몇몇 상황에선 메모리나 디스크 이슈를 일으킬 수 있다. 가장 일반적인 예시는 파일 업로드 엔드포인트이다. 엔드포인트는 일반적으로 디스크가 꽉 차는걸 방지하기위해 크기 제한을 가지고 있지만, 이를 직접 구현하는건 지루할 수 있다.

[LimitReader](https://golang.org/pkg/io/#LimitReader)는 Reader가 전체 바이트 수를 제한하도록 래핑함으로써 이 기능을 제공한다.

```go
func LimitReader(r Reader, n int64) Reader
```
> Reader : [Reader](https://golang.org/pkg/io/#Reader), int64 : [int64](https://golang.org/pkg/builtin/#int64)

LimitReader의 한가지 문제는 Reader가 n을 초과하는지에 대한것을 알려주지 않는다는 것이다. 이는 단순히 r에서 n 바이트를 읽게되면 [io.EOF](https://golang.org/pkg/io/#EOF)를 반환할 것이다. 여러분이 사용할 수 있는 한가지 트릭은 제한값을 n+1로 설정한 후 마지막 바이트을 보고 n 바이트보다 많은 바이트를 읽었는지 아닌지를 판별하는 것이다.

<br>

## 바이트 쓰기

스트림으로부터 바이트를 읽는것에 대해 다뤄봤다. 이제 이를 어떻게 스트림에 쓸 수 있는에 대해 살펴보자.

### Writer 인터페이스

Writer 인터페이스는 단순히 Reader의 반대이다. 우리는 스트림에 넣기 위한 바이트 버퍼를 제공한다.

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), int : [int](https://golang.org/pkg/builtin/#int), error : [error](https://golang.org/pkg/builtin/#error)

일반적으로 바이트 쓰기는 읽기보다 간단하다. Reader는 부분 읽기를 허용하기 때문에 데이터 다루기가 까다롭지만, 부분 쓰기는 항상 에러를 반환한다.

### 쓰기 복제

가끔은 쓰기 작업을 여러개의 스트림에 보내고 싶을 것이다. 아마 로그 파일 또는 STDERR로. 이는 읽기를 복제하는 대신 쓰기를 복제한다는것만 제외하면 [TeeReader](https://golang.org/pkg/io/#TeeReader)와 유사하다

이 경우엔 [MultiWriter](https://golang.org/pkg/io/#MultiWriter)가 유용하다:

```go
func MultiWriter(writers ...Writer) Writer
```
> Writer : [Writer](https://golang.org/pkg/io/#Writer)

이는 [MultiReader](https://golang.org/pkg/io/#MultiReader)의 Writer 버전이 아니기 때문에 이름이 약간 혼란스러울 수 있다. MultiReader는 여러개의 Reader를 하나로 합쳐주는데 반해, MultiWriter는 각 쓰기를 여러개의 Writer에 복제하는 하나의 Writer를 반환한다.

나는 서비스가 제대로 로깅을 하고 있다는걸 단언해야하는 단위 테스트에서 광범위하게 MultiWriter를 사용하고 있다.

```go
type MyService struct {
    LogOutput io.Writer
}

var buf bytes.Buffer
var s MyService
s.LogOutput = io.MultiWriter(&buf, os.Stderr)
```
> Buffer : [Buffer](https://golang.org/pkg/bytes/#Buffer), MultiWriter : [MultiWriter](https://golang.org/pkg/io/#MultiWriter), Stderr : [Stderr](https://golang.org/pkg/os/#Stderr)

MultiWriter를 사용하면 디버깅을 위해 터미널에서 전체 로그를 보는 동시에 버퍼의 내용을 검증할 수 있게 해준다.

### 문자열 쓰기 최적화

표준 라이브러리에는 문자열을 바이트 슬라이스로 변환할 때 별다른 메모리 할당을 요구하지 않음으로써 쓰기 성능을 향상 시킬 수 있는 WriteString() 메서드를 가진 많은 Writer가 있다. io.[WriteString](https://golang.org/pkg/io/#WriteString)() 함수를 사용하면 이 최적화를 활용할 수 있다.

이 함수는 간단하다. 먼저 Writer가 WriteString() 메서드를 구현하고 있는지를 확인하며 만약 구현이 되어있으면 이를 사용한다. 그렇지 않은 경우엔 Write() 메서드를 사용하여 문자열을 바이트 슬라이스로 복사한다.

(이 부분을 짚어준 [Bouke van der Bijl](https://twitter.com/BvdBijl)에게 감사를 전한다)

<br>

## 바이트 복사

이제 우린 바이트를 읽고 쓸 수 있으며, 이 양쪽을 연결하는것과 Reader와 Writer간의 복사만 이해하면된다.

### Reader와 Writer를 연결

Reader를 Writer로 복사하는 가장 기초적인 방법은 적절하게 명명된 [Copy](https://golang.org/pkg/io/#Copy)() 함수이다:

```go
func Copy(dst Writer, src Reader) (written int64, err error)
```
> Writer : [Writer](https://golang.org/pkg/io/#Writer), Reader : [Reader](https://golang.org/pkg/io/#Reader), int64 : [int64](https://golang.org/pkg/builtin/#int64), error : [error](https://golang.org/pkg/builtin/#error)

이 함수는 src로부터 값을 읽기 위해 32KB 버퍼를 사용하며 이를 dst에 쓴다. 만약 읽기 또는 쓰기를 하는 도중 io.EOF 이외의 어떤 에러가 발생하면 복사는 중단되며 에러가 반환된다.

Copy()의 한가지 문제점은 바이트 수의 최댓값을 보장할 수 없다는 것이다. 예를 들면, 로그 파일을 현재 파일 사이즈만큼 복사하고 싶을 수 있다. 만약 로그가 복사를 하는중에 계속 증가하게되면 예상했던것보다 더 많은 바이트를 읽게될 것이다. 이 경우엔 정확히 몇 바이트를 쓸건지 지정할 수 있는 [CopyN](https://golang.org/pkg/io/#CopyN)() 함수를 사용할 수 있다.

```go
func CopyN(dst Writer, src Reader, n int64) (written int64, err error)
```
> Writer : [Writer](https://golang.org/pkg/io/#Writer), Reader : [Reader](https://golang.org/pkg/io/#Reader), int64 : [int64](https://golang.org/pkg/builtin/#int64), error : [error](https://golang.org/pkg/builtin/#error)

Copy()의 또 다른 문제는 매 호출마다 32KB의 할당이 필요하다는 것이다. 만약 많은 양의 복사를 한다고하면 [CopyBuffer](https://golang.org/pkg/io/#CopyBuffer)()를 대신 사용함으로써 버퍼를 재사용 할 수 있다:

```go
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error)
```
> Writer : [Writer](https://golang.org/pkg/io/#Writer), Reader : [Reader](https://golang.org/pkg/io/#Reader), byte : [byte](https://golang.org/pkg/builtin/#byte), int64 : [int64](https://golang.org/pkg/builtin/#int64), error : [error](https://golang.org/pkg/builtin/#error)

나는 Copy()의 오버헤드가 매우 커지는 경우를 본 적이 없어서 개인적으로는 CopyBuffer()를 사용하지 않는다.

### 복사 최적화

중간 버퍼를 아예 사용하지 않기위해, 타입은 직접 읽기와 쓰기 인터페이스를 구현할 수 있다. 구현된 경우, Copy() 함수는 중간 버퍼를 사용하지 않고 이러한 구현을 직접 사용한다.

[WriterTo](https://golang.org/pkg/io/#WriterTo) 인터페이스는 직접 데이터를 쓰고자하는 타입에서 사용할 수 있다.

```go
type WriterTo interface {
    WriteTo(w Writer) (n int64, err error)
}
```
> Writer : [Writer](https://golang.org/pkg/io/#Writer), int64 : [int64](https://golang.org/pkg/builtin/#int64), error : [error](https://golang.org/pkg/builtin/#error)

나는 이걸 BoltDB의 사용자가 트랜젝션으로부터 데이터베이스 스냅샷을 만들 수 있도록 해주는 [Tx.WriteTo()](https://godoc.org/github.com/boltdb/bolt#Tx.WriterTo)에 사용했었다.

읽기쪽에서는, [ReaderFrom](https://golang.org/pkg/io/#ReaderFrom)이 타입으로 하여금 Reader로부터 데이터를 직접 읽을 수 있게 해준다.

```go
type ReaderFrom interface {
    ReadFrom(r Reader) (n int6, err error)
}
```
> Reader : [Reader](https://golang.org/pkg/io/#Reader), int64 : [int64](https://golang.org/pkg/builtin/#int64), error : [error](https://golang.org/pkg/builtin/#error)

### Reader와 Writer 개조하기

가끔 Reader를 받는 함수가 있지만 Writer만 가지고 있는 경우가 있을 수 있다. 아마도 여러분은 HTTP 요청에 동적으로 데이터를 써 보내야할 필요가 있을지도 모른다. 그러나 http.[NewRequest](https://golang.org/pkg/net/http/#NewRequest)()는 오직 Reader만 받는다.

io.[Pipe](https://golang.org/pkg/io/#Pipe)()를 사용해 Writer를 반전시킬 수 있다:

```go
func Pipe() (*PipeReader, *PipeWriter)
```
> PipeReader : [PipeReader](https://golang.org/pkg/io/#PipeReader), PipeWriter : [PipeWriter](https://golang.org/pkg/io/#PipeWriter)

이는 새로운 Reader와 Writer를 제공해준다. 새로운 PipeWriter에 대한 모든 쓰기는 PipeReader로 이동할 것이다.

나는 이 기능을 직접 사용해본적은 거의 없지만, exec.[Cmd](https://golang.org/pkg/os/exec/#Cmd)()가 명령어 실행 작업을 할 때 매우 유용하게 사용되는 Stdin, Stdout, 그리고 Stderr 파이프를 구현하는데 이를 사용한다.

<br>

## 스트림 닫기

모든 좋은 것들은 마무리를 지어야하며 이는 바이트 작업시에도 예외는 아니다. 스트림을 닫기 위한 일반적인 방법으로 [Closer](https://golang.org/pkg/io/#Closer) 인터페이스가 제공된다.

```go
type Closer interface {
    Close() error
}
```
> error : [error](https://golang.org/pkg/builtin/#error)

Closer는 매우 간단해서 별로 말할게 없지만, 나는 Closer가 필요할 때 내가 만든 타입이 이를 구현할 수 있도록 내 Close()로부터 항상 에러를 반환시키는게 유용하다는걸 발견했다. Closer는 항상 직접 사용되지는 않지만 가끔 [ReadCloser](https://golang.org/pkg/io/#ReadCloser), [WriteCloser](https://golang.org/pkg/io/#WriteCloser), 그리고 [ReadWriteCloser](https://golang.org/pkg/io/#ReadWriteCloser)와 같은 다른 인터페이스와 결합해서 사용될 수 있다.

<br>

## 스트림 내에서의 이동

스트림은 보통 시작부터 끝까지 연속적인 바이트의 흐름(flow)이지만, 몇 가지 예외가 있다. 예를 들면, 파일은 스트림으로 동작할 수 있지만 파일 내의 특정한 위치로 건너뛸 수도 있다.

스트림 내에서 건너뛸 수 있도록 [Seeker](https://golang.org/pkg/io/#Seeker) 인터페이스가 제공된다.

```go
type Seeker interface {
    Seek(offset int64, whence int) (int64, error)
}
```
> int64 : [int64](https://golang.org/pkg/builtin/#int64), int : [int](https://golang.org/pkg/builtin/#int), error : [error](https://golang.org/pkg/builtin/#error)

건너뛰기 위한 방법은 3가지가 있다: 현재 위치를 기준으로 이동하기, 시작점을 기준으로 이동하기, 그리고 끝점을 기준으로 이동하기. whence 인자를 사용해 이동 모드를 지정할 수 있다. offset 인자는 몇 바이트를 이동할 것인지를 지정한다.

<br>

## 데이터 타입 최적화

청크에서의 읽기와 쓰기는 단일 바이트나 단일 룬이 필요할 때에는 지루해질 수 있다. Go는 이를 쉽게 만들어주는 몇 가지 인터페이스를 제공한다.

### 단일 바이트 작업

[ByteReader](https://golang.org/pkg/io/#ByteReader)와 [ByteWriter](https://golang.org/pkg/io/#ByteWriter) 인터페이스는 단일 바이트를 읽고 쓰기 위한 간단한 인터페이스를 제공한다:

```go
type ByteReader interface {
    ReadByte() (c byte, err error)
}

type ByteWriter interface {
    WriteByte(c byte) error
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error : [error](https://golang.org/pkg/builtin/#error)

길이는 항상 0 또는 1이 될 것이기 때문에 길이 인자가 없다는걸 볼 수 있다. 만약 바이트가 읽히거나 쓰이지 않으면 에러가 반환된다.

버퍼링된 바이트 Reader로 작업을 하기 위한 [ByteScanner](https://golang.org/pkg/io/#ByteScanner) 인터페이스 또한 제공된다.

```go
type ByteScanner interface {
    ByteReader
    UnreadByte() error
}
```
> ByteReader : [ByteReader](https://golang.org/pkg/io/#ByteReader), error : [error](https://golang.org/pkg/builtin/#error)

이는 다음에 읽을 수 있도록 이전에 읽은 바이트를 다시 Reader에 넣는다. 이는 다음에 사용가능한 바이트를 미리 볼 수 있도록 해주기 때문에 LL(1) 파서를 작성할 때 특히 유용하다.

### 단일 룬 작업

만약 유니코드 데이터를 파싱중이라면 개별 바이트 대신 룬으로 작업을 해야할 것이다. 이 경우, [RuneReader](https://golang.org/pkg/io/#RuneReader)와 [RuneScanner](https://golang.org/pkg/io/#RuneScanner)가 대신 사용된다.

```go
type RuneReader interface {
        ReadRune() (r rune, size int, err error)
}
type RuneScanner interface {
        RuneReader
        UnreadRune() error
}
```
> rune : [rune](https://golang.org/pkg/builtin/#rune), int : [int](https://golang.org/pkg/builtin/#int), error : [error](https://golang.org/pkg/builtin/#error)

<br>

## 결론

바이트 스트림은 대부분의 Go 프로그램에 필수적이다. 이들은 네트워크 연결에서 디스크의 파일, 키보드로부터의 사용자 입력에 이르기까지의 모든 것에 대한 인터페이스이다. [io](https://golang.org/pkg/io/) 패키지는 이러한 모든 인터랙션을 위한 기초를 제공한다.

우리는 바이트 읽기, 바이트 쓰기, 바이트 복사하기, 그리고 마지막으로 이 연산들을 최적화하는 방법들을 살펴봤다. 이러한 기본적인 요소들은 간단해 보일 수 있지만 이들은 모든 데이터 중심 애플리케이션을 위한 빌딩 블록을 제공한다. [io](https://golang.org/pkg/io/) 패키지를 살펴보고 여러분의 애플리케이션에서 이들의 인터페이스를 고려해보길 바란다.
