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

오브젝트 파일 안에 있는 정보가 모두 다 직접적으로 매핑되어 있는 것은 아니다. 몇몇 필드들은 링커에만 사용된다. 여전히 여기에서 가장 흥미로운 필드들은 *pcsp*, *pcfile*, 그리고 *pcln* 이다. 이 필드들은 [program counter](http://en.wikipedia.org/wiki/Program_counter)가 stack pointer, filename, 그리고 line으로 번역될 때 사용된다.


This is required, for example, when *panic* occurs. At that exact moment, the runtime only knows about the program counter of the current assembly instruction that has triggered the *panic*. So, the runtime uses that counter to obtain the current file, line number, and full stack trace. The file and line number are resolved directly, using the *pcfile* and *pcln* fields. The stack trace is resolved recursively, using *pcsp*.

Now that I have a program counter, the question is, how do I get a corresponding line number? To answer it, you need to look through assembly code and understand how line numbers are stored in the object file:

>```
1 0x001a 00026 (test.go:4)    MOVQ    $1,(SP)
2     0x0022 00034 (test.go:4)    PCDATA  $0,$0
3     0x0022 00034 (test.go:4)    CALL    ,runtime.printint(SB)
4     0x0027 00039 (test.go:5)    ADDQ    $8,SP
5     0x002b 00043 (test.go:5)    RET ,
```

We can see that program counters from 26 to 38 inclusive correspond to line number 4 and counters from 39 to *next_function_program_counter – 1* correspond to line number 5. For space efficiency, it is enough to store the following map:

>```
1 26 - 4
2 39 - 5
3 …
```

This is almost exactly what the compiler does. The *pcln* field points to a particular offset in a map that corresponds to the first program counter of the current function. Knowing this offset and also the offset of the first program counter of the next function, the runtime can use binary search to find the line number that corresponds to the given program counter.

In Go, this idea is generalized. Not only a line number or stack pointer can be mapped to a program counter, but also any integer value. This is done via the *PCDATA* instruction. Each time, the linker finds the following instruction:

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
