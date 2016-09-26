+++

title = "Golang의 내부, 6부: 부트스트래핑과 메모리 할당자"
draft = false
date = "2016-09-20T16:20:29-04:00"

tags = ["Golang", "Internals", "runtime", "bootstrap", "memory", "allocator"]
categories = ["번역", "핵킹"]
series = ["Golang  Internals"]
authors = ["Jhonghee Park"]

toc = true

+++

이 포스트는 Golang 내부 시리즈의 연속이다. Go 런타임을 자세히 이해하는데 열쇠와 같은 부트스트랩 과정을 살펴볼 것이다. 이번에는 시작하는 순서의 두번째 부분을 섭렵해서 어떻게 인수들이 초기화되고, 어떤 함수들이 호출되는지 등을 배우겠다.

# 시작하는 순서

지난 번에 얘기하다가 만 *runtime.rt0_go* 함수를 다시 다루어야 겠다. 아직 이 함수에서 살펴보지 않은 부분이 여전히 있다.

>```
01 CLD                         // convention is D is always left cleared
02 CALL    runtime·check(SB)
03
04 MOVL    16(SP), AX          // copy argc
05 MOVL    AX, 0(SP)
06 MOVQ    24(SP), AX          // copy argv
07 MOVQ    AX, 8(SP)
08 CALL    runtime·args(SB)
09 CALL    runtime·osinit(SB)
10 CALL    runtime·schedinit(SB)
```

