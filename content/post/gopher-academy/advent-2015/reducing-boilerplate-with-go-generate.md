+++
date = "2016-10-16T01:03:50+09:00"
title = "Reducing boilerplate with go generate"
tags = [
  "golang",
  "read",
  "번역",
]
categories = [
  "번역",
]
authors = [
  "Timo Boll",
]
series = [
  "Advent 2015"
]
draft = false
toc = false
+++


> [Reducing boilerplate with go generate][original-blog-url] 를 번역한 글입니다

Go는 대단한 언어입니다. 단순하고, 파워풀하며, 훌륭한 도구들을 가지고 있고, 우리 중 많은 
이들은 매일 사용하는 것을 즐깁니다. 하지만 강한 타입의 언어들에서 일상적으로 발생하게 되는, 이것저것을 연결하기 위해서
필수로 사용해야 하는 boilerplate를 쓰게 됩니다.

이 포스트에서 다음의 세가지 포인트를 다룰 것입니다:

1. 코드 생성(code generation)을 사용하여 boilerplate를 줄이도록 도와주는 Go 도구들을 만들 수 있어야 하는 이유는 무엇입니까.
2. Go생성 시 코드 생성을 위한 블록 구성 요소는 무엇입니까.
3. 코드 생성 도구를 배우기 위한 예제는 어디에서 찾을 수 있습니까.

# boilerplate를 줄이기 위해 코드 생성을 사용하는 이유는 무엇입니까?

때때로 우리는 reflection을 쓰고 `interface{}`를 받아들이는 메서드들을 프로젝트에 채움으로 boilerplate를 줄이려고 노력합니다.
그러나 메서드가 `interface{}`를 받아들일 때마다, 우리는 type 안정성을 창밖으로 던져버립니다.
type assertions와 reflection을 사용할 때, 컴파일러는 우리가 올바른 타입들을 패싱하는지 확인할 수 없으며, 런타임 panic에 더욱 노출됩니다.

우리가 만들어놓은 boilerplate 코드의 몇몇은 우리의 프로젝트에서 이미 가지고 있는 코드로 부터 추론될 수 있습니다.
그로 인해, 우리는 프로젝트의 코드를 읽어서 relevant 코드를 생성하는 도구들을 제작할 수 있습니다.

# 코드 생성을 위한 building blocks

## 코드 읽어들이기

기본 라이브러리는 코드를 읽고 파싱할때 무거운 작업들을 들어올릴 준비가 되어 있는 훌륭한 패키지를 가지고 있습니다.

* `go/build`: go 패키지에 대한 정보를 수집합니다. 패키지 이름이 주어지면, 소스코드를 포함하고 있는 디렉토리가 무엇인지,
  디렉토리안의 코드와 테스트 파일이 무엇인지, 의존성이 있는 다른 패키지는 무엇인지 등등.
* `go/scanner` `go/parser`: 소스코드를 읽고 파싱하여 [Abstract Syntax Tree][ast] (AST)를 생성합니다.
* `go/ast`: AST를 표현하는데 사용되는 타입들을 선언하고 tree를 동작하고 변경하는데 도움을 주는 메서드들을 포함합니다.
* `go/types`: 데이터 타입들을 선언하고 Go 패키지 타입 체킹을 위해 사용되는 알고리즘을 구현합니다.
  `go/ast` 가 raw tree를 포함하고 있는데 비해 이 패키지는 AST를 프로세싱하기 위한 모든 작업을 수행하여 타입들에 대한 정보를 바로 얻을 수 있습니다.

## 코드 생성하기

코드를 생성할 때, 대부분의 프로젝트들을 단지 좋은 옛 `text/template` 에 의존하여 코드를 생성합니다.

자동으로 생성되는 파일에는 코드가 자동으로 생성되었으며, 생성한 도구가 무엇인지, 수작업으로 편집되지 않아야 함을 코멘트로 시작하는 것을 권장합니다.

```go
/*
* CODE GENERATED AUTOMATICALLY WITH github.com/ernesto-jimenez/gogen/unmarshalmap
* THIS FILE SHOULD NOT BE EDITED BY HAND
*/
```

역시 `go/format` 패키지를 이용하여 이를 쓰기 전에 코드를 format 할 수 있습니다.
이 패키지는 `go fmt`가 사용하는 로직을 포함하고 있습니다.

## go generate

프로그램을 위해 소스코드를 생성하는 도구를 작성하기 시작할 때, 두가지의 의문이 재빨리 나타납니다:
우리의 개발 과정에서 코드를 생성하는 시점은 어느 정도입니까? 
생성된 코드를 최신 상태로 유지하려면 어떻게 해야 합니까?

