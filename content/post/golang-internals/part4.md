+++

title = "Golang의 내부, 4부: 오브젝트 파일, 그리고 함수 메타데이터"
draft = true
date = "2016-09-20T16:20:29-04:00"

tags = ["Golang", "Internals", "linker", "object file", "relocations", "metadata"]
categories = ["번역", "핵킹"]
series = ["Golang  Internals"]
authors = ["Jhonghee Park"]

toc = true

+++

오늘은, Func 구조에 대해 좀 더 자세히 들여다 보고 어떻게 가비지 컬렉션이 작동하는지 몇가지 자세한 내용을 논하겠다.

이 포스트는 [Golang의 내부, 3부: 링커, 오브젝트 파일, 그리고 재배치](/post/golang-internals/part3/)의 연속이어서, 독자가 아직 읽지 않았다면 이 포스트를 소화하기 전에 읽기를 적극 권장한다.

# 함수 메타데이터의 구조

재배치에 대한 주요한 아이디어는 3부를 통해 분명해 졌을 것이다. 이제 main 메서드의 Func 구조를 살펴보자:

>```
01 Func: &goobj.Func{
02     Args:    0,
03     Frame:   8,
04     Leaf:    false,
05     NoSplit: false,
06     Var:     {
07     },
08     PCSP:   goobj.Data{Offset:255, Size:7},
09     PCFile: goobj.Data{Offset:263, Size:3},
10     PCLine: goobj.Data{Offset:267, Size:7},
11     PCData: {
12         {Offset:276, Size:5},
13     },
14     FuncData: {
15         {
16             Sym:    goobj.SymID{Name:"gclocals·3280bececceccd33cb74587feedb1f9f", Version:0},
17          Offset: 0,
18     },
19     {
20          Sym:    goobj.SymID{Name:"gclocals·3280bececceccd33cb74587feedb1f9f", Version:0},
21                Offset: 0,
22            },
23        },
24        File: {"/home/adminone/temp/test.go"},
25    },
```

