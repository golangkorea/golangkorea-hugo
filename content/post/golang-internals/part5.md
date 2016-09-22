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

부트스트래핑 과정은 Go의 런타임이 어떻게 작동하는지를 이해하는데 열쇠와 같은 구실을 한다. Go와 함께 앞으로 나아가고자 한다면 반드시 배워야한다. 그래서 Golang의 내부 시리즈의 다섯번째는 Go의 런타임, 특히 Go의 부트스트래핑 과정에 바치겠다. 이번에 독자가 배울 항목들은:

 * Go 부트스트래핑
 * 가변 스택 구현
 * TLS 내부 구현

이 포스트에 어셈블러 코드가 많이 포함되어 있는 점을 주목하라. 진행하기 위해 적어도 어셈블러의 기본 지식은 필요할 것이다. (속성 [Go 어셈블러 가이드](https://golang.org/doc/asm)가 여기 있다.) 이제 시작해 보자!


# 프로그램 시작점 찾기

우선, Go 프로그램이 시작된 후 즉시 실행되는 함수가 무엇인지 찾아보자. 그러기 위해, 간단한 Go 앱을 제작할 것이다:

>```go
1 package main
2
3 func main() {
4     print(123)
5 }
```

그런 다음 컴파일하고 링크 할 필요가 있다:

>```
1 go tool 6g test.go
2 go tool 6l test.6
```

이 과정을 통해 *6.out* 이라고 불리는 실행 파일이 현재 디렉토리에 만들어 진다. 다음 단계는 [objdump](https://sourceware.org/binutils/docs/binutils/objdump.html) 툴을 사용한다. 이 툴은 리눅스에만 해당되는 툴이어서 윈도우나 맥 사용자들은 유사한 툴을 찾던지 이 단계를 그냥 건너 뛰어야 한다. 이제 다음 명령을 실행하라:

>```
1 objdump -f 6.out
```

이것을 통해 시작 주소를 담고 있는 출력을 얻을 것이다:

>```
1 6.out:     file format elf64-x86-64
2 architecture: i386:x86-64, flags 0x00000112:
3 EXEC_P, HAS_SYMS, D_PAGED
4 start address 0x000000000042f160
```

다음은, 실행파일을 역어셈블하고 이 주소에 위치한 함수가 무엇인지 알아 낸다:

>```
1 objdump -d 6.out > disassemble.txt
```

그런 다음 *disassemble.txt* 파일을 열어서 “*42f160*.”를 검색하여 다음과 같은 결과를 얻는다:

>```
1 000000000042f160 <_rt0_amd64_linux>:
2   42f160:   48 8d 74 24 08              lea    0x8(%rsp),%rsi
3   42f165:   48 8b 3c 24                 mov    (%rsp),%rdi
4   42f169:   48 8d 05 10 00 00 00    lea    0x10(%rip),%rax        # 42f180 <main>
5   42f170:   ff e0                           jmpq   *%rax
```

좋아! 찾았다! 저자의 OS와 아키텍쳐에 해당하는 시작점은 *_rt0_amd64_linux* 라는 함수이다.

# 시작하는 순서

이제 이 함수를 Go 런타임 소스코드에서 찾을 필요가 있다. 위치한 곳은 [rt0_linux_amd64.s](https://github.com/golang/go/blob/master/src/runtime/rt0_linux_amd64.s) 파일이다. Go runtime 패키지속을 들여다 보면, 많은 파일의 이름들이 OS와 아키텍쳐 이름에 연관된 어미들(postfixes)로 되어 있음을 발견할 수 있다. runtime 패키지가 빌드될 때, 현재 OS와 아키텍쳐에 상응하는 파일들만 선택되고 나머지는 건너뛴다. [rt0_linux_amd64.s](https://github.com/golang/go/blob/master/src/runtime/rt0_linux_amd64.s)를 더 자세히 들여다 보자:

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

*_rt0_amd64_linux* 함수는 매우 단순하다. main 함수를 부르고 인수값 (*argc* and *argv*) 을 레지스터 (*DI* and *SI*)에 저장한다. 인수들은 스택에 위치하고 *SP* (스택 포인터) 레지스터를 통해 접근할 수 있다. main 함수 역시 매우 간단하다. *runtime.rt0_go* 를 호출한다. *runtime.rt0_go* 함수는 좀 길고 더 복잡하다. 그래서 작은 부분들로 분해한 다음 하나씩 따로 설명할 것이다.

첫번째 섹션은 이러하다:

>```
1 MOVQ    DI, AX      // argc
2 MOVQ    SI, BX      // argv
3 SUBQ    $(4*8+7), SP        // 2args 2auto
4 ANDQ    $~15, SP
5 MOVQ    AX, 16(SP)
6 MOVQ    BX, 24(SP)
```

이전에 저장해 두었던 코맨드라인 인수값을 *AX* 와 *BX* 에 두고 스택 포인터를 감소시킨다. 두개의 4 바이트 변수를 위한 공간을 추가하고 16 비트로 정렬되게 조정한다. 마지막으로 인수값은 다시 스택에 이동시킨다.

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

두번째 부분은 좀 더 까다롭다. 우선, 전역 변수 *runtime.g0* 의 주소를 DI 레지스터에 올린다. 이 변수는 [proc1.go](https://github.com/golang/go/blob/master/src/runtime/proc1.go) 파일에 정의되어 있고 *runtime,g* 타입에 속한다. 이 타입의 변수들은 시스템내 각 고루틴(goroutine)마다 만들어 진다. 독자가 추측할 수도 있듯이, *runtime.g0* 는 루트 고루틴(root goroutine)을 나타낸다. 그런 다음 이 루트 고루틴의 스택을 묘사하는 필드들을 초기화한다. *stack.lo* 와 *stack.hi* 가 뜻하는 바는 분명하다. 이것들은 현재 고루틴의 시작과 끝을 가리키는 포인터 들이다. 그런데 *stackguard0* 와 *stackguard1* 필드는 무엇일까? 이 것들을 이해하기 위해서는 *runtime.rt0_go* 함수를 분석하는 일을 잠시 접어 두고 Go 언어에서 스택 크기 변화에 대해 좀 더 자세히 알아 보아야 한다.

# Go 언어에서 크기를 조정할 수 있는 스택의 구현

Go 언어는 크기를 조정할 수 있는 스택을 사용한다. 각 고루틴은 작은 스택으로 시작해서 한계치에 도달하면 크기를 바꾼다. 물론 이 한계치에 도달했는지를 알아보는 방법이 있다. 사실 각 함수는 시작할 때 스택이 한계에 도달했는지를 확인한다. 이것이 어떻게 작동하는지 알아보기 위해 샘플 프로그램을 *-S* 플래그를 이용해 다시 한번 컴파일 하자. 어셈블리 코드을 보게 될 것 이다. main 함수의 시작부분은 다음과 같다:

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

우선 쓰레드 로컬 스토리지 (TLS)에서 한 값을 CX 레지스터에 올린다(TLS가 무엇인지는 [이전 포스트](/post/golang-internals/part3/)에서 이미 설명한 바 있다). 이 값은 항상 현재 고루틴에 상응하는  *runtime.g* 구조에 대한 포인터를 담고 있다. 그런 다음 스택포인터를 *runtime.g* 구조내 16 바이트의 오프셋에 위치한 값과 비교한다. 계산해 보면 이 값이 *stackguard0* 필드에 상응한다는 것을 쉽게 알 수 있다.

바로 이것이 스택 한계치에 도달했는지를 확인하는 방식이다. 아직 도달하지 않았다면, 확인은 실패로 간주되어서 스택에 충분한 메모리가 할당될 때 까지 *runtime.morestack_noctxt* 함수를 반복적으로 호출한다. *stackguard1* 필드는 *stackguard0* 와 매우 유사하게 작동한다. 하지만 Go 대신 C 스택 성장 프롤로그 (C stack growth prologue)내에서 사용된다. *runtime.morestack_noctxt* 의 내부 작동 원리 또한 매우 흥미로운 주제이긴 하지만 나중에 논하기로 하겠다. 지금은 부트스트랩 과정으로 다시 돌아가기로 하자.


# 계속되는 Go 부트스트래핑에 대한 조사

시작하는 순서에 대해 더 나아가기 위해서 *runtime.rt0_go* 함수내 다음 부분에 있는 코드를 살펴보기로 하자:

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