1.4 때부터 go tool 은 `generate` 커맨드를 제공합니다.
이 도구를 사용하면 go tool 자체를 사용하여 코드 생성에 사용하는 도구들을 실행할 수 있습니다.

단지 아래의 포맷으로 코멘트를 작성해 주면 됩니다:

```
//go:generate shell command
```

작성하고 나면, `go generate` 는 실행시마다 항상 자동으로 `command` 를 호출합니다.

기억해야할 중요한 두 포인트가 있습니다:

* `go generate` 는 프로그램이나 패키지를 작성하는 개발자에 의해 실행하도록 의도되어집니다.
  이것은 `go get`에 의해 결코 자동으로 호출되지 않습니다.
* 당신은 `go generate`에 의해 실행되는 모든 도구들을 이미 설치하고 시스템 안에서 setup을 해두어야 합니다.
  어떤 도구를 사용하려고 하고 어디에서 다운로드 받을 수 있는지 문서화해야 합니다.

또한, 만약 당신의 코드 생성 도구가 동일한 repository에 들어 있다면, `go:generate`로부터 
`go run`을 호출하도록 권장합니다. 그러면 도구를 변경하려는 때마다 매번 수동으로 빌드하고 설치하는 작업 없이
`generate` 할 수 있습니다.

# 자신만의 도구를 만들기 시작하는 방법은 무엇입니까?

stdlib 패키지로 코드를 분석하고 생성하는 것은 훌륭하지만, 해당 문서는 거대하고, 패키지를 어떻게 사용해야 할지에 대한
노하우를 단지 문서로 부터 얻는 것은 꽤 벅찹니다.

코드 생성을 시작할 때 내가 했던 가장 좋은 방법은 이미 존재하는 몇 도구들을 배우는 것이었습니다.

1. 빌드할 수 있는 그러한 종류의 도구들로부터 몇몇 영감을 얻을 것입니다.
2. 그 도구들의 소스 코드로 부터 배울 수 있는 기회를 가질 것입니다.
3. 이러한 도구들 중에 스스로 유용한 도구들이 무엇인지 찾아낼 수 있습니다.

# 배우기 위한 프로젝트들

## 인터페이스 구현을 위한 stubs 생성하기

구현하려는 인터페이스에 정의된 메서드 목록을 복사하고 붙여 넣은 자신을 발견한 적이 있습니까?

stubs를 자동으로 생성하기 위해 [`impl`][impl]를 사용할 수 있습니다.
인터페이스를 위해 stdlib 의 패키지를 이용하여 구현해야할 메서드를 출력합니다.


```bash
$ impl 'f *File' io.ReadWriteCloser
func (f *File) Read(p []byte) (n int, err error) {
    panic("not implemented")
}

func (f *File) Write(p []byte) (n int, err error) {
    panic("not implemented")
}

func (f *File) Close() error {
    panic("not implemented")
}
```

## mockery로 mocks 자동 생성하기

[testify][testify]는 유닛테스팅을 실행할 때 쉽게 의존성을 mock할 수 있는 [mock][testify-mock] 패키지를 가지고 있습니다.

인터페이스들이 암시적으로 만족하기 때문에, 의존성들을 인터페이스들을 이용하여 특정화 할 수 있으며, 유닛 테스팅 중에 외부 의존성보다는 mock을 사용할 수 있습니다.

이론적인 downcaser 인터페이스를 mock 하는 방법에 대한 매우 간단한 예제:

```go
package main

import (
  "testing"

  "github.com/stretchr/testify/mock"
)

type downcaser interface {
  Downcase(string) (string, error)
}

func TestMock(t *testing.T) {
  m := &mockDowncaser{}
  m.On("Downcase", "FOO").Return("foo", nil)
  m.Downcase("FOO")
  m.AssertNumberOfCalls(t, "Downcase", 1)
}
```

mock 구현은 꽤 직관적입니다:

```go
type mockDowncaser struct {
  mock.Mock
}

func (m *mockDowncaser) Downcase(a0 string) (string, error) {
  ret := m.Called(a0)
  return ret.Get(0).(string), ret.Error(1)
}
```

실제적으로, 구현체로부터 볼 수 있는 것은, 꽤 직관적이어서 인터페이스 정의 자체가 mock을 자동으로 생성하기 위한 모든 정보를 가지고 있습니다.

[`mockery`][mockery] 이 하는 것:

```
$ mockery -inpkg -testonly -name=downcaser
Generating mock for: downcaser
```