첫번째 명령 (CLD)는 *FLAGS* 레지스터의 [direction](https://en.wikipedia.org/wiki/Direction_flag) 프래그를 지운다. 이 플래그는 문자열 처리 방향에 영향을 준다.

다음 함수는 *runtime.check* 함수를 호출하는데, 그 또한 이 문서의 목적에 비추어 그리 중요하지는 않다. 런타임은 모든 내장 타입의 인스턴스들을 만들고, 타입의 크기와 파라미터들을 확인하는 정도의 일을 한다. 그리고 만약 작업중 문제가 생기면 *panic* 한다. [function](https://github.com/golang/go/blob/go1.5.1/src/runtime/runtime1.go#L136)를 통해 쉽게 알아볼 수 있다.

# 인수 분석하기

그 다음 함수 [runtime.Args](https://github.com/golang/go/blob/go1.5.1/src/runtime/runtime1.go#L48) 는 좀 더 흥미롭다. The next function, 리눅스 시스템에서 (*argc* 와 *argv*) 인수를 정적 변수속에 저장하는 것말고도 이 함수는 ELF 보조 벡터를 분석하며 *syscall* 주소를 초기화한다.

설명이 좀 더 필요하겠다. 운영체계가 프로그램을 메모리에 올릴때, 그 프로그램의 초기 스택을 미리 정해진 포맷의 어떤 데이타로 초기화한다. 스택의 꼭대기에는 환경 변수들의 포인터인 인수들이 깔린다. 스택의 바닥에는 "ELF 보조 벡터"를 볼 수 인데, 실제로 이 것은 어떤 유용한 정보를 담고 있는 기록의 배열들이다. 예를 들면, 프로그램 헤더의 수와 크기들이다. 여기 이 [문서](http://articles.manugarg.com/aboutelfauxiliaryvectors)를 통해 ELF 보조 벡터 포맷에 대해 좀 자세히 알아 보라.

*runtime.Args* 함수는 벡터를 파싱하는 책임이 있다. 런타임은 벡터에 담고 있는 모든 정보들 중에 단지 *startupRandomData* 만을 사용하는데, 주로 헤싱 함수(hashing functions)들과 어떤 syscall들의 위치를 가리키는 포인터들을 초기화하는데 주로 사용된다. 다음에 나오는 변수들을 초기화한다:

>```
1 __vdso_time_sym
2 __vdso_gettimeofday_sym
3 __vdso_clock_gettime_sym
```

이 것들은 여러 함수들내 현재 시간을 획득하는데 사용된다. 이 모든 변수들은 기본값을 가진다. 이렇게 함으로써 Golang에서 상응하는 함수들을 호출하기 위해 *[vsyscall](http://www.ukuug.org/events/linux2001/papers/html/AArcangeli-vsyscalls.html)* 메카니즘의 사용이 허용된다.

# *runtime.osinit* 함수의 내부

시작 순서에서 그 다음으로 호출되는 함수는 *[runtime.osinit](https://github.com/golang/go/blob/go1.5.1/src/runtime/os1_linux.go#L172)* 이다. 리눅스 시스템에서 단 한가지 하는 일이 있는데 그것은 시스템내 CPU 숫자를 가지고 있는 ncpu 변수를 한 syscall을 통해 초기화하는 것이다.

# *runtime.schedinit* 함수의 내부

시작 순서에서 다음 함수인 *[runtime.schedinit](https://github.com/golang/go/blob/go1.5.1/src/runtime/proc1.go#L40)* 는 더 흥미롭다. 현재의 고루틴을 얻는 것으로 시작하는데, 사실 이 것은 *[g](https://github.com/golang/go/blob/go1.5.1/src/runtime/runtime2.go#L211)* 구조의 포인터이다. 이미 이 포인터가 어떻게 저장되는 지는 TLS 구현을 논할 때 얘기한 바있다. 다음은 *[runtime.raceinit](https://github.com/golang/go/blob/go1.5.1/src/runtime/race1.go#L110)* 를 호출한다. runtime.raceinit에 대한 토론은 넘어가겠다. 왜냐하면 이 함수는 race 조건이 활성화되지 않은 경우는 보통 호출되지 않기 때문이다. 그 다음에는 몇몇 다른 초기화 함수들이 호출된다.

하나씩 살펴보자.

# traceback 초기화

*[runtime.tracebackinit](https://github.com/golang/go/blob/go1.5.1/src/runtime/traceback.go#L58)* 함수는 traceback을 초기화하는 일을 한다. Traceback이라는 것은 현재의 실행지점에 오기까지 호출된 함수들의 스택을 말한다. 예를 들어, panic이 일어날 때마다 볼 수 있다. Traceback은 주어진 프로그램 카운터로 *[runtime.gentraceback](https://github.com/golang/go/blob/go1.5.1/src/runtime/traceback.go#L120)* 함수를 호출함으로써 얻어진다. 이 함수가 동작하려면, 몇몇 내장함수들의 주소를 알 필요가 있는데, 그 이유는 traceback에 이 함수들이 나타나는 것을 원하지 않기 때문이다. 이 주소들은 *runtime.tracebackinit* 을 통해 초기화된다.

# 링커 심볼들 검사하기

링커 심볼들은 링커가 실행파일과 오브젝트 파일에 뿜어내는 데이터를 말한다. 대부분의 내용은 [Golang의 내부, 3부: 링커, 오브젝트 파일, 그리고 재배치](/post/golang-internals/part3/)에서 논했다. runtime 패키지에서 링커 심볼들은 *[moduledata](https://github.com/golang/go/blob/go1.5.1/src/runtime/symtab.go#L37)* 구조체에 맵핑되어 있다. *[runtime.moduledataverify](https://github.com/golang/go/blob/go1.5.1/src/runtime/symtab.go#L95)* 함수를 통해 이 데이타에 대한 확인이 이루어지고 오염여부와 구조가 바른지를 검사한다.

# 스택 풀 초기화하기

다음 초기화 단계를 이해하기 위해서는 Go 언어에서 스택이 어떻게 자라는지를 대한 지식이 조금 필요하다. 새로운 고루팅이 만들어 지면 작고 크기가 고정된 스택이 할당된다. 스택이 어떤 한계치에 도달하면, 크기가 두배가 되고 스택은 다른 장소로 복사된다.

아직 살펴보아야 할 자세한 내용들로 어떻게 한계치에 도달했는지를 결정하는지, Go가 어떻게 스택내 포인터들을 조정하는지가 남아 있다. 이전 포스트에서 *stackguard0* 필드와 함수 메타데이더를 얘기할 때 이런 주제에 대해 약간 다루기도 했다. 이 주제에 대해 유용한 정보는 [이 문서](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)를 통해 찾을 수 있다.

Go는 현재 이용되지 않은 스택들을 저장하기 위해 스택 풀을 이용한다. 스택 풀은 *[runtime.stackinit](https://github.com/golang/go/blob/go1.5.1/src/runtime/stack1.go#L54)* 함수내 초기화된 배열이다. 이 배열안의 각 아이템은 같은 크기를 갖는 스택들의 linked list 구조를 담고 있다.

이 단계에 초기화되는 또 다른 변수로 *runtime.stackFreeQueue* 가 있다. 이것 역시 스택들로 이루어진 linked list를 담고 있지만, 가비지 컬렉션중에 리스트에 추가되며 끝나면 지워진다. 2 KB, 4 KB, 8 KB 그리고 16 KB 크기의 스택만이 저장되고 그 보다 큰 것들은 직접 할당되는 점을 주목하라.

# 메모리 할당자 초기화하기

메모리 할당 과정은 이 [소스 코드 커밋]((https://github.com/golang/go/blob/go1.5.1/src/runtime/malloc.go#L5))에 서술되어 있다. 메모리 할당이 어떻게 작동하는지를 이해하고자 한다면 꼭 읽어 보길 권장한다. 이 주제는 앞으로 나올 포스트에서 더 자세히 다루도록 하겠다. 메모리 할당자의 초기화는 *[runtime.mallocinit](https://github.com/golang/go/blob/go1.5.1/src/runtime/malloc.go#L216)* 함수내 자리잡고 있다. 가까이 들여다 보자.

# 사이즈 클래스들(size classes)를 초기화하기

여기에 처음 볼 수 있는 것은 *runtime.mallocinit* 가 또 다른 함수-*[initSizes](https://github.com/golang/go/blob/go1.5.1/src/runtime/msize.go#L66)* 를 호출하는 것이다. 이 함수는 사이즈 클래스들를 계산하는 일을 한다. 하지만 클래스의 크기를 어떻게 결정하는가? (32 KB 가 안되는) 작은 객체를 할당할 때, Go 런타임은 처음 미리 정해둔 클래스 크기로 객체의 크기를 반올림한다. 그래서 할당된 메모리 블록은 실제 필요한 객체 크기보 보통 좀 더 큰, 미리 정해진 크기들중에 하나를 선택할 수 밖에 없다. 이로 인해 메모리가 조금 낭비되긴 하지만 이 방법을 통해서 다른 객체에 이미 할당된 메모리를 재사용하는 것이 용이해 진다.

*initSizes* 함수는 이런 클래스를 계산하는 일을 한다. 함수 꼭대기에서 다음 코드를 볼 수 있다:

>```
01     align := 8
02     for size := align; size <= _MaxSmallSize; size += align {
03         if size&(size-1) == 0 {
04             if size >= 2048 {
05                 align = 256
06             } else if size >= 128 {
07                 align = size / 8
08             } else if size >= 16 {
09                 align = 16
10 …
11             }
12         }
```

보다시피 가장 작은 사이즈 클래스들은 8과 16 바이트이다. 그 다음 클래스는 128 바이트까지 16 바이트가 올라갈 때마다 위치해 있다. 128부터 2,048 바이트까지는 size/8 바이트마다 클래스가 위치해 있다. 2,048 바이트 이후에는 256 바이트마다 사이즈 클래스가 위치해 있다.

*initSizes* 메서드는 *[class_to_size](https://github.com/golang/go/blob/go1.5.1/src/runtime/msize.go#L49)* 배열을 초기화한다. 이 배열은 하나의 클래스(여기에서 크기라 함은 클래스 리스트내 서열을 의미한다)를 그 크기로 변환한다. 이 함수는 또 *[class_to_allocnpages](https://github.com/golang/go/blob/go1.5.1/src/runtime/msize.go#L50)* 배열을 초기화하는데 이 배열은 주어진 클래스의 객체를 채우기 위해 OS로 부터 얼마나 많은 메모리 페이지를 받아내야 하는가에 대한 데이터를 저장한다. 이외 배열을 두개 더-*[size_to_class8](https://github.com/golang/go/blob/go1.5.1/src/runtime/msize.go#L53)* 와 *[size_to_class128](https://github.com/golang/go/blob/go1.5.1/src/runtime/msize.go#L54)* 를 초기화한다. 이 두 배열은 객체 크기로 부터 상응하는 클래스 서열로 변환하는데 사용되는데 첫번째 것은 1 KB 보다 작은 객체의 크기를 변환하고, 두번째는 1-32 KB 크기의 객체에 대한 변환을 맡는다.

# 가상 메모리 예약하기

*mallocinit* 함수가 다음으로 하는 일은 미래에 있을 할당들을 위해 가상 메모리를 예약하는 것이다. x64 아키텍쳐에서 이것이 어떻게 구현되었는지 알아보자. 무엇보다도 우선 다음 변수들을 초기화해야 한다:

>```
1 arenaSize := round(_MaxMem, _PageSize)
2 bitmapSize = arenaSize / (ptrSize * 8 / 4)
3 spansSize = arenaSize / _PageSize * ptrSize
4 spansSize = round(spansSize, _PageSize)
```

 * *arenaSize* 은 객체 할당의 예약에 사용될 수 있는 가상 메모리의 최대치이다. 64비트 아키텍쳐에서는 512 GB에 해당한다.
 * *bitmapSize* 는 가비지 컬렉션 (GC) 비트맵을 위해 예약될 수 있는 메모리의 양에 상응한다. GC 비트맵은 특별한 메모리 타입으로 메모리에서 포인터들이 정확히 어디에 위치하는지를 보여주는데 사용되고 이 포인터들이 가리키는 객체들이 GC에 의해 표시(mark)될 것인지의 여부를 결정하는데 사용된다.
 * *spansSize* 는 모든 메모리 스팬(memory span)들을 가리키는 포인터들의 배열을 저장하기 위해 얼마나 많은 메모리를 예약되었는지를 나타낸다. 메모리 스팬은 객체 할당을 위해 사용된 메모리 블록를 둘러 싸는 구조이다.

이 변수들이 모두 계산되면, 실제 예약이 끝나는 것이다:

>```
1 pSize = bitmapSize + spansSize + arenaSize + _PageSize
2 p = uintptr(sysReserve(unsafe.Pointer(p), pSize, &reserved))
```

마침내, 모든 메모리에 상관된 객체들을 위한 중앙 저장소로 사용될 *[mheap](https://github.com/golang/go/blob/go1.5.1/src/runtime/mheap.go#L65)* 전역 변수를 초기화 할 수 있다.

>```
1 p1 := round(p, _PageSize)
2
3 mheap_.spans = (**mspan)(unsafe.Pointer(p1))
4 mheap_.bitmap = p1 + spansSize
5 mheap_.arena_start = p1 + (spansSize + bitmapSize)
6 mheap_.arena_used = mheap_.arena_start
7 mheap_.arena_end = p + pSize
8 mheap_.arena_reserved = reserved
```

처음부터, *mheap_.arena_used* 가 *mheap_.arena_start* 와 동일한 주소를 가지고 초기화되었음을 주목하라. 이유는 아직 아무 것도 할당되지 않았기 때문이다.

# 힢(Heap) 초기화


다음으로 *[mHeap_Init](https://github.com/golang/go/blob/go1.5.1/src/runtime/mheap.go#L273)* 함수가 호출된다. 할당자의 초기화가 제일 먼저 진행되었다.

>```
1 fixAlloc_Init(&h.spanalloc, unsafe.Sizeof(mspan{}), recordspan, unsafe.Pointer(h), &memstats.mspan_sys)
2 fixAlloc_Init(&h.cachealloc, unsafe.Sizeof(mcache{}), nil, nil, &memstats.mcache_sys)
3 fixAlloc_Init(&h.specialfinalizeralloc, unsafe.Sizeof(specialfinalizer{}), nil, nil, &memstats.other_sys)
4 fixAlloc_Init(&h.specialprofilealloc, unsafe.Sizeof(specialprofile{}), nil, nil, &memstats.other_sys)
```

할당자가 무엇인가를 더 잘 이해하기 위해서, 우선 어떻게 사용되는지 살펴보자. 모든 할당자는 *[fixAlloc_Alloc](https://github.com/golang/go/blob/go1.5.1/src/runtime/mfixalloc.go#L54)* 함수 내에서 운영된다. 이 함수는 다음과 같은 구조체의 새로운 할당이 필요할때 호출된다 - *[mspan](https://github.com/golang/go/blob/go1.5.1/src/runtime/mheap.go#L101)*, *[mcache](https://github.com/golang/go/blob/go1.5.1/src/runtime/mcache.go#L11)*, *[specialfinalizer](https://github.com/golang/go/blob/go1.5.1/src/runtime/mheap.go#L1009)*, 그리고 *[specialprofile](https://github.com/golang/go/blob/go1.5.1/src/runtime/mheap.go#L1050)*. 이 함수의 주요한 부분은:

>```
1 if uintptr(f.nchunk) < f.size {
2     f.chunk = (*uint8)(persistentalloc(_FixAllocChunk, 0, f.stat))
3     f.nchunk = _FixAllocChunk
4 }
```

이 부분은 메모리를 할당하지만 구조체의 실제 크기인 *f.size* 바이트를 할당하는 대신 (현재는 16 KB)인 *[_FixAllocChunk](https://github.com/golang/go/blob/go1.5.1/src/runtime/malloc.go#L130)* 바이트를 따로 떼어 놓은다. 나머지 사용가능한 공간은 할당자내 저장된다. 다음번에 같은 타입의 구조체를 할당할 필요가 있으면 시간을 많이 소비하는 [persistentalloc](https://github.com/golang/go/blob/go1.5.1/src/runtime/malloc.go#L828)를 호출할 필요가 없게 된다.

*persistentalloc* 함수는 가비지 컬렉트되어서는 않되는 메모리를 할당하는 일을 한다. 이 함수의 웍플로우는 다음과 같다:

 1. 만약 할당된 블록이 64 KB보다 크면 OS 메모리로 부터 직접 할당된다.
 2. 그렇기 않은 경우는, 우선 지속성 할당자(persistent allocator)를 찾을 필요가 있다.
    * 지속성 할당자는 각 프로세서에 부착되어 있다. 이유는 하나만 가지고 작업할 경우 locks을 사용해야 함을 피하기 위해서이다. 그래서 현재 프로세서에 부착된 지속성 할당자를 사용하고자 하는 것이다.
    * 만약 현재 프로세서의 정보를 얻을 수 없는 경우는, 전역 할당자 (global allocator)가 사용된다.
 3. 만약 할당자가 캐쉬에 사용가능한 메모리가 충분치 않을 경우는 OS로 부터 메모리를 더 가지고 온다.
 4. 필요한 만큼의 메모리가 할당자의 캐쉬에서 반환된다.

*persistentalloc* 과 *fixAlloc_Alloc* 함수는 비슷하게 작동한다. 이 함수들은 두가지 레벨의 캐쉬를 구현하고 있다고 말할 수 있겠다. *persistentalloc* 는 *fixAlloc_Alloc* 에만 사용되는 것이 아니라 지속성 메모리(persistent memory)를 할당할 필요가 있는 여러 군데에서 사용되고 있다는 것을 숙지하여야 한다.

*mHeap_Init* 함수로 다시 돌아가자. 답을 해야할 중요한 질문이 하나 더 있는데, 그것은 이 함수의 시작부분에서 할당자가 초기화되는 목적이었더 네개의 구조가 어떻게 사용되는지에 대한 것이다:

 * *mspan* 는 가비지 컬렉트되어야 하는 메모리 블록을 감싸는 것이다. 사이즈 클래스들(size classes)을 논할 때 얘기했었다. 특정한 사이즈 클라스의 새로운 객체를 할당할 필요가 생겼을때 새로운 mspan이 만들어진다.
 * *mcache* 는 각 프로세서에 부착된 구조체이다. 메모리 스팬을 캐취에 저장하는 일을 한다. 프로세스마다 따로 캐쉬를 갖는 이유는 locking을 피하기 위해서 이다.
 * *specialfinalizeralloc* 는 *runtime.SetFinalizer* 함수가 호출될 때 할당되는 구조체이다. 객체가 제거되면서 정리정돈(cleanup)하는 코드를 실행시키길 원한다면 이 구조체를 만들어야 할 것이다. 좋은 예제로 새로운 파일마다 finalizer를 연계시키는 *os.NewFile* 함수이다. 이 finalizer가 OS 파일 설명자(file descriptor)를 닫아야 한다.
 * *specialprofilealloc* 는 메모리 프로파일러에 사용되는 구조체이다.

메모리 할당자들을 초기화하고 난 후, *mHeap_Init* 함수는 *[mSpanList_Init](https://github.com/golang/go/blob/go1.5.1/src/runtime/mheap.go#L863)* 을 호출함으로서 리스트들을 초기화 한다. 간단한 일로 linked 리스트를 위한 첫번째 입력을 초기화하는 것이다. *mheap* 구조체는 그러한 linked 리스트를 몇개 가지고 있다.

 * *mheap.free* 와 *mheap.busy* 는 *free* 와 *busy* 리스트를 담고 있는 배열들로 대형(32 KB보다 크고, 1 MB보다 작은) 객체들을 위한 메모리 스팬을 갖고 있다. 각 배열은 가능한 크기의 아이템을 가지고 있다. 여기서 크기들은 페이지로 측정된다. 한 페이지는 32 KB에 해당한다. 첫번째 아이템은 32 KB 스팬을 가지는 리스트를 담고 있고, 두번째는 64 KB 스팬을 가지는 리스트를 담고 있는 그런 식이다.
 * *mheap.freelarge* 와 *mheap.busylarge* 는 1 MB이상의 객체들을 위한 free 와 busy 리스트들이다.

다음 단계는 *mheap.central* 를 초기화하는 것인데, (32 KB 보다 작은) 소형 객체들을 위한 스팬들을 저장한다. *mheap.central* 안에서는 리스트가 사이즈 클래스에 맞게 그룹을 형성한다. 초기화는 이제까지 우리가 본 것들과 유사하다. 단순히 각 free 리스트에 대한 linked 리스트를 초기화 하는 것이다.

# 캐쉬의 초기화

이제 메모리 할당자 초기화에 대한 얘기가 거의 끝나간다. *mallocinit* 함수내 마지막으로 남은 것은 mcache 초기화이다:

>```
1 _g_ := getg()
2 _g_.m.mcache = allocmcache()
```

여기를 보면, 우선 현재의 고루틴을 얻는다. 각 고루틴은 *m* 구조체로 연결된 링크를 담고 있다. 이 구조체는 시스템 쓰레드를 감싸는 구조이다. 이 구조체의 내부에는 *mcache* 라는 필드가 있는데 위에보면 초기화되고 있다. *allocmcache* 함수는 *fixAlloc_Alloc* 을 호출하여 새로운 *mcache* 구조체를 초기화한다. 이미 어떻게 할당이 되었는지 이 구조체의 의미는 무엇인지에 대해서 논한 바 있다 (상위의 내용을 살펴보라).

조심스런 독자는 저자가 *mcache* 가 프로세스마다 부착되어 있다고 얘기한 걸 기억할 것이다. 그런데 이제 보니 프로세서(processor)가 아닌 OS 프로세스(process)에 상응하는 *m* 구조체에 부착되어 있는 것이었다. 그게 맞는 말이다-mcache는 현재 실행중인 쓰레드에 대해서만 초기화되며 프로세스가 바뀔때(process switch) 마다 또 다른 쓰레드로 재배치된다.

# 곧 추가될 Go 부트스트래핑에 대한 포스트

다음 포스트에서는 어떻게 가비지 컬랙터가 초기화되는지 긔고 어떻게 main 고루틴이 시작되는지를 살펴보며 부트스트랩 과정을 더 논하겠다. 그때까지 아래 코멘트란에 독자의 생각이나 의견을 공유하는데 주저하지 말라.

* 원문: [Golang Internals, Part 6: Bootstrapping and Memory Allocator Initialization](http://blog.altoros.com/golang-internals-part-6-bootstrapping-and-memory-allocator-initialization.html)
* 저자: Siarhei Matsiukevich
* 번역자: Jhonghee Park
