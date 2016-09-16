+++
title = "Golang의 내부, 2부: Go 컴파일러 들여다 보기"
draft = true
date = "2016-09-15T05:53:48-04:00"

tags = ["Golang", "Internals", "Compiler", "Structure"]
categories = ["번역"]
series = ["Golang  Internals"]
authors = ["Jhonghee Park"]

toc = true
+++

독자는 인터페이스 레퍼런스를 통해 변수를 사용할 경우 Go 런타임내에서 어떤 일이 있는지 정확하게 알고 있는가? 이 질문에 쉽게 답할 수 없는 이유는 어떤 인터페이스를 구현하는 타입이 이 인터페이스를 지목하는 어떤 레퍼런스도 갖고 있지 않기 때문이다. 여전히 시도는 해 볼 수 있는데 [이전 블로그 포스트](/post/golang-internals/part1/)에서 논했던 Go 컴파일러의 지식을 이용하는 것이다.

그러면, Go 컴파일러속으로 잠수해 들어가자: 간단한 Go 프로그램을 제작하고 Go 타입캐스팅(typecasting)이 내부적으로 어떻게 동작하는 지 살펴보겠다. 이 것을 예로 들면서, 어떻게 노드 트리가 생성되고 사용되는지 설명하겠다. 그러므로 독자도 이 지식을 다른 Go 컴파일러 기능에 적용할 수 있다.

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

정말 간단하지 않은가? 불필요하게 생각되는 단 하나는 17째 줄인데, *i* 변수를 출력하는 부분이다. 그럼에도 불구하고 이 줄이 없다면, *i* 사용하지 않은 변수로 간주되어 프로그램은 컴파일 되지 않을 것이다. 다음 단계는 이 프로그램을 *-W* 를 사용해 컴파일 하는 것이다:

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

In the explanation below, I will use an abridged version, from which I removed all the unnecessary details.

The first node is rather simple:

>```
DCL l(15)
.   NAME-main.t l(15) PTR64-*main.T
```

The first node is a declaration node. *l(15)* tells us that this node is defined in line 15. The declaration node references the name node that represents the *main.t* variable. This variable is defined in the main package and is actually a 64-bit pointer to the *main.T* type. You can look at line 15 and easily understand what declaration is represented there.

The next one is a bit trickier.

>```
AS l(15)
.   NAME-main.t l(15) PTR64-*main.T
.   PTRLIT l(15) PTR64-*main.T
.   .   STRUCTLIT l(15) main.T
.   .   .   TYPE l(15) type=PTR64-*main.T PTR64-*main.T
```

The root node is the assignment node. Its first child is the name node that represents the main.t variable. The second child is a node that we assign to *main.t*—a pointer literal node (&). It has a child struct literal, which, in its turn, points to the type node that represents the actual type (*main.T*).

The next node is another declaration. This time, it is a declaration of the main.i variable that belongs to the main.I type.

>```
DCL l(16)
.   NAME-main.i l(16) main.I
```

Then, the compiler creates another variable, *autotmp_0000*, and assigns the *main.t* variable to it.

>```
AS l(16) tc(1)
.   NAME-main.autotmp_0000 l(16) PTR64-*main.T
.   NAME-main.t l(15) PTR64-*main.T
```

Finally, we came to the nodes that we are actually inetersted in.

>```
AS l(16)
.   NAME-main.i l(16)main.I
.   CONVIFACE l(16) main.I
.   .   NAME-main.autotmp_0000 PTR64-*main.T
```

Here, we can see that the compiler has assigned a special node called *CONVIFACE* to the main.i variable. But this does not give us much information about what’s happening under the hood. To find out what’s going on, we need to look into the node tree of the main method after all node tree modifications have been applied (you can find this information in the “after walk main” section of your output).


# How the compiler translates the assignment node

Below, you can see how the compiler translates our assignment node:

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

As you can see from the output, the compiler first adds an initialization node list (*AS-init*) to the assignment node. Inside the *AS-init* node, it creates a new variable, *main.autotmp_0003*, and assigns the value of the *go.itab.*””.T.””*.I variable to it. After that, it checks whether this variable is nil. If the variable is nil, the compiler calls the *runtime.typ2Itab* function and passes the following to it:

a pointer to the *main.T type* ,
a pointer to the *main.I* interface type,
and a pointer to the *go.itab.*””.T.””.I* variable.

From this code, it is quite evident that this variable is for caching the result of type conversion from *main.T* to *main.I*.


# Inside the *getitab* method

The next logical step is to find *runtime.typ2Itab*. Below is the listing of this function:

>```
func typ2Itab(t *_type, inter *interfacetype, cache **itab) *itab {
	tab := getitab(inter, t, false)
	atomicstorep(unsafe.Pointer(cache), unsafe.Pointer(tab))
	return tab
}
```

It is quite evident that the actual work is done inside the *getitab* method, because the second line simply stores the created tab variable in the cache. So, let’s look inside *getitab*. Since it is rather big, I only copied the most valuable part.

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

First, we allocate memory for the result:

>```
(*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*ptrSize, 0, &memstats.other_sys))
```

Why should we allocate memory in Go and why is this done in such a strange way? To answer this question, we need to look at the *itab* struct definition.

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

The last property, *fun*, is defined as an array of one element, but it is actually variable-sized. Later, we’ll see that this property contains an array of pointers to methods defined in a particular type. These methods correspond to the methods in the interface type. The authors of Go use dynamic memory allocation for this property (yes, such things are possible, when you use an unsafe package). The amount of memory to be allocated is calculated by adding the size of the struct itself to the number of methods in the interface multiplied by a pointer size.

>```
unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*ptrSize
```

Next, you can see two nested loops. First, we iterate through all interface methods. For each method in the interface, we try to find a corresponding method in a particular type (the methods are stored in the *mhdr* collection). The process of checking whether two methods are equal is quite self-explanatory.

>```
if t.mtyp == itype && t.name == iname && t.pkgpath == ipkgpath
```

If we find a match, we store a pointer to the method in the *fun* property of the result:

>```
*(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*ptrSize)) = t.ifn
```

A small note on performance: since methods are sorted alphabetically for interface and pre-set type definitions, this nested loop can repeat *O(n + m)* times instead of *O(n * m)* times, where n and m correspond to the number of methods.

Finally, do you remember the last part of the assignment?

>```
AS l(16)
.   NAME-main.i l(16) main.I
.   EFACE l(16) main.I
.   .   NAME-main.autotmp_0003 l(16) PTR64-*uint8
.   .   NAME-main.autotmp_0000 l(16) PTR64-*main.T
```

Here, we assign the *EFACE* node to the main.i variable. This node (*EFACE*) contains references to the *main.autotmp_0003* variable—a pointer to the itab struct that was returned by the *runtime.typ2Itab* method—and to the *autotmp_0000* variable that contains the same value as the *main.t* variable. This is all we need to call methods by interface references.

So, the *main.i* variable contains an instance of the *iface* struct defined in the runtime package:

>```
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
```

# What’s next?

I understand that I’ve only covered a very small part of the Go compiler and the Go runtime so far. There are still plenty of interesting things to talk about, such as object files, the linker, relocations, etc.—they will be overviewed in the upcoming blog posts.
