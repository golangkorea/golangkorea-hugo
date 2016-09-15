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

Do you know what exactly happens in the Go runtime, when you use a variable via interface reference? This is not a trivial question, because in Go a type that implements an interface does not contain any references to this interface whatsoever. Still, we can try answering it, using our knowledge of the Go compiler, which was discussed in the previous blog post.

So, let’s take a deep dive into the Go compiler: create a basic Go program and see the internal workings of the Go typecasting. Using it as an example, I’ll explain how a node tree is generated and utilized. So, you can further apply this knowledge to other Go compiler’s features.


# Before you start

To perform the experiment, we will need to work directly with the Go compiler (not the Go tool). You can access it by using the command:

>```
go tool 6g test.go
```

It will compile the *test.go* source file and create an object file. Here, *6g* is the name of the compiler on my machine that has an AMD64 architecture. Note that you should use different compilers for different architectures.

When we work directly with the compiler, we can use some handy command line arguments (more details [here](https://golang.org/cmd/gc/#hdr-Command_Line)). For the purposes of this experiment, we’ll need the -W flag that will print the layout of the node tree.


# Creating a simple Go program

First of all, we are going to create a sample Go program. My version is below:

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

Really simple, isn’t it? The only thing that might seem unnecessary is the 17th line, where we print the *i* variable. Nevertheless, without it, *i* will remain unused and the program will not be compiled. The next step is to compile our program using the -W switch:

>```
go tool 6g -W test.go
```

After doing this, you will see output that contains node trees for each method defined in the program. In our case, these are the main and *init* methods. The *init* method is here because it is implicitly defined for all programs, but we actually do not care about it right now.

For each method, the compiler prints two versions of the node tree. The first one is the original node tree that we get after parsing the source file. The second one is the version that we get after type checking and applying all the necessary modifications.


# Understanding the node tree of the main method

Let’s take a closer look at the original version of the node tree from the main method and try to understand what exactly is going on.

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
