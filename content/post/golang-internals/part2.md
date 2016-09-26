+++
title = "Golang의 내부, 2부: Go 컴파일러 들여다 보기"
draft = false
date = "2016-09-15T05:53:48-04:00"

tags = ["Golang", "Internals", "Compiler", "Structure"]
categories = ["번역", "핵킹"]
series = ["Golang  Internals"]
authors = ["Jhonghee Park"]

toc = true
+++

독자는 인터페이스 레퍼런스를 통해 변수를 사용할 경우 Go 런타임내에서 어떤 일이 있는지 정확하게 알고 있는가? 이 질문에 쉽게 답할 수 없는 이유는 어떤 인터페이스를 구현하는 타입의 경우 그 인터페이스를 가리키는 어떤 레퍼런스도 갖고 있지 않기 때문이다. 하지만 여전히 시도는 해 볼 수 있는데 [이전 블로그 포스트](/post/golang-internals/part1/)에서 논했던 Go 컴파일러의 지식을 이용하는 것이다.

그러면, Go 컴파일러속으로 잠수해 들어가자: 간단한 Go 프로그램을 제작하고 Go 타입캐스팅(typecasting)이 내부적으로 어떻게 동작하는 지 살펴보겠다. 이 것을 예로 들면서, 어떻게 노드 트리가 생성되고 사용되는지 설명하겠다. 이렇게 함으로써 독자도 이 지식을 다른 Go 컴파일러 기능에 적용할 수 있을 것이다.

# 시작하기 전에

실험을 하기 전에, (Go 툴을 쓰는 것이 아니라) Go 컴파일러를 직접 사용할 필요가 있다. 다음 명령을 사용해 이 기능에 접근할 수 있다:

>```
go tool 6g test.go
```

이 명령으로 *test.go* 소스파일은 컴파일되고 오브젝트 파일(object file)이 만들어 진다. 여기서 *6g* 는 AMD64 아키텍쳐인 저자의 머신을 위한 컴파일러의 이름이다. 다른 아키텍쳐에서는 상응하는 컴파일러를 사용해야 함을 주목하라.

