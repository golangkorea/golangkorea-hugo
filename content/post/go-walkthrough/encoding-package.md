+++

title = "Go 둘러보기 - encoding 패키지"
draft = false
date = "2016-11-09T15:33:33+09:00"

tags = ["Golang", "Encoding"]
categories = ["번역", "Encoding"]
series = ["Go  Walkthrough"]
authors = ["mingrammer"]

toc = true

+++

> [Go Walkthrough](https://medium.com/go-walkthrough) 시리즈의 [Go Walkthrough: encoding package](https://medium.com/go-walkthrough/go-walkthrough-encoding-package-bc5e912232d#.hgvug2knk)를 번역한 글입니다.

우리는 이제까지 [로우(raw) 바이트 스트림](https://mingrammer.com/translation-go-walkthrough-io-package)과 [제한된 바이트 슬라이스](https://mingrammer.com/translation-go-walkthrough-bytes-strings-packages)를 다뤄봤지만 단순히 바이트만을 사용하는 애플리케이션은 별로 없다. 바이트 자체는 많은 의미를 전달해주지 않지만 바이트 위에서 데이터 구조를 인코딩 한다면 우리는 진정으로 유용한 애플리케이션을 구축할 수 있다.

이 포스트는 표준 라이브러리를  이해하는데 도움을 주기위한 Go 둘러보기 시리즈의 일부이다. 기존에 생성된 문서(
자동으로 생성된 Go 문서)는 많은 정보를 제공하지만, 이는 패키지를 실제 상황에서 이해하기에는 어려울 수 있다.
이 시리즈는 일상적으로 사용되는 애플리케이션에서 표준 패키지들이 어떻게 사용되는지에 대한 컨텍스트를 제공할
수 있도록 도와준다. 질문이나 코멘트가 있다면 트위터에서 [@benbjohnson](https://twitter.com/benbjohnson)로 찾
아오면 된다.

<br/>

## 인코딩이란 정확히 무엇인가?

컴퓨터 과학에서는 간단한 개념에 대한 팬시한 단어들이 있다. 이 뿐만 아니라 많은 경우 하나의 개념을 지칭하는 많은 팬시한 단어들이 존재하기도 한다. *인코딩(Encoding)* 은 그 중 하나이다. 가끔 이는 *시리얼라이제이션(serialization)* 또는 *마샬링(marshaling)* 으로 불리기도 하며 이는 모두 로우(raw) 바이트에 논리적 구조체를 더한다는 것을 의미한다.

Go 표준 라이브러리에서, 우리는 두 가지의 분리되었지만 서로 관련이 있는 아이디어를 위해 *encoding*과 *marshaling* 라는 용어를 사용한다. Go에서 *encoder* 는 구조체를 바이트 스트림에 적용하는 객체인 반면 *marshaling* 은 구조체를 제한된 인메모리 바이트에 적용하는 것을 말한다.

예를 들면, *encoding/json* 패키지는 각각의 io.[Writer](https://golang.org/pkg/io/#Writer)와 io.[Reader](https://golang.org/pkg/io/#Reader)로 작업을 하기 위한 json.[Encoder](https://golang.org/pkg/encoding/json/#Encoder)와 json.[Decoder](https://golang.org/pkg/encoding/json/#Decoder)를 가지고있다. 이 패키지는 또한 바이트 슬라이스에 데이터를 쓰고 읽기 위한 json.[Marshaler](https://golang.org/pkg/encoding/json/#Marshaler)과 json.[Unmarshaler](https://golang.org/pkg/encoding/json/#Unmarshaler)를 가지고 있다.


### 두 가지 타입의 인코딩

인코딩간에는 또 다른 중요한 차이점이 있다. 몇몇의 인코딩 패키지는 문자열, 정수등의 프리미티브에서 동작한다. 문자열은 아스키나 [유니코드](https://golang.org/pkg/unicode/) 또는 다른 [언어별 인코딩](https://godoc.org/golang.org/x/text)과 같은 문자 인코딩을 가지고 인코딩이 된다. 정수는 [엔디안(endianness)](https://en.wikipedia.org/wiki/Endianness)기반 또는 가변 길이 인코딩에 따라 조금 다르게 인코딩 될 수 있다. 심지어 바이트자체는 종종 출력가능한 문자들로 변환 하기위해 [Base64](https://golang.org/pkg/encoding/base64/)와 같은 방식으로 인코딩된다.

그러나 우리는 종종 인코딩이라고 하면, 객체 인코딩을 떠올린다. 이는 *구조체(structs)*, *맵(maps)*, 그리고 슬라이스(slices) 같은 복잡한 구조체들을 바이트열로 변환하는것을 지칭한다. 이런 변환을 하는데 있어선 많은 트레이드 오프가 있으며 오랜 시간에 걸쳐 [많은 사람들이 서로 다른 객체 인코딩 방식을 개발해왔다](https://en.wikipedia.org/wiki/Comparison_of_data_serialization_formats).

### 타협(Trade-offs)하기

이 구조체들은 이미 내부적으로 바이트 형태의 인메모리로 표현되기 때문에 처음엔 논리적 구조체를 바이트로 변환하는게 충분히 간단해 보일 수 있다. 그냥 이 포맷을 사용하면 되지 않나?

왜 Go의 인메모리 포맷이 바이트를 변환해서 디스크에 저장하거나 네트워크를 통해 전송하는데에 적합하지 않은지에 대한 이유는 다양하다. 첫번째로 호환성이다. Go의 내부 데이터 구조 포맷은 Java의 내부 포맷과 맞지 않기에 우리는 서로 다른 시스템간 통신을 할 수가 없다. 가끔 우리는 프로그래밍 언어가 아닌 사람과의 호환성이 필요하다. [CSV](https://en.wikipedia.org/wiki/Comma-separated_values), [JSON](http://www.json.org/), 그리고 [XML](https://en.wikipedia.org/wiki/XML)은 모두 사람이 읽을 수 있는 포맷이며 보거나 수정하기가 쉽다.

사람이 읽기 가능한 포맷을 만드는것은 트레이드 오프를 이끌어낸다. 사람이 분석하기 쉬운 포맷은 컴퓨터가 분석하기에는 느리다. 정수가 좋은 예이다. 사람은 정수를 10진법으로 읽지만 컴퓨터는 2진법으로 동작한다. 사람은 또한 1또는 1,000과 같은 가변 길이의 숫자를 읽을 수 있지만 컴퓨터는 32비트나 64비트의 정수와 같은 고정 크기의 숫자를 가지고 동작한다. 성능이 숫자 하나에 대해선 별로 차이가 없어 보일 수 있지만 수백만, 수십억개의 숫자를 분석할 때에는 그 차이가 빠르게 벌어진다.

또한 우리가 처음에 생각하지 않은 다른 트레이드 오프도 있다. 데이터 구조는 시간이 지나면서 변하지만 우리는 여전히 오래전에 인코딩된 바이트 위에서 동작을 시켜야 할 필요가 있다. [프로토콜 버퍼(Protocol Buffers)](https://developers.google.com/protocol-buffers/)와 같은 몇몇 인코딩은 여러분의 데이터와 필드의 버전(새로운 필드가 추가될 수 있는 반면, 이전 필드가 더 이상 사용되지 않을 수 있다)에 대한 스키마를 작성할 수 있도록 해준다. 이것의 단점은 객체를 인코딩하고 디코딩 하기위해선 스키마 정의가 필요하다는 것이다. Go의 자체적인 [gob](https://golang.org/pkg/encoding/gob/) 포맷은 다른 방법을 택하는데 실제로 인코딩시 스키마 포맷을 포함한다. 그러나, 이 방법의 단점은 인코딩 사이즈가 매우 커질 수 있다는 것이다.

일부 포맷은 전적으로 주의를 기울여야 하고 유연한 스키마를 지향한다. [JSON](http://www.json.org/)과 [MessagePack](http://msgpack.org/index.html)는 여러분이 바로 구조체를 인코딩 하도록 해주지만 기존의 포맷에서 구조체를 안전하게 디코딩 하는것에 대한 보장을 제공해주진 않는다.

우리는 또한 따로 인코딩을 생각하지 않아도 우리를 위해 인코딩을 대신 수행해주는 시스템을 사용한다. 예를 들면, 데이터베이스는 우리의 논리적 데이터 구조를 가지고 이들을 디스크에 바이트로 영구 저장하는 우회적인 방법이다. 이는 네트워크 호출, SQL 파싱, 그리고 쿼리 계획을 포함 할 수 있지만 이들은 모두 기본적으로 인코딩이다.

마지막으로, 만약 여러분이 다른 것들보다 정말로 속도가 필요하다면, 데이터 저장을 위해 Go의 내부 포맷을 사용할 수 있다. 심지어 나는 이를 위해 [raw](https://github.com/boltdb/raw)라는 라이브러리를 만들기도했다. 이것의 인코딩과 디코딩 시간은 말그대로 0초다. 여러분은 이걸 프로덕션 환경에서 사용해야 하는가? 아마 아닐 것이다.

<br>

## 4가지 인코딩 인터페이스

만약 여러분이 [encoding](https://golang.org/pkg/encoding/) 패키지를 들여다본 몇 안되는 사람들 중 한 명이라면, 조금 실망했을 것이다. 이는 [errors](https://golang.org/pkg/errors/) 패키지 다음으로 두번째로 가장 작은 패키지이며 단 4개의 인터페이스만 가지고있다.

처음에 살펴볼 두 개의 인터페이스는 [BinaryMarshaler](https://golang.org/pkg/encoding/#BinaryMarshaler)와 [BinaryUnmashaler](https://golang.org/pkg/encoding/#BinaryUnmarshaler)이다:

```go
type BinaryMarshaler interface {
        MarshalBinary() (data []byte, err error)   
}

type BinaryUnmashaler interface {
        UnmarshalBinary (data []byte) error
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error : [error](https://golang.org/pkg/errors/)

이들은 객체를 바이너리 포맷으로 변환하거나 역변환하는 방법을 제공한다. 이는 time.[Time](https://golang.org/pkg/time/#Time).[MarshalBinary](https://golang.org/pkg/time/#Time.MarshalBinary)와 같은 표준 라이브러리에서 몇군데 사용된다. 여러분은 이를 많이 보진 못할텐데 보통 객체를 바이너리 포맷으로 마샬링하는 방법이 단일하게 정의되어 있지 않기 때문이다. 보다시피, 수많은 serialization 포맷이 있다.

그러나, 애플리케이션 레벨에서 여러분은 아마 마샬링을 위해 하나의 포맷만 선택할 것이다. 예를 들면, 여러분은 여러분의 모든 데이터에 대해 프로토콜 버퍼를 선택 했을 수 있다. 일반적으로 애플리케이션 데이터를 위해 여러개의 바이너리 포맷을 지원할 이유가 없으므로 [BinaryMarshaler](https://golang.org/pkg/encoding/#BinaryMarshaler)를 구현하는건 의미가 있다.

다음으로 살펴볼 두 인터페이스는 [TextMarshaler](https://golang.org/pkg/encoding/#TextMarshaler)와 [TextUnmarshaler](https://golang.org/pkg/encoding/#TextUnmarshaler)이다:

```go
type TextMarshaler interface {
        MarshalText() (text []byte, err error)
}

type TextUnmarshaler interface {
        UnmarshalText(text []byte) error
}
```
> byte : [byte](https://golang.org/pkg/builtin/#byte), error : [error](https://golang.org/pkg/errors/)

이 두 인터페이스는 출력값이 UTF-8 포맷인걸 제외하고는 바이너리 마샬링 인터페이스와 유사하다.

몇몇 포맷은 json.[Marshaler](https://golang.org/pkg/encoding/json/#Marshaler)와 같이 동일한 네이밍 스타일을 따르는 자체적인 마샬링 인터페이스 가지고 있다.

<br>

## 인코딩 패키지 개요

표준 라이브러리에는 많은 유용한 인코딩 패키지들이 있다. 우리는 이들의 자세한 내용들은 나중 포스트에서 다룰 것이지만 일단 개요를 살펴보고자 한다. 이들 중 몇몇은 *encoding* 의 서브패키지인 반면 그 외에는 다른 곳에 뿔뿔이 흩어져 있다.

### 프리미티브 인코딩

Go를 시작할 때 처음으로 사용하게될 패키지는 아마 [fmt](https://golang.org/pkg/fmt/) 패키지일 것이다. ("fumpt"라 발음한다.) 이는 숫자, 문자열, 바이트, 그리고 일부 지원되는 객체 인코딩까지 포함한 것들을 인코딩 및 디코딩 하기 위해 C 스타일의 [printf](http://pubs.opengroup.org/onlinepubs/009695399/functions/fprintf.html)() 컨벤션을 사용한다. [fmt](https://golang.org/pkg/fmt/) 패키지는 템플릿으로부터 사람이 쉽게 읽을 수 있는 문자열을 훌륭하고 쉽게 만들 수 있는 방법을 제공하지만 템플릿 파싱은 추가적인 오버헤드를 발생시킬 수 있다.

만약 더 나은 성능이 필요하다면 문자열 변환 패키지인 [strconv](https://golang.org/pkg/strconv/)을 사용함으로써 템플레이팅을 피할 수 있다. 이는 기본적인 포맷팅 그리고 문자열, 정수, 실수, 그리고 부울을 위한 스캐닝을 제공하는 로우 레벨 패키지이며 매우 빠르다.

Go 자체와 함께 이 패키지들은 여러분이 UTF-8의 문자열을 인코딩함을 가정한다. 표준 라이브러리에서 유니코드가 아닌 문자의 인코딩 지원이  미약한건 지난 수년간에 걸쳐 UTF-8의 표준이 인터넷의 많은 부분을 빠르게 지배하고 있기 때문에 가능하거나 [Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike)가 Go와 UTF-8의 공동개발자이기 때문에 가능한일 일 것이다. 누가 아는가? 나는 운좋게도 여태까지 Go에서 비 UTF-8의 인코딩을 처리할 필요가 전혀 없었다. 그러나, [unicode/utf16](https://golang.org/pkg/unicode/utf16/), [encoding/ascii85](https://golang.org/pkg/encoding/ascii85/), 그리고 [golang.org/x/text](https://godoc.org/golang.org/x/text) 패키지 트리에 몇가지 인코딩 지원이 있긴하다. *"x"* 패키지 트리는 Go 프로젝트의 일부로 엄청난 패키지들을 많이 포함하고 있지만 [Go 1 호환성 요건](https://golang.org/doc/go1compat)으로는 적용되지 않는다. 

정수 인코딩을 위해, [encoding/binary](https://golang.org/pkg/encoding/binary/) 패키지는 큰 엔디안과 작은 엔디안 인코딩뿐만 아니라 가변 길이의 인코딩도 제공한다. 엔디안이란 바이트가 디스크에 쓰여지는 순서를 가리키는 말이다. 예를 들면, *1,000* (16진법으로는 *0x03E8*) 의 *uint16* 표현식은 두 바이트 *03* 과*E8* 의 조합으로 이루어진다. 큰 엔디안 인코딩에서는 바이트가 "03 E8"의 순서로 쓰여진다. 작은 엔디안에서는 "E8 03"으로 순서가 뒤바뀐다. 많은 일반적인 CPU 아키텍처는 작은 엔디안을 사용한다. 그러나, 큰 엔디안은 보통 네트워크를 통해 바이트를 전송할 때 사용된다. 큰 엔디안은 그래서 *(네트워크 바이트 순서) network byte order* 라고도 한다.

마지막으로, 바이트 인코딩을 위해 사용할 수 있는 패키지 쌍이 있다. 바이트 인코딩은 보통 바이트를 출력가능한 포맷으로 변환하는데 사용된다. 예를 들면, [encoding/hex](https://golang.org/pkg/encoding/hex/) 패키지는 바이너리 데이터를 16진법으로 볼 필요가 있을때 사용될 수 있다. 나는 개인적으로 디버깅 목적으로만 사용해봤다. 반면, 가끔은 역사적으로 제한된 바이너리 지원 (예로 이메일이 있다.)을 가지고 프로토콜 위에서 데이터를 전송해야하기 때문에 출력가능한 포맷이 필요할 때도 있다. [encoding/base32](https://golang.org/pkg/encoding/base32/)과 [encoding/base64](https://golang.org/pkg/encoding/base64/) 패키지는 이의 한 예이다. 또 다른 예시는 TLS 인증서를 인코딩 하기 위해 사용되는 [encoding/pem](https://golang.org/pkg/encoding/pem/) 패키지가 있다.

### 객체 인코딩

우리는 표준 라이브러리에서 객체 인코딩을 위한 몇개의 패키지들을 찾았다. 그러나 실제로 이 패키지들은 우리가 필요로 하는 모든 것들을 가지고 있다.

만약 여러분이 지난 10년간 세상을 등지고 살아왔다면, 아마 [JSON](http://www.json.org/)이 인터넷의 기본 객체 인코딩이 되었다는 것을 알아챘을 것이다. 위에서도 언급했듯이, JSON은 결점을 가지고 있지만 사용하기가 쉬우며 모든 언어에서 라이브러리가 지원 되기 때문에 채택이 급증했다. [encoding/json](https://golang.org/pkg/encoding/json/) 패키지는 이 프로토콜을 위한 훌륭한 지원을 제공하며 [ffjson](https://github.com/pquerna/ffjson)과 같은 빠른 파서를 생성하기 위한 서드파티 구현체들 또한 존재한다.

JSON이 머신 사이의 프로토콜로서 지배하고 있는 동안, [CSV](https://en.wikipedia.org/wiki/Comma-separated_values) 포맷은 사람들에게 데이터를 내보내기 위한 더 일반적인 프로토콜이다. [encoding/csv](https://golang.org/pkg/encoding/csv/) 패키지는 테이블 형식의 데이터를 이 포맷으로 내보내기 위한 좋은 인터페이스를 제공한다.

만약 여러분이 약 2000년경에 구축된 시스템과 상호 작용을 하는 경우엔 아마 [XML](https://en.wikipedia.org/wiki/XML)을 사용해야 할 것이다. [encoding/xml](https://golang.org/pkg/encoding/xml/) 패키지는 [json](https://golang.org/pkg/encoding/json/) 패키지와 유사한 추가적인 태그 기반 marshaler/unmarshaler를 가진 [SAX](https://en.wikipedia.org/wiki/Simple_API_for_XML) 스타일의 인터페이스를 제공한다. 만약 DOM, XPath, XSD, 또는 XSLT같은 좀 더 복잡한 기능들을 찾고 있다면 아마 cgo를 통한 [libxml2](http://xmlsoft.org/)을 사용해야 할 것이다.

Go는 또한 [gob](https://golang.org/pkg/encoding/gob/)라고 하는 자체적인 스트림 인코딩을 가지고 있다. 이 패키지는 두 Go 서비스간의 원격 프로시저 호출을 구현하기 위한 [net/rpc](https://golang.org/pkg/net/rpc/) 패키지에의해 사용된다. Gob은 사용하기가 쉽다. 그러나 이는 그 어떤 언어 크로스도 지원하지 않는다. 만약 서로 다른 언어간 통신이 필요하다면 [gRPC](http://www.grpc.io/)가 인기있는 대안이 될 수 있을 것 같다.

마지막으로, [encding/asn1](https://golang.org/pkg/encoding/asn1/)라는 패키지가 있다. 문서에는 제한된 정보만 있고 패키지에는 오직 25페이지의 텍스트로 이루어진 [layman의 ANS.1 가이드](http://luca.ntop.org/Teaching/Appunti/asn1.html)에 대한 링크만 있다. ANS.1은 특히 SSL/TLS의 X.509 인증서에 많이 사용되는 복잡한 객체 인코딩 스키마이다.

<br> 

## 결론

인코딩은 바이트 위에서 정보를 레이어링 하기위한 기초적인 기반을 제공한다. 이게 없다면 우린 문자열이나 데이터 구조나 데이터베이스 또는 그 어떤 유용한 애플리케이션도 가질 수 없을 것이다. 상대적으로 간단한 개념처럼 보이는것이 많은 구현의 역사와 다양한 트레이드 오프를 가지고 있다.

이 포스트에서 우리는 표준 라이브러리에 있는 다양한 인코딩 구현의 개요와 그것들의 트레이드 오프들을 살펴봤다. 우리는 이러한 프리미티브와 객체 인코딩 패키지가 우리의 바이트 스트림과 슬라이스의 지식선상에서 어떻게 구현되는지를 보았다. 다음 몇 개의 포스트에선 실제 상황에서 이들을 어떻게 사용할 수 있는지를 알아보기 위해 이 패키지들을 좀 더 깊이 파헤쳐 볼 것이다.