이 구조체는 Go언어의 런타임이 사용하는, 컴파일러가 오브젝트 파일에 방출한 함수 메타데이터로 생각해 볼 수 있다. [이 문서](https://docs.google.com/document/d/1lyPIbmsYbXnpNj57a261hgOYVpNRcgydurVQIyZOz_o/pub)를 통해 Func내 정확한 포맷과 여러 필드들의 의미에 대한 설명을 접할 수 있다. 이제, 런타임에서 이 메타데이터가 어떻게 사용되는지 보겠다.

runtime 패키지 안에서 이 메타데이터는 다음 구조체에 매핑되어 있다.

>```
01 type _func struct {
02     entry   uintptr // start pc
03     nameoff int32   // function name
04
05     args  int32 // in/out args size
06     frame int32 // legacy frame size; use pcsp if possible
07
08     pcsp      int32
09     pcfile    int32
10     pcln      int32
11     npcdata   int32
12     nfuncdata int32
13 }
```

오브젝트 파일 안에 있는 정보가 모두 다 직접적으로 매핑되어 있는 것은 아니다. 몇몇 필드들은 링커에만 사용된다. 그렇다 해도 여기에서 가장 흥미로운 필드들은 *pcsp*, *pcfile*, 그리고 *pcln* 이다. 이 필드들은 [프로그램 카운터](http://en.wikipedia.org/wiki/Program_counter)가 stack pointer, filename, 그리고 line으로 번역될 때 사용된다.

예를 들면, *panic* 이 발생할 경우 이 메타데이터 필드들이 필요할 것이다. 그 순간에 런타임이 알고 있는 바는 오직 *panic* 을 야기한 현재의 어셈블리 명령의 프로그램 카운터이다. 그래서 런타임은 그 카운터를 이용해 현재 파일과 라인 번호, 그리고 stack trace 전부를 얻는 것이다. 파일과 라인 번호는 *pcfile* 과 *pcln* 필드를 이용하면 바로 해결된다. stack trace는 *pcsp* 를 이용하여 재귀적으로 해결한다.

이제 프로그램 카운터를 가지고 어떻게 상응하는 라인 번호를 얻을 수 있는지에 대한 질문에 답을 하자. 답을 얻기 위해서는 어셈블리 코드를 들여다 보고 오브젝트 파일에 라인 번호가 어떻게 저장되어 있는 지를 이해해야 한다:

>```
1 0x001a 00026 (test.go:4)    MOVQ    $1,(SP)
2     0x0022 00034 (test.go:4)    PCDATA  $0,$0
3     0x0022 00034 (test.go:4)    CALL    ,runtime.printint(SB)
4     0x0027 00039 (test.go:5)    ADDQ    $8,SP
5     0x002b 00043 (test.go:5)    RET ,
```

프로그램 카운터 26에서 38까지는 라인 번호 4에 상응하고 카운터 39에서 *next_function_program_counter - 1* 까지는 라인 번호 5에 해당된다. 효율적 공간사용을 생각하면 다음과 같은 맵을 저장하는 것으로 충분하다:

>```
1 26 - 4
2 39 - 5
3 …
```

이것은 컴파일러가 하는 일과 거의 일치한다. *pcln* 필드는 현재 실행중인 함수의 첫번째 프로그램 카운터에 상응하는 특정한 오프셋을 맵안에서 가리키고 있다. 이 오프셋과 또 다음 함수의 첫번째 프로그램 카운터의 오프셋을 알고, 런타임은 바이너리 검색을 이용해 주어진 프로그램 카운터에 상응하는 라인 번호를 찾는다.

Go 언어에서 이런 아이디어는 일반화 되어 있다. 라인 번호와 스택 포인터만 프로그램 카운터에 맵팅되어 있는게 아니라 어떤 정수 값도 매핑될 수 있다. *PCDATA* 명령을 통해서 가능한 것이다. 매번 링커는 다음 명령을 찾는다:

>```
1 0x0022 00034 (test.go:4)    PCDATA  $0,$0
```

It doesn’t generate any actual assembler instructions. Instead, it stores the second argument of this instruction in a map with the current program counter, while the first argument indicates what map is used. With this first argument, we can easily add new maps, which meaning is known to the compiler and runtime but is opaque to the linker.


# How the garbage collector uses function metadata

The last thing that still needs to be clarified in function metadata is the FuncData array. It contains information necessary for garbage collection. Go uses a [mark-and-sweep](http://www.brpreiss.com/books/opus5/html/page424.html) garbage collector (GC) that operates in two stages. During the first stage (mark), it traverses through all objects that are still in use and marks them as reachable. All the unmarked objects are removed during the second (sweep) stage.

So, the garbage collector starts by looking for a reachable object in several known locations, such as global variables, processor registers, stack frames, and pointers in objects that have already been reached. However, if you think about it carefully, looking for pointers in stack frames is far from a trivial task. So, when the runtime is performing garbage collection, how does it distinguish whether a variable in the stack is a pointer or belongs to a non-pointer type? This is where *FuncData* comes into play.

For each function, the compiler creates two variables. One contains a bitmap vector for the arguments area of the stack frame. The other one contains a bitmap for the rest of the frame that includes all the local variables of pointer types defined in the function. Each of these variables tells the garbage collector, where exactly in the stack frame the pointers are located, and that information is enough for it to do its job.

It is also worth mentioning that, like *PCDATA*, *FUNCDATA* is also generated by a pseudo-Go assembly instruction:

>```
1 0x001a 00026 (test.go:3)    FUNCDATA    $0,gclocals·3280bececceccd33cb74587feedb1f9f+0(SB)
```

The first argument of this instruction indicates, whether this is function data for arguments or a local variables area. The second one is actually a reference to a hidden variable that contains a GC mask.


# More on Golang

In the upcoming posts, I will tell you about the Go bootstrap process, which is the key to understanding how the Go runtime works. See you in a week.

* [Golang Internals, Part 4: Object Files and Function Metadata](http://blog.altoros.com/golang-part-4-object-files-and-function-metadata.html)
* 저자: Siarhei Matsiukevich
* 번역자: Jhonghee Park
