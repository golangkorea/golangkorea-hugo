+++

title = "Golang의 내부, 5부: 런타임 부트스트랩"
draft = true
date = "2016-09-19T16:20:29-04:00"

tags = ["Golang", "Internals", "runtime", "bootstrap"]
categories = ["번역", "핵킹"]
series = ["Golang  Internals"]
authors = ["Jhonghee Park"]

toc = true

+++

The bootstrapping process is the key to understanding how the Go runtime works. Learning it is essential, if you want to move forward with Go. So the fifth installment in our Golang Internals series is dedicated to the Go runtime and, specifically, the Go bootstrap process. This time you will learn about:

 * Go bootstrapping
 * resizable stacks implementation
 * internal TLS implementation

Note that this post contains a lot of assembler code and you will need at least some basic knowledge of it to proceed (here is a quick [guide to Go’s assembler](https://golang.org/doc/asm)). So let’s get going!



# Finding an entry point

First, we need to find what function is executed immediately after we start a Go program. To do this, we will write a simple Go app:

>```go
1 package main
2
3 func main() {
4     print(123)
5 }
```

Then we need to compile and link it:

>```
1 go tool 6g test.go
2 go tool 6l test.6
```

This will create an executable file called *6.out* in your current directory. The next step involves the [objdump](https://sourceware.org/binutils/docs/binutils/objdump.html) tool, which is specific to Linux. Windows and Mac users can find analogs or skip this step altogether. Now run the following command:

>```
1 objdump -f 6.out
```

You should get output that will contain the start address:

>```
1 6.out:     file format elf64-x86-64
2 architecture: i386:x86-64, flags 0x00000112:
3 EXEC_P, HAS_SYMS, D_PAGED
4 start address 0x000000000042f160
```

Next, we need to disassemble our executable and find what function is located at this address:

>```
1 objdump -d 6.out > disassemble.txt
```

Then we need to open the *disassemble.txt* file and search for “*42f160*.” Here is what I got:

>```
1 000000000042f160 <_rt0_amd64_linux>:
2   42f160:   48 8d 74 24 08              lea    0x8(%rsp),%rsi
3   42f165:   48 8b 3c 24                 mov    (%rsp),%rdi
4   42f169:   48 8d 05 10 00 00 00    lea    0x10(%rip),%rax        # 42f180 <main>
5   42f170:   ff e0                           jmpq   *%rax
```

Nice, we have found it! The entry point for my OS and architecture is a function called *_rt0_amd64_linux*.


# The starting sequence

Now we need to find this function in Go runtime sources. It is located in the [rt0_linux_amd64.s](https://github.com/golang/go/blob/master/src/runtime/rt0_linux_amd64.s) file. If you look inside the Go runtime package, you can find many filenames with postfixes related to OS and architecture names. When a runtime package is built, only the files that correspond to the current OS and architecture are selected. The rest are skipped. Let’s take a closer look at [rt0_linux_amd64.s](https://github.com/golang/go/blob/master/src/runtime/rt0_linux_amd64.s):

>```
1 TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
2     LEAQ    8(SP), SI // argv
3     MOVQ    0(SP), DI // argc
4     MOVQ    $main(SB), AX
5     JMP AX
6
7 TEXT main(SB),NOSPLIT,$-8
8     MOVQ    $runtime·rt0_go(SB), AX
9     JMP AX
```

The *_rt0_amd64_linux* function is very simple. It calls the main function and saves arguments (*argc* and *argv*) in registers (*DI* and *SI*). The arguments are located in the stack and can be accessed via the *SP* (stack pointer) register. The main function is also very simple. It calls *runtime.rt0_go*. The *runtime.rt0_go* function is longer and more complicated, so I will break it into small parts and describe each one separately.

The first section goes like this:

>```
1 MOVQ    DI, AX      // argc
2 MOVQ    SI, BX      // argv
3 SUBQ    $(4*8+7), SP        // 2args 2auto
4 ANDQ    $~15, SP
5 MOVQ    AX, 16(SP)
6 MOVQ    BX, 24(SP)
```

Here, we put some previously saved command line argument values inside the *AX* and *BX* decrease stack pointers. We also add space for two more four-byte variables and adjust it to be 16-bit aligned. Finally, we move the arguments back to the stack.

>```
1 // create istack out of the given (operating system) stack.
2 // _cgo_init may update stackguard.
3 MOVQ    $runtime·g0(SB), DI
4 LEAQ    (-64*1024+104)(SP), BX
5 MOVQ    BX, g_stackguard0(DI)
6 MOVQ    BX, g_stackguard1(DI)
7 MOVQ    BX, (g_stack+stack_lo)(DI)
8 MOVQ    SP, (g_stack+stack_hi)(DI)
```

The second part is a bit more tricky. First, we load the address of the global *runtime.g0* variable into the DI register. This variable is defined in the [proc1.go](https://github.com/golang/go/blob/master/src/runtime/proc1.go) file and belongs to the *runtime,g* type. Variables of this type are created for each goroutine in the system. As you can guess, *runtime.g0* describes a root goroutine. Then we initialize the fields that describe the stack of the root goroutine. The meaning of *stack.lo* and *stack.hi* should be clear. These are pointers to the beginning and the end of the stack for the current goroutine, but what are the *stackguard0* and *stackguard1* fields? To understand this, we need to set aside the investigation of the *runtime.rt0_go* function and take a closer look at stack growth in Go.


# Resizable stack implementation in Go

The Go language uses resizable stacks. Each goroutine starts with a small stack and its size changes each time a certain threshold is reached. Obviously, there is a way to check whether we have reached this threshold or not. In fact, the check is performed at the beginning of each function. To see how it works, let’s compile our sample program one more time with the *-S* flag (this will show the generated assembler code). The beginning of the main function looks like this:

>```
1 "".main t=1 size=48 value=0 args=0x0 locals=0x8
2     0x0000 00000 (test.go:3)    TEXT    "".main+0(SB),$8-0
3     0x0000 00000 (test.go:3)    MOVQ    (TLS),CX
4     0x0009 00009 (test.go:3)    CMPQ    SP,16(CX)
5     0x000d 00013 (test.go:3)    JHI ,22
6     0x000f 00015 (test.go:3)    CALL    ,runtime.morestack_noctxt(SB)
7     0x0014 00020 (test.go:3)    JMP ,0
8     0x0016 00022 (test.go:3)    SUBQ    $8,SP
```

First, we load a value from thread local storage (TLS) to the CX register (I have already explained what TLS is in one of my [previous posts](http://blog.altoros.com/golang-internals-part-3-the-linker-and-object-files.html)). This value always contains a pointer to the *runtime.g* structure that corresponds to the current goroutine. Then we compare the stack pointer to the value located at an offset of 16 bytes in the *runtime.g* structure. We can easily calculate that this corresponds to the *stackguard0* field.

So, this is how we check if we have reached the stack threshold. If we haven’t reached it yet, the check fails. In this case, we call the *runtime.morestack_noctxt* function repeatedly until enough memory has been allocated for the stack. The stackguard1 field works very similarly to *stackguard0*, but it is used inside the C stack growth prologue instead of Go. The inner workings of *runtime.morestack_noctxt* is also a very interesting topic, but we will discuss it later. For now, let’s return to the bootstrap process.


# Continuing the investigation of Go bootstrapping

We will proceed with the starting sequence by looking at the next portion of code inside the *runtime.rt0_go* function:

>```
01     // find out information about the processor we're on
02     MOVQ    $0, AX
03     CPUID
04     CMPQ    AX, $0
05     JE  nocpuinfo
06
07     // Figure out how to serialize RDTSC.
08     // On Intel processors LFENCE is enough. AMD requires MFENCE.
09     // Don't know about the rest, so let's do MFENCE.
10     CMPL    BX, $0x756E6547  // "Genu"
11     JNE notintel
12     CMPL    DX, $0x49656E69  // "ineI"
13     JNE notintel
14     CMPL    CX, $0x6C65746E  // "ntel"
15     JNE notintel
16     MOVB    $1, runtime·lfenceBeforeRdtsc(SB)
17 notintel:
18
19     MOVQ    $1, AX
20     CPUID
21     MOVL    CX, runtime·cpuid_ecx(SB)
22     MOVL    DX, runtime·cpuid_edx(SB)
23 nocpuinfo:
```

This part is not crucial for understanding major Go concepts, so we will look through it briefly. Here, we are trying to figure out what processor we are using. If it is Intel, we set the *runtime·lfenceBeforeRdtsc* variable. The *runtime·cputicks* method is the only place where this variable is used. This method utilizes a different assembler instruction to get cpu ticks depending on the value of *runtime·lfenceBeforeRdtsc*. Finally, we call the CPUID assembler instruction, execute it, and save the result in the *runtime·cpuid_ecx* and *runtime·cpuid_edx* variables. These are used in the [alg.go](https://github.com/golang/go/blob/master/src/runtime/alg.go) file to select a proper hashing algorithm that is natively supported by your computer’s architecture.

Ok, let’s move on and examine another portion of code:

>```
01 // if there is an _cgo_init, call it.
02 MOVQ    _cgo_init(SB), AX
03 TESTQ   AX, AX
04 JZ  needtls
05 // g0 already in DI
06 MOVQ    DI, CX  // Win64 uses CX for first parameter
07 MOVQ    $setg_gcc<>(SB), SI
08 CALL    AX
09
10 // update stackguard after _cgo_init
11 MOVQ    $runtime·g0(SB), CX
12 MOVQ    (g_stack+stack_lo)(CX), AX
13 ADDQ    $const__StackGuard, AX
14 MOVQ    AX, g_stackguard0(CX)
15 MOVQ    AX, g_stackguard1(CX)
16
17 CMPL    runtime·iswindows(SB), $0
18 JEQ ok
```

This fragment is only executed when *cgo* is enabled. cgo is a topic for a separate discussion and we might talk about it in one of the upcoming posts. At this point, we only want to understand the basic bootstrap workflow, so we will skip it.

The next code fragment is responsible for setting up TLS:

>```
01 needtls:
02     // skip TLS setup on Plan 9
03     CMPL    runtime·isplan9(SB), $1
04     JEQ ok
05     // skip TLS setup on Solaris
06     CMPL    runtime·issolaris(SB), $1
07     JEQ ok
08
09     LEAQ    runtime·tls0(SB), DI
10     CALL    runtime·settls(SB)
11
12     // store through it, to make sure it works
13     get_tls(BX)
14     MOVQ    $0x123, g(BX)
15     MOVQ    runtime·tls0(SB), AX
16     CMPQ    AX, $0x123
17     JEQ 2(PC)
18     MOVL    AX, 0   // abort
```

I have already mentioned TLS before. Now it is time to understand how it is implemented.


# Internal TLS implementation

If you look at the previous code fragment carefully, you can easily understand that the only lines that do actual work are:

>```
1 LEAQ    runtime·tls0(SB), DI
2     CALL    runtime·settls(SB)
```

All the other stuff is used to skip TLS setup when it is not supported on your OS and check that TLS works correctly. The two lines above store the address of the *runtime·tls0* variable in the DI register and call the *runtime·settls* function. The code of this function is shown below:

>```
01 // set tls base to DI
02 TEXT runtime·settls(SB),NOSPLIT,$32
03     ADDQ    $8, DI  // ELF wants to use -8(FS)
04
05     MOVQ    DI, SI
06     MOVQ    $0x1002, DI // ARCH_SET_FS
07     MOVQ    $158, AX    // arch_prctl
08     SYSCALL
09     CMPQ    AX, $0xfffffffffffff001
10     JLS 2(PC)
11     MOVL    $0xf1, 0xf1  // crash
12     RET
```

From the comments, we can understand that this function makes an *arch_prctl* system call and passes *ARCH_SET_FS* as an argument. We can also see that this system call sets a base for the *FS* segment register. In our case, we set TLS to point to the *runtime·tls0* variable.

Do you remember the instruction that we saw at the beginning of the assembler code for the main function?

>```
1 0x0000 00000 (test.go:3)    MOVQ    (TLS),CX
```

I have previously explained that it loads the address of the *runtime.g* structure instance into the CX register. This structure describes the current goroutine and is stored in thread local storage. Now we can find out and understand how this instruction is translated into machine assembler. If you open the previously created *disassembly.txt* file and look for the *main.main* function, the first instruction inside it should look like this:

>```
1 400c00:       64 48 8b 0c 25 f0 ff    mov    %fs:0xfffffffffffffff0,%rcx
```

The colon in this instruction (*%fs:0xfffffffffffffff0*) stands for segmentation addressing (you can read more on it [here](http://thestarman.pcministry.com/asm/debug/Segments.html)).


# Returning to the starting sequence

Finally, let’s look at the last two parts of the *runtime.rt0_go* function:

>```
01 ok:
02     // set the per-goroutine and per-mach "registers"
03     get_tls(BX)
04     LEAQ    runtime·g0(SB), CX
05     MOVQ    CX, g(BX)
06     LEAQ    runtime·m0(SB), AX
07
08     // save m->g0 = g0
09     MOVQ    CX, m_g0(AX)
10     // save m0 to g0->m
11     MOVQ    AX, g_m(CX)
```

Here, we load the TLS address into the BX register and save the address of the *runtime·g0* variable in TLS. We also initialize the *runtime.m0* variable. If *runtime.g0* stands for root goroutine, then *runtime.m0* corresponds to the root operating system thread used to run this goroutine. We may take a closer look at *runtime.g0* and *runtime.m0* structures in upcoming blog posts.

The final part of the starting sequence initializes arguments and calls different functions, but this is a topic for a separate discussion.


# More on Golang

So, we have learned the inner mechanisms of the bootstrap process and found out how stacks are implemented. To move forward, we need to analyze the last part of the starting sequence. That will be the subject of my next post. If you want to get notified as soon as it comes out, hit the subscribe button below or follow [@altoros](http://www.twitter.com/altoros).


* 원문: [Golang Internals, Part 5: the Runtime Bootstrap Process](http://blog.altoros.com/golang-internals-part-5-runtime-bootstrap-process.html)
* 저자: Siarhei Matsiukevich
* 번역자: Jhonghee Park