컴파일러를 직접 사용할 때 유용한 커맨드 라인 인수들을 사용할 수 있는데, 자세한 내용은 [여기](https://golang.org/cmd/gc/#hdr-Command_Line)를 참고하라. 이 실험을 위해서, 노드 트리의 레이아웃을 출력해 주는 *-W* 플래그를 사용하겠다.

# 간단한 Go 프로그램 만들기

우선 간단한 Go 프로그램을 만들자. 저자의 버전은 다음과 같다:

>```go
  1  package main
  2
  3  type I interface {
  4          DoSomeWork()
  5  }
  6
  7  type T struct {
  8          a int
  9  }
 10
 11  func (t *T) DoSomeWork() {
 12  }
 13
 14  func main() {
 15          t := &T{}
 16          i := I(t)
 17          print(i)
 18  }
```

정말 간단하지 않은가? 불필요하게 생각되는 단 하나는 17째 줄인데, *i* 변수를 출력하는 부분이다. 불필요하다고 판단됨에도 불구하고 이 줄이 없다면, *i* 는 사용하지 않은 변수로 간주되어 프로그램은 컴파일 되지 않을 것이다. 다음 단계는 이 프로그램을 *-W* 를 사용해 컴파일 하는 것이다:

>```
go tool 6g -W test.go
```

이 명령을 실행한 후에, 프로그램내 정의된 각 메서드에 해당하는 노드 트리를 포함한 출력을 보게 될 것이다. 이 경우, *main* 과 *init* 메서드가 있다. *init* 메서드가 언급된 이유는 모든 프로그램에 암시적으로 정의되어 있기 때문인데, 실제로 여기에서는 다루지 않겠다.

각 메서드마다, 컴파일러는 두개의 노드트리 버전을 출력한다. 첫번째는 소스파일을 파싱하고 얻는 노드 트리의 원본이고 두번째는 타입체킹후 모든 필요한 수정을 거친 버전이다.

# main 메서드의 노드 트리에 대한 이해

우선 main 메서드에서 나온 노드 트리의 원본을 자세히 들여다 보고 정확하게 무슨 일이 일어나고 있는지를 이해하기 위한 시도를 해보자.

>```
DCL l(15)
.   NAME-main.t u(1) a(1) g(1) l(15) x(0+0) class(PAUTO) f(1) ld(1) tc(1) used(1) PTR64-*main.T

AS l(15) colas(1) tc(1)
.   NAME-main.t u(1) a(1) g(1) l(15) x(0+0) class(PAUTO) f(1) ld(1) tc(1) used(1) PTR64-*main.T
.   PTRLIT l(15) esc(no) ld(1) tc(1) PTR64-*main.T
.   .   STRUCTLIT l(15) tc(1) main.T
.   .   .   TYPE <S> l(15) tc(1) implicit(1) type=PTR64-*main.T PTR64-*main.T

DCL l(16)
.   NAME-main.i u(1) a(1) g(2) l(16) x(0+0) class(PAUTO) f(1) ld(1) tc(1) used(1) main.I

AS l(16) tc(1)
.   NAME-main.autotmp_0000 u(1) a(1) l(16) x(0+0) class(PAUTO) esc(N) tc(1) used(1) PTR64-*main.T
.   NAME-main.t u(1) a(1) g(1) l(15) x(0+0) class(PAUTO) f(1) ld(1) tc(1) used(1) PTR64-*main.T

AS l(16) colas(1) tc(1)
.   NAME-main.i u(1) a(1) g(2) l(16) x(0+0) class(PAUTO) f(1) ld(1) tc(1) used(1) main.I
.   CONVIFACE l(16) tc(1) main.I
.   .   NAME-main.autotmp_0000 u(1) a(1) l(16) x(0+0) class(PAUTO) esc(N) tc(1) used(1) PTR64-*main.T

VARKILL l(16) tc(1)
.   NAME-main.autotmp_0000 u(1) a(1) l(16) x(0+0) class(PAUTO) esc(N) tc(1) used(1) PTR64-*main.T

PRINT l(17) tc(1)
PRINT-list
.   NAME-main.i u(1) a(1) g(2) l(16) x(0+0) class(PAUTO) f(1) ld(1) tc(1) used(1) main.I
```

아래 설명할 때는 불필요한 부분을 모두 제거한 요약한 버전을 사용하겠다.

첫번째 노드는 꽤 단순하다.

>```
DCL l(15)
.   NAME-main.t l(15) PTR64-*main.T
```

첫번째 노드는 선언 노드이다. *l(15)* 는 노드가 줄 15에 정의되어 있음을 알려준다. 선언 노드는 *main.t* 변수를 나타내는 이름 노드(name node)에 레퍼런스를 갖는다. 변수가 정의된 곳은 main 패키지이고 실제로 *main.T* 타입를 가리키는 64비트 포인터이다. 15째 줄을 보면 어떤 선언이 되어 있는지 쉽게 이해할 수 있다.

다음 것은 약간 까다롭다.

>```
AS l(15)
.   NAME-main.t l(15) PTR64-*main.T
.   PTRLIT l(15) PTR64-*main.T
.   .   STRUCTLIT l(15) main.T
.   .   .   TYPE l(15) type=PTR64-*main.T PTR64-*main.T
```

최상위 노드(root node)는 할당(assignment) 노드이다. 첫번째 자식노드는 이름 노드(name node)로 *main.t* 변수를 대표한다. 두번째 자식노드는 *main.t* 에 할당되는, 포인터 리터럴 노드이다: &를 생각하라. 이 노드는 struct 리터럴 노드를 자식으로 갖고 있고, 그 노드는 또 실제 타입인 (*main.T*)를 대표하는 타입 노드를 포인터로 가리킨다.

다음 노드는 또 다른 선언이다. 이번에는 *main.I* 타입에 속하는 *main.i* 변수의 선언이다.

>```
DCL l(16)
.   NAME-main.i l(16) main.I
```

그런 다음, 컴파일러는 또 다른 변수, *autotmp_0000* 를 만들고, *main.t* 변수를 할당한다.

>```
AS l(16) tc(1)
.   NAME-main.autotmp_0000 l(16) PTR64-*main.T
.   NAME-main.t l(15) PTR64-*main.T
```

마침내, 흥미로운 노드들에 도착했다.

>```
AS l(16)
.   NAME-main.i l(16)main.I
.   CONVIFACE l(16) main.I
.   .   NAME-main.autotmp_0000 PTR64-*main.T
```

여기를 보면, 컴파일러가 *CONVIFACE* 라고 불리는 특별한 노드를 *main.i* 에 할당하는 것을 볼 수 있다. 하지만 이것만으로는 실제로 내부에서 어떤 일이 일어나는지 알 수 없다. 한가지 방법은 모든 모드 트리 수정이 적용되고 난 후에 main 메서드의 노드 트리속을 들여다 보는 것인데, 출력된 내용 중 "after walk main" 라는 섹션내의 정보를 통해 알아보는 것이다.

# 컴파일러는 어떻게 할당노드를 번역하는가

아래를 보면 컴파일러가 어떻게 할당 노드(assignment node)를 번역하는 지 알 수 있다:

>```
AS-init
.   AS l(16)
.   .   NAME-main.autotmp_0003 l(16) PTR64-*uint8
.   .   NAME-go.itab.*"".T."".I l(16) PTR64-*uint8

.   IF l(16)
.   IF-test
.   .   EQ l(16) bool
.   .   .   NAME-main.autotmp_0003 l(16) PTR64-*uint8
.   .   .   LITERAL-nil I(16) PTR64-*uint8
.   IF-body
.   .   AS l(16)
.   .   .   NAME-main.autotmp_0003 l(16) PTR64-*uint8
.   .   .   CALLFUNC l(16) PTR64-*byte
.   .   .   .   NAME-runtime.typ2Itab l(2) FUNC-funcSTRUCT-(FIELD-
.   .   .   .   .   NAME-runtime.typ·2 l(2) PTR64-*byte, FIELD-
.   .   .   .   .   NAME-runtime.typ2·3 l(2) PTR64-*byte PTR64-*byte, FIELD-
.   .   .   .   .   NAME-runtime.cache·4 l(2) PTR64-*PTR64-*byte PTR64-*PTR64-*byte) PTR64-*byte
.   .   .   CALLFUNC-list
.   .   .   .   AS l(16)
.   .   .   .   .   INDREG-SP l(16) runtime.typ·2 G0 PTR64-*byte
.   .   .   .   .   ADDR l(16) PTR64-*uint8
.   .   .   .   .   .   NAME-type.*"".T l(11) uint8

.   .   .   .   AS l(16)
.   .   .   .   .   INDREG-SP l(16) runtime.typ2·3 G0 PTR64-*byte
.   .   .   .   .   ADDR l(16) PTR64-*uint8
.   .   .   .   .   .   NAME-type."".I l(16) uint8

.   .   .   .   AS l(16)
.   .   .   .   .   INDREG-SP l(16) runtime.cache·4 G0 PTR64-*PTR64-*byte
.   .   .   .   .   ADDR l(16) PTR64-*PTR64-*uint8
.   .   .   .   .   .   NAME-go.itab.*"".T."".I l(16) PTR64-*uint8
AS l(16)
.   NAME-main.i l(16) main.I
.   EFACE l(16) main.I
.   .   NAME-main.autotmp_0003 l(16) PTR64-*uint8
.   .   NAME-main.autotmp_0000 l(16) PTR64-*main.T
```

출력을 통해 볼 수 있듯이, 컴파일러는 우선 초기화 노드 리스트 (*AS-init*) 를 할당 노드에 첨가한다. *AS-init* 내에서는, *go.itab.\*””.T.””*.I 변수 값을 새로 만든 변수 *main.autotmp_0003* 에 할당한다. 그런 다음, 변수가 *nil* 인지를 검사한다. 만약에 *nil* 이면, 컴파일러는 *runtime.typ2Itab* 함수를 다음의 인수들을 사용해 호출한다:

 * *main.T* 타입의 포인터,
 * *main.I* 인터페이스 타입의 포인터,
 * *go.itab.\*””.T.””.I* 변수를 가리키는 포인터.

코드에서 보면, 이 변수가 *main.T* 에서 *main.I* 로 타입변환된 결과를 저장하는데 사용되고 있음이 명백하다.

# *getitab* 메서드의 내부

논리적인 다음 단계는 *runtime.typ2Itab* 를 찾아보는 것이다. 아래에 이 함수를 나열해 놓았다.

>```
func typ2Itab(t *_type, inter *interfacetype, cache **itab) *itab {
	tab := getitab(inter, t, false)
	atomicstorep(unsafe.Pointer(cache), unsafe.Pointer(tab))
	return tab
}
```

확실해 보이는 것은 실제 작업은 *getitab* 메서드 내부에서 진행된다는 점인데, 두번째 줄에서 단순히 tab 변수를 cache 변수에 저장하는 점을 통해 알 수 있다. 그럼 *getitab* 내부를 들여다 보자. 꽤 긴 내용이라 가장 들여다 볼 가치가 있는 부분만 복사했다.

>```
m =
    (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*ptrSize, 0,
    &memstats.other_sys))
    m.inter = interm._type = typ

ni := len(inter.mhdr)
nt := len(x.mhdr)
j := 0
for k := 0; k < ni; k++ {
	i := &inter.mhdr[k]
	iname := i.name
	ipkgpath := i.pkgpath
	itype := i._type
	for ; j < nt; j++ {
		t := &x.mhdr[j]
		if t.mtyp == itype && t.name == iname && t.pkgpath == ipkgpath {
			if m != nil {
				*(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*ptrSize)) = t.ifn
			}
		}
	}
}
```

우선 결과를 저장할 메모리를 할당한다.

>```
(*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*ptrSize, 0, &memstats.other_sys))
```

Go 언어에서 왜 메모리를 할당하는지 그리고 왜 이렇게 이상한 방식으로 하는지 궁금해 진다. 이 질문에 답하기 위해 *itab* struct 정의를 들여다 보아야 할 필요가 있다.

>```
type itab struct {
	inter  *interfacetype
	_type  *_type
	link   *itab
	bad    int32
	unused int32
	fun    [1]uintptr // variable sized
}
```

마지막 특성인 *fun* 은 요소가 하나인 배열로 정의되어 있다. 하지만 실제로 배열의 크기는 변할 수 있다고 코멘트되어 있다. 나중에 보겠지만 이 특성은 특정한 타입내 정의된 메서드들을 가리키는 포인터 배열를 가지고 있다. 이 메서드들은 인터페이스 타입의 메서드에 상응한다. Go 언어의 저자들은 이 특성을 위해 동적 메로리 할당을 사용한다. (그렇다, unsafe 패키지를 사용하면 이런 일들이 가능하다.) 메모리를 얼마나 할당해야 하는지는 struct 자체의 크기에 인터페이스내 메서드의 숫자에 포인터 크기를 곱한 값을 더해 계산할 수 있다.

>```
unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*ptrSize
```

다음으로 두개의 중첩된 루프를 볼 수 있다. 첫째로 인터페이스의 모든 메서드를 차례로 처리한다. 인터페이스의 각 메서드에 대해서 특정한 타입내에 상응하는 메서드를 찾으려 한다. (메서드는 *mhdr* 컬렉션에 저장되어 있다.) 두 메서드가 서로 같은지를 비교하는 과정은 굳이 설명할 필요가 없겠다.

>```
if t.mtyp == itype && t.name == iname && t.pkgpath == ipkgpath
```

만약 매치를 찾으면, 결과값의 *fun* 에 메서드를 가리키는 포인터를 저장한다:

>```
*(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*ptrSize)) = t.ifn
```

성능에 대해 짧게 언급하면: 인터페이스와 사전에 설정된 타입 정의에 대해 메서드는 알파벳 순서로 정렬되어 있어서, 이 중첩 루프는 *O(n * m)* 대신 *O(n + m)* 으로 반복한다. n 과 m 은 상응하는 메서드 숫자들

마지막으로, 할당의 마지막 부분을 기억하는가?


>```
AS l(16)
.   NAME-main.i l(16) main.I
.   EFACE l(16) main.I
.   .   NAME-main.autotmp_0003 l(16) PTR64-*uint8
.   .   NAME-main.autotmp_0000 l(16) PTR64-*main.T
```

여기에서 *EFACE* 노드를 *main.i* 변수에 할당한다. *EFACE* 노드는 *main.autotmp_0003* 와 *main.autotmp_0000* 변수들에 대한 레퍼런스를 가지고 있는데 - *main.autotmp_0003* 는 *runtime.typ2Itab* 에 의해 반환된 itab struct를 가리키는 포인터이고, *main.autotmp_0000* 변수는 *main.t* 와 같은 값을 가지고 있다. 인터페이스 레퍼런스를 통해 메서드를 호출하는데 필요한 것은 이게 전부이다.

그래서, *main.i* 변수는 런타임 [runtime](https://godoc.org/runtime) 패키지내 정의된 [iface](https://golang.org/src/runtime/iface.go) struct의 인스턴스를 가지고 있다.

>```
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
```

# 다음에 살펴볼 내용은?

저자가 지금까지 Go 컴파일러와 런타임에 대해 아주 작은 부분만 설명했다는 점을 이해한다. 얘기해 볼만한 흥미로운 주제들이 여전히 많이 남아있다. 예를 들면, 오브젝트 파일, 링커, 재배치(relocations), 등에 대해서는 다음 블로그 포스트에서 살펴보기로 하겠다.

* 원문: [Golang Internals, Part 2: Diving Into the Go Compiler](http://blog.altoros.com/golang-internals-part-2-diving-into-the-go-compiler.html)
* 저자: Siarhei Matsiukevich
* 번역자: Jhonghee Park
