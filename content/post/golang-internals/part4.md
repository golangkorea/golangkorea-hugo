+++

title = "Golang의 내부, 4부: 오브젝트 파일, 그리고 함수 메타데이터"
draft = true
date = "2016-09-18T16:20:29-04:00"

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

이 명령은 실제로 어셈블러 명령을 생산하지 않는다. 대신, 이 명령의 두번째 인수를 맵안에서 현재의 프로그램 카운터를 사용해 저장하고, 첫번째 인수는 어떤 맵이 사용되는 지를 나타낸다. 이 첫번째 인수를 통해, 새로운 맵을 쉽게 첨가할 수 있는데, 컴파일러와 런타임에는 그 의미가 알려지지만 링커에게는 보이지 않는다.

# 가비지 컬렉터는 어떻게 함수 메타데이터를 사용하는가

마지막으로 아직 설명을 더 명확하게 해야할 함수 메타데이터의 내용은 FuncData 배열인데, 가비지 컬렉션에 필요한 정보를 담고 있다. Go 언어는 [mark-and-sweep](http://www.brpreiss.com/books/opus5/html/page424.html) 가비지 컬렉터 (GC)를 사용하는데 두 단계를 거쳐 동작한다. 첫번째 단계인 (mark)에서는, 모든 객체를 섭렵하면서 아직 사용중인 것들은 "reachable"로 표시한다. 표시되지 않은 모든 객체는 두번째 단계인 (sweep)에서 제거된다.

그래서, 가비지 컬렉터는 전역 변수, 프로세서 레지스터, 스택 프레임, 그리고 이미 위치가 알려진 객체내 포인터들과 같이 잘 알려진 위치들내 도달할 수 있는(reachable) 객체를 찾아보면서 시작한다. 하지만 곰곰히 생각해 보면, 스택 프레임안에서 포인터를 찾아내는 것이 그렇게 쉽지는 않는 일이다. 그렇다면 런타임이 가비지 컬렉션을 실행할때 스택안에서 어떻게 포인터와 포인터가 아닌 타입의 변수들을 알아 내는 것일까? 바로 여기에서 *FuncData* 가 등장하는 것이다.

각 함수마다 컴파일러는 두개의 변수를 만든다. 하나는 스택 프레임의 인수들을 위한 비트맵 벡터를 담고 있다. 다른 하나는 나머지 프레임을 위한 비트맵으로 함수에 정의된 포인터 타입의 모든 지역 변수들을 포함한다. 이 변수들은 각자 가비지 컬렉터에게 스택 프레임안에 포인터들이 어디에 위치하는지 정확하게 알려주고 이는 가비지 컬렉터가 일을 하는 데 충분한 정보이다.

*PCDATA* 와 같이 *FUNCDATA* 역시 의사-Go 어셈블리 명령(pseudo-Go assembly instruction)에 의해 발생된 것임을 언급할 가치가 있겠다:

>```
1 0x001a 00026 (test.go:3)    FUNCDATA    $0,gclocals·3280bececceccd33cb74587feedb1f9f+0(SB)
```

이 명령의 첫번째 인수는 이것이 인수들을 위한 함수 데이터인지 아니면 지역 변수 공간인지를 나타낸다. 두번째 인수는 실제로 GC 마스크를 담고 있는 숨겨진 변수에 대한 참조 값이다.

# Golang에 대해 더 알아보기

앞으로 나올 포스트에서는 Go 언어의 부트 스트랩 과정을 얘기하겠다. 런타임이 어떻게 작동하는지를 이해하는데 중요한 단서이다. 일주일 뒤에 보자.

* 원문: [Golang Internals, Part 4: Object Files and Function Metadata](http://blog.altoros.com/golang-part-4-object-files-and-function-metadata.html)
* 저자: Siarhei Matsiukevich
* 번역자: Jhonghee Park
