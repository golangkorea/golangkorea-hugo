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

Today, I will speak about the Go linker, Go object files, and relocations.

Why should we care about these things? Well, if you want to learn the internals of any large project, the first thing you need to do is split it into components or modules. Second, you need to understand what interface these modules provide to each other. In Go, these high-level modules are the compiler, linker, and runtime. The interface that the compiler provides and the linker consumes is an object file and that’s where we will start our investigation today.


# Generating a Go object file

Let’s do a practical experiment—write a super simple program, compile it, and see what object file will be produced. In my case, the program was as follows:

>```go
package main

func main() {
	print(1)
}
```

Really straightforward, isn’t it? Now we need to compile it:

>```
go tool 6g test.go
```

This command produces the test.6 object file. To investigate its internal structure, we are going to use the [goobj](https://github.com/golang/go/tree/master/src/cmd/internal/goobj) library. It is employed internally in Go source code, mainly for implementing a set of unit tests that verifies whether object files are generated correctly in different situations. For this blog post, I wrote a very simple program that prints the output generated from the *googj* library to the console. You can take a look at the sources of this program [here](https://github.com/s-matyukevich/goobj_explorer).

First of all, you need to download and install my program:

>```
go get github.com/s-matyukevich/goobj_explorer
```

Then execute the following command:

>```
goobj_explorer -o test.6
```

Now you should be able to see the *goob.Package* structure in your console.


# Investigating the object file

The most interesting part of our object file is the *Syms* array. This is actually a symbol table. Everything that you define in your program—functions, global variables, types, constants, etc.—is written to this table. Let’s look at the entry that corresponds to the *main* function. (Note that I have cut the Reloc and Func fields from the output for now. We will discuss them later.)

>```
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

The names of the fields in the *goobj.Sum* structure are pretty self-explanatory:

<style type="text/css"><!--
.myTable { background-color:white;border-collapse:collapse; } .myTable th { background-color:#E0E0E0;color:black; } .myTable td, .myTable th { padding:5px;border:1px solid #989898; }
--></style>

<table class="myTable">
<tbody>
<tr>
<th><center>Field</center></th>
<th style="width: 530px;" width="70%"><center>Description</center></th>
</tr>
<tr>
<td><strong>SumID</strong></td>
<td>The unique symbol ID that consists of the symbol’s name and version. Versions help to differentiate symbols with identical names.</td>
</tr>
<tr>
<td><strong>Kind</strong></td>
<td>Indicates to what kind the symbol belongs (more details later).</td>
</tr>
<tr>
<td><strong>DupOK</strong></td>
<td>This field indicates whether duplicates (symbols with the same name) are allowed.</td>
</tr>
<tr>
<td><strong>Size</strong></td>
<td>The size of symbol data.</td>
</tr>
<tr>
<td><strong>Type</strong></td>
<td>A reference to another symbol that represents a symbol type, if any.</td>
</tr>
<tr>
<td><strong>Data</strong></td>
<td>Contains binary data. This field has different meanings for symbols of different kinds, e.g., assembly code for functions, raw string content for string symbols, etc.</td>
</tr>
<tr>
<td><strong>Reloc</strong></td>
<td>The list of relocations (more details will be provided later)</td>
</tr>
<tr>
<td><strong>Func</strong></td>
<td>Contains special function metadata for function symbols (see more details below).</td>
</tr>
</tbody>
</table>

Now, let’s look at different kinds of symbols. All possible kinds of symbols are defined as constants in the *goobj* package (you can find them [here](https://github.com/golang/go/blob/master/src/cmd/internal/goobj/read.go#L30)). Below, I copied the first part of these constants:

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

As we can see, the main.main symbol belongs to kind 1 that corresponds to the *STEXT* constant. *STEXT* is a symbol that contains executable code. Now, let’s look at the *Reloc* array. It consists of the following structs:

>```go
type Reloc struct {
	Offset int
	Size   int
	Sym    SymID
	Add    int
	Type int
}
```

Each relocation implies that the bytes situated at the *[Offset, Offset+Size]* interval should be replaced with a specified address. This address is calculated by summing up the location of the *Sym* symbol with the *Add* number of bytes.


# Understanding relocations

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
