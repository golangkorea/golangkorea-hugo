+++

title = "Golang의 내부, 3부: 링커, 오브젝트 파일, 그리고 재배치"
draft = true
date = "2016-09-17T16:20:29-04:00"

tags = ["Golang", "Internals", "linker", "object file", "relocations"]
categories = ["번역"]
series = ["Golang  Internals"]
authors = ["Jhonghee Park"]

toc = true

+++

오늘은 Go 링커와 오브젝트 파일, 그리고 재배치(relocations)에 대해 얘기해 보자.

이런 것들이 독자들과 무슨 상관이 있을까? 만약 독자가 어떤 대형 프로젝트의 내부에 대해 배우고자 한다면, 첫번째 할 일이 그 것을 콤포넌트나 모듈로 자를 필요가 있다. 둘째로 이 모듈들이 서로에게 어떤 인터페이스를 제공하는지 이해할 필요가 있다. Go 언어 프로젝트의 경우, 이런 상위 모듈들이 컴파일러, 링커, 그리고 런타임이다. 컴파일러가 제공하고 링커가 사용하는 것이 오프젝트 파일인데, 오늘은 그 것으로 조사를 시작해 보자.

# Go 오브젝트 파일 생성하기

실용적인 실험을 하나 해 보자-아주 간단한 프로그램을 하나 만들고, 컴파일하고, 어떤 오브젝트 파일이 만들어 지는지 관찰하자. 저자의 경우, 프로그램은 다음과 같다:

>```go
1: package main
2:
3: func main() {
4: 	print(1)
5: }
```

너무 쉽지 않은가? 이제 컴파일을 한다:

>```
go tool 6g test.go
```

