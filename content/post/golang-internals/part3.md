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
package main

func main() {
	print(1)
}
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


Now let’s use an example and see how relocations work. To do this, we need to compile our program using the *-S* switch that will print the generated assembly code:

>```
go tool 6g -S test.go
```

Let’s look through the assembler and try to find the main function.

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

In later blog posts, we’ll have a closer look at this code and try to understand how the Go runtime works. For now, we are interested in the following line:

>```
0x0022 00034 (test.go:4)	CALL	,runtime.printint(SB)
```

This command is located at an offset of 0x0022 (in hex) or 00034 (decimal) within the function data. This line is actually responsible for calling the *runtime.printint* function. The issue is that the compiler does not know the exact address of the *runtime.printint* function during compilation. This function is located in a different object file the compiler knows nothing about. In such cases, it uses relocations. Below is the exact relocation that corresponds to this method call (I copied it from the first output of the *goobj_explorer* utility):

>```
{
                    Offset: 35,
                    Size:   4,
                    Sym:    goobj.SymID{Name:"runtime.printint", Version:0},
                    Add:    0,
                    Type:   3,
                },
```

This relocation tells the linker that, starting from an offset of 35 bytes, it needs to replace 4 bytes of data with the address of the starting point of the *runtime.printint* symbol. But an offset of 35 bytes from the main function data is actually an argument of the call instruction that we have previously seen. (The instruction starts from an offset of 34 bytes. One byte corresponds to call instruction code and four bytes—to the address of this instruction.)


# How the linker operates

Now that we understand this, we can figure out how the linker works. The following schema is very simplified, but it reflects the main idea:

 * The linker gathers all the symbols from all the packages that are referenced from the main package and loads them into one big byte array (or a binary image).
 * For each symbol, the linker calculates an address in this image.
 * Then it applies the relocations defined for every symbol. It is easy now, since the linker knows the exact addresses of all other symbols referenced from those relocations.
 * The linker prepares all the headers necessary for the Executable and Linkable (ELF) format (on Linux) or the Portable Executable (PE) format (on Windows). Then, it generates an executable file with the results.


# Understanding TLS

A careful reader will notice a strange relocation in the output of the goobj_explorer utility for the main method. It doesn’t correspond to any method call and even points to an empty symbol:

>```
{
                    Offset: 5,
                    Size:   4,
                    Sym:    goobj.SymID{},
                    Add:    0,
                    Type:   9,
                },
```

So, what does this relocation do? We can see that it has an offset of 5 bytes and its size is 4 bytes. At this offset, there is a command:

>```
0x0000 00000 (test.go:3)	MOVQ	(TLS),CX
```

It starts at an offset of 0 and occupies 9 bytes (since the next command starts at an offset of 9 bytes). We can guess that this relocation replaces the strange *(TLS)* statement with some address, but what is TLS and what address does it use?

TLS is an abbreviation for Thread Local Storage. This technology is used in many programming languages (more details [here](https://en.wikipedia.org/wiki/Thread-local_storage)). In short, it enables us to have a variable that points to different memory locations when used by different threads.

In Go, TLS is used to store a pointer to the G structure that contains internal details of a particular Go routine (more details on this in later blog posts). So, there is a variable that—when accessed from different Go routines—always points to a structure with internal details of this Go routine. The location of this variable is known to the linker and this variable is exactly what was moved to the CX register in the previous command. TLS can be implemented differently for different architectures. For AMD64, TLS is implemented via the *FS* register, so our previous command is translated into *MOVQ FS, CX*.

To end our discussion on relocations, I am going to show you the enumerated type (enum) that contains all the different types of relocations:

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

As you can see from this enum, relocation type 3 is R_CALL and relocation type 9 is R_TLS. These enum names perfectly explain the behaviour that we discussed previously.


# More on Go object files

In the next post, we’ll continue our discussion on object files. I will also provide more information necessary for you to move forward and understand how the Go runtime works. If you have any questions, feel free to ask them in the comments.


 * [Golang Internals, Part 3: The Linker, Object Files, and Relocations](http://blog.altoros.com/golang-internals-part-3-the-linker-and-object-files.html)
 * 저자: Siarhei Matsiukevich
 * 번역자: Jhonghee Park