나는 항상 `go generate` 를 사용하여 인터페이스들에 대한 mocks를 자동으로 생성합니다.
우리는 이전 예제에 mock up 과 실행을 위해 단지 한 라인만 추가해 주면 됩니다.

```go
package main

import (
  "testing"
)

type downcaser interface {
  Downcase(string) (string, error)
}

//go:generate mockery -inpkg -testonly -name=downcaser

func TestMock(t *testing.T) {
  m := &mockDowncaser{}
  m.On("Downcase", "FOO").Return("foo", nil)
  m.Downcase("FOO")
  m.AssertNumberOfCalls(t, "Downcase", 1)
}
```

go generatoe 를 실행했을 때 모든 것이 한번에 set up 되는 지 볼 수 있습니다:

```
$ go test
# github.com/ernesto-jimenez/test
./main_test.go:14: undefined: mockDowncaser
FAIL    github.com/ernesto-jimenez/test [build failed]

$ go generate
Generating mock for: downcaser

$ go test
PASS
ok      github.com/ernesto-jimenez/test 0.011s
```

인터페이스 변경을 할 때 마다 `go generate`를 실행하면 해당하는 mock이 업데이트 될 것입니다.

[`mockery`][mockery] 는 내가 [`testify/mock`][testify-mock]에 contribute를
시작한 주된 이유이고 `testify`의 maintainer가 되었습니다.
그렇지만, `go/types` 가 1.5의 표준 라이브러리에 포함되기 이전에 개발되었기 때문에, 저 레벨 `go/ast`을 이용하여 구현되었고,
코드를 보기 어렵게 만들었으며 [failing to generate mocks from interfaces using
composition][mockery-issue] 같은 버그가 나타났습니다.

## gegen 실험

나는 코드 생성에 대해 익히기 위해 [`gogen`][gogen] 패키지에서 만들었던 코드 생성 도구들을 오픈소스화 했습니다.

아래의 세가지 도구들을 포함합니다:

* [goautomock][goautomock]: mockery 와 유사하지만 `go/ast`가 아닌 `go/types`를 이용해 구현되었습니다.
  따라서 composed 인터페이스들에 대해서 역시 동작합니다. 역시 표준 라이브러리로부터 인터페이스를 mock 하기에 용이합니다.
* [gounmarshalmap][gounmarshalmap]: 구조체를 가져 map을 구조체로 디코딩하는 `UnmarshalMap(map[string]interface{})` 함수를 생성합니다.
  reflection 보다는 [`mapstructure`][mapstructure] 의 대안으로 코드 생성에 동작하도록 작성되었습니다.
* [gospecific][gospecific]: `interface{}` 에 의존하는 제네릭으로부터 특정한 패키지를 생성하는 작은 실험입니다.
  제네릭의 패키지 소스코드를 읽어서 `interface{}` 를 사용하는 제네릭 패키지에서 특정한 타입을 사용하는 새로운 패키지를 생성합니다.
  
# 랩핑 업

코드 생성은 대단합니다. 그것은 우리의 프로그램 타입을 안전하게 지키는 동시에 반복적인 코드를 쓸 수 있습니다.
우리는 [Slackline][slackline]을 만들 때 폭넓게 사용하였고 곧 [testify][testify-codegen] 에도 사용할 것입니다.

그럼에도 스스로에게 질문하기를 기억하십시오: 이러한 도구를 작성하는 것이 시간을 아낍니까?

[xkcd][xkcd] 그 대답에 대답하는데 도움이 될 것입니다.
[![](http://imgs.xkcd.com/comics/is_it_worth_the_time.png)][xkcd]

[go-generate-post]: https://blog.golang.org/generate
[xkcd]: https://xkcd.com/1205/
[slackline]: https://slackline.io
[testify-codegen]: https://github.com/stretchr/testify/pull/241
[impl]: https://github.com/josharian/impl
[testify]: https://github.com/stretchr/testify
[testify-mock]: https://godoc.org/github.com/stretchr/testify/mock
[mockery]: https://github.com/vektra/mockery
[mockery-issue]: https://github.com/vektra/mockery/issues/18
[ast]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[gogen]: https://github.com/ernesto-jimenez/gogen
[mapstructure]: https://github.com/mitchellh/mapstructure
[goautomock]: https://github.com/ernesto-jimenez/gogen/tree/master/cmd/goautomock/main.go
[gounmarshalmap]: https://github.com/ernesto-jimenez/gogen/tree/master/cmd/gounmarshalmap
[gospecific]: https://github.com/ernesto-jimenez/gogen/tree/master/cmd/gospecific
[original-blog-url]: https://blog.gopheracademy.com/advent-2015/reducing-boilerplate-with-go-generate/