이 명령은 *test.6* 오브젝트 파일을 생산한다. 이 파일의 내부 구조를 조사하기 위해, [goobj](https://github.com/golang/go/tree/master/src/cmd/internal/goobj) 라이브러리를 사용하겠다. 이 라이브러리는 내부적으로 Go 소스 코드에 채택되어 주로 유닛 테스트를 구현하는데 쓰인다. 이 유닛 테스트는 여러 상황에서 오브젝트 파일이 정확히 생성되었는지를 테스트한다. 이 블로그 포스트를 위해 *goobj* 라이브러리를 통해 생성된 출력을 콘솔로 프린트하는 매우 간단한 프로그램을 만들었다. 이 프로그램의 소스코드는 [여기](https://github.com/s-matyukevich/goobj_explorer)에서 살펴볼 수 있다.

무엇보다도 우선, 저자의 프로그램을 다운받아 설치해야 한다:

>```
go get github.com/s-matyukevich/goobj_explorer
```

그런 후에 다음의 명령을 실행하라:

>```
goobj_explorer -o test.6
```

이제 *goob.Package* 구조를 콘솔안에서 살펴 볼 수 있을 것이다.

# 오브젝트 파일 조사하기

이 오브젝트 파일에서 가장 흥미로운 부분은 *Syms* 배열이다. 이것은 실제로 심볼 테이블이다. 프로그램안에 정의된 모든 것들, 함수, 전역 변수, 타입, 상수, 등이 이 테이블에 적혀있다. *main* 함수에 상응하는 엔트리에 대해 살펴보자. (Roloc 과 Func 필드는 출력에서 생략되었음을 주목하라. 이 필드들은 나중에 논하겠다.)

>```go
&goobj.Sym{
            SymID: goobj.SymID{Name:"main.main", Version:0},
            Kind:  1,
            DupOK: false,
            Size:  48,
            Type:  goobj.SymID{},
            Data:  goobj.Data{Offset:137, Size:44},
            Reloc: ...,
            Func:  ...,
}
```

*goobj.Sum* 구조내 필드의 이름들은 따로 설명이 필요 없다:

<style type="text/css"><!--
.myTable { background-color:white;border-collapse:collapse; } .myTable th { background-color:#E0E0E0;color:black; } .myTable td, .myTable th { padding:5px;border:1px solid #989898; }
--></style>

<table class="myTable">
<tbody>
<tr>
<th><center>필드</center></th>
<th style="width: 530px;" width="70%"><center>설명</center></th>
</tr>
<tr>
<td><strong>SumID</strong></td>
<td>독특한 심볼 아이디로 심볼의 이름과 버전으로 구성된다. 버전을 통해 동일한 이름에 차이를 부여한다.</td>
</tr>
<tr>
<td><strong>Kind</strong></td>
<td>어떤 종류의 심볼에 속하는지를 나타낸다 (상세한 내용은 나중에).</td>
</tr>
<tr>
<td><strong>DupOK</strong></td>
<td>이 필드는 중복된 이름(같은 이름의 심볼들)이 허락되는지를 나타낸다.</td>
</tr>
<tr>
<td><strong>Size</strong></td>
<td>심볼 데이터의 크기.</td>
</tr>
<tr>
<td><strong>Type</strong></td>
<td>만약 있는 경우, 심볼 타입을 대표하는 또 다른 심볼에 대한 레퍼런스.</td>
</tr>
<tr>
<td><strong>Data</strong></td>
<td>바이너리 데이터를 가진다. 다른 종류의 심볼에 따라 다른 의미를 갖고 있다. 예를 들어, 함수에는 어셈블리 코드를, 문자열 심볼에는 원자재 문자열 콘텐트, 기타 등등.</td>
</tr>
<tr>
<td><strong>Reloc</strong></td>
<td>재배치 리스트 (더 상세한 내용은 나중에 제공될 것이다.)</td>
</tr>
<tr>
<td><strong>Func</strong></td>
<td>함수 심볼에 대한 특별한 함수 메타 데이터를 갖고 있다. (자세한 내용은 아래를 보라).</td>
</tr>
</tbody>
</table>

이제, 다른 종류의 심볼들을 살펴보자. 모든 사용 가능한 종류의 심볼들이 상수로서 *goobj* 패키지 ([여기](https://github.com/golang/go/blob/master/src/cmd/internal/goobj/read.go#L30))에서 찾아 볼수 있)안에 정의되어 있다. 아래에, 이러한 상수들의 첫번째 부분을 복사해 놓았다:

>```
const (
	_ SymKind = iota

	// readonly, executable
	STEXT
	SELFRXSECT

	// readonly, non-executable
	STYPE
	SSTRING
	SGOSTRING
	SGOFUNC
	SRODATA
	SFUNCTAB
	STYPELINK
	SSYMTAB // TODO: move to unmapped section
	SPCLNTAB
	SELFROSECT
	...
```

보다시피, main.main 심볼은 종류 1에 속하고 *STEXT* 상수에 상응한다. *STEXT* 는 실행 가능한 코드를 갖는 심볼이다. 이제, *Reloc* 배열을 살펴보자. 다음과 같은 struct들로 구성되어 있다:

>```go
type Reloc struct {
	Offset int
	Size   int
	Sym    SymID
	Add    int
	Type int
}
```

각 재배치는 *[Offset, Offset+Size]* 간격에 위치한 바이트들이 특정 주소로 교체되어야 함을 암시한다. 이 주소는 *Sym* 심볼의 위치에 *Add* 바이트 숫자를 더하여 계산된다.

# 재배치 이해하기

이제 예를 통해 재배치가 어떻게 작동하는지를 알아보자. 그러기 위해서, *-S* 스위치를 이용해 프로그램을 컴파일 할 필요가 있다. *-s* 스위치는 생성된 어셈블리 코드를 출력할 것이다:

>```
go tool 6g -S test.go
```

어셈블러를 들여다 보면서 main 함수를 찾아보자.

>```
"".main t=1 size=48 value=0 args=0x0 locals=0x8
	0x0000 00000 (test.go:3)	TEXT	"".main+0(SB),$8-0
	0x0000 00000 (test.go:3)	MOVQ	(TLS),CX
	0x0009 00009 (test.go:3)	CMPQ	SP,16(CX)
	0x000d 00013 (test.go:3)	JHI	,22
	0x000f 00015 (test.go:3)	CALL	,runtime.morestack_noctxt(SB)
	0x0014 00020 (test.go:3)	JMP	,0
	0x0016 00022 (test.go:3)	SUBQ	$8,SP
	0x001a 00026 (test.go:3)	FUNCDATA	$0,gclocals·3280bececceccd33cb74587feedb1f9f+0(SB)
	0x001a 00026 (test.go:3)	FUNCDATA	$1,gclocals·3280bececceccd33cb74587feedb1f9f+0(SB)
	0x001a 00026 (test.go:4)	MOVQ	$1,(SP)
	0x0022 00034 (test.go:4)	PCDATA	$0,$0
	0x0022 00034 (test.go:4)	CALL	,runtime.printint(SB)
	0x0027 00039 (test.go:5)	ADDQ	$8,SP
	0x002b 00043 (test.go:5)	RET	,
```

나중에 올 블로그 포스트에서 이 코드에 대해 더 자세히 살펴보며 Go의 런타임이 어떻게 작동하는지를 이해하기 위한 시도들 할 것이다. 지금은 다음 한줄에 관심이 있다:

>```
0x0022 00034 (test.go:4)	CALL	,runtime.printint(SB)
```

이 명령은 함수 데이터내 (16진수로는) 0x0022의 오프셋 이나 (10진수로는) 00034 오프셋에 위치한다. 이 줄은 실제로 *runtime.printint* 함수를 호출하는 책임을 진다. 문제는 컴파일러가 컴파일이 진행되는 동안 *runtime.printint* 함수의 정확한 주소를 모른다는 것이다. 이 함수는 컴파일러가 전혀 모르는 다른 오브젝트 파일내에 위치한다. 그런 경우, 컴파일러는 재배치를 사용한다. 아래는 이 메서드 호출에 상응하는 정확한 재배치이다. (저자가 *goobj_explorer* 유틸리티의 첫번째 출력에서 복사해 왔다.):

>```
{
    Offset: 35,
    Size:   4,
    Sym:    goobj.SymID{Name:"runtime.printint", Version:0},
    Add:    0,
    Type:   3,
},
```

이 재배치는 링커에게 35 바이트의 오프셋에서 시작하면서, 4 바이트의 데이터를 *runtime.printint* 심볼의 시작점 주소로 교체할 필요가 있다고 말한다. 하지만 메인 함수 데이터로 부터 35 바이트의 오프셋는 실제로 이전에 본적이 있는 호출 명령(call instruction)의 인수이다. (이 (호출) 명령은 34 바이트의 오프셋에서 시작한다. 1 바이트는 호출 명령 코드이고 4 바이트는 이 명령의 주소를 가리킨다.)

# 링커는 어떻게 작동하는가

이제 위의 설명을 이해한다면, 링커가 어떻게 작동하는 지를 알아낼 수 있다. 다음의 개요는 매우 단순화 시킨 것이긴 하지만 주요한 아이디어를 반영한다:

 * 링커는 메인 패키지로 부터 참조된 모든 패키지의 심볼을 모아서 하나의 긴 바이트 배열(혹은 바이너리 이미지)에 실는다.
 * 각 심볼에 대해서는, 링커가 이러한 이미지내의 주소를 계산한다.
 * 그런다음, 모든 심볼에 대해 정의된 재배치를 적용한다. 링커가 그런 재배치에서 참조된 모든 다른 심볼들의 정확한 주소들들 알고 있기 때문에 매우 쉬운 일이다.
 * 링커는 (리눅스의) Executable and Linkable (ELF) 포맷이나 (윈도우의) Portable Executable (PE) 포맷에 필요한 모든 헤더를 준비한다. 그런 다음, 그 결과물로 링커는 실행파일을 발생시킨다.

# TLS 이해하기

조심성 있는 독자는 main 메서드에 대해 *goobj_explorer* 유틸리티 출력속에 이상한 재배치가 있음을 알아챌 것이다. 어떤 메서드 호출에도 상응하지 않고 심지어 빈 심볼을 가리키고 있다:

>```
{
    Offset: 5,
    Size:   4,
    Sym:    goobj.SymID{},
    Add:    0,
    Type:   9,
},
```

과연, 이 재배치가 하는 것이 무엇일까? 5 바이트의 오프셋을 가지고 있고 크기가 4 바이트임을 알 수 있다. 이 오프셋에는 다음 명령이 있다:

>```
0x0000 00000 (test.go:3)	MOVQ	(TLS),CX
```

0 오프셋에서 시작하고 9 바이트을 차지한다 (다음 명령이 9 바이트 오프셋애서 시작하는 걸로 알 수 있다). 추측컨대, 이 재배치는 낯선 *(TLS)* 구문을 어떤 주소로 교체한다. 그러면 TLS는 무엇이며, 무슨 주소를 사용하는가?

TLS는 쓰레드 지역 저장 공간(Thread Local Storage)의 축약형이다. 이 기술은 많은 프로그래밍 언어에 사용되었는데 상세한 내용은 [여기](https://en.wikipedia.org/wiki/Thread-local_storage)를 참조하라. 간단하게 설명하면, 다른 쓰레드에 의해 사용될 때, 다른 메모리 장소를 가리키는 변수의 사용을 가능하게 한다.

Go 언어에서 TLS는 *G 구조체* 를 가리키는 포인터를 저장하는 데 사용된다. *G 구조체* 는 특정한 Go 루틴 내부의 상세한 내용을 담고 있는데 나중에 올 블로그 포스트에서 더 자세히 다룰 것이다. 그러므로, 다른 Go 루틴들이 어떤 한 Go 루틴을 접근할 때, 이 Go 루틴 내부의 자세한 정보를 담고 있는 구조체를 가리키는 변수가 항상 존재한다는 얘기다. 이 변수의 위치는 링커에게 알려져 있어서 (우리가) 분석중인 명령안에서 이 변수가 CX 레지스터에 이동된다는 것을 알 수 있다. TLS는 아키텍쳐마다 다르게 구현될 수 있다. AMD64에서는, *FS* 레지스터를 통해 구현되어서, 이 명령은 *MOVQ FS, CX* 로 번역될 수 있다.

재배치에 대한 토론을 마감하기 위해, 모든 재배치 타입을 담고 있는 열거형 타입(enum) 을 소개하겠다:

>```
// Reloc.type
enum
{
	R_ADDR = 1,
	R_SIZE,
	R_CALL, // relocation for direct PC-relative call
	R_CALLARM, // relocation for ARM direct call
	R_CALLIND, // marker for indirect call (no actual relocating necessary)
	R_CONST,
	R_PCREL,
	R_TLS,
	R_TLS_LE, // TLS local exec offset from TLS segment register
	R_TLS_IE, // TLS initial exec offset from TLS base pointer
	R_GOTOFF,
	R_PLT0,
	R_PLT1,
	R_PLT2,
	R_USEFIELD,
};
```

이 enum에서 볼 수 있듯이, 재배치 타입 3는 R_CALL 이고 재배치 타입 9은 R_TLS이다. 이 enum 이름들은 방금 설명한 행동들을 완벽하게 설명한다.

# Go 오브젝트 파일에 대한 부연 설명

다음 포스트에서 오브젝트 파일에 대한 설명을 계속해 나가겠다. 또한 Go 런타임이 어떻게 작동하는 지를 이해하는데 필요한 정보들을 더 제공하겠다. 질문이 있다면 코멘트란에 부담없이 해 주길 바란다.

 * [Golang Internals, Part 3: The Linker, Object Files, and Relocations](http://blog.altoros.com/golang-internals-part-3-the-linker-and-object-files.html)
 * 저자: Siarhei Matsiukevich
 * 번역자: Jhonghee Park
