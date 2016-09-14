+++
title = "Golang의 내부, 1부: 주요 컨셉트와 프로젝트 구조"
draft = true
date = "2016-09-13T13:18:28-04:00"

tags = ["Golang", "Internals", "Compiler", "Structure"]
categories = ["번역"]
series = ["Golang  Internals"]
authors = ["Jhonghee Park"]

toc = true
+++

이 블로그 시리즈는 기본적인 Go 언어특성에 이미 익숙하며 좀 더 심도있게 내부구조를 알고자 하는 독자들을 위해 쓰여졌다. 이 포스트는 Go언어의 소스코드의 구조와 Go 컴파일러의 내부를 어느 정도 상세히 살펴보겠다. 이 글을 읽고 난 후, 독자는 다음과 같은 질문에 답을 얻을 것이다.

  1. Go의 소스코드는 어떤 구조를 가지고 있는가?
  2. Go의 컴파일러는 어떻게 동작하는가?
  3. Go의 노드 트리의 기본 구조는 무엇인가?

# 시작하며

새로운 프로그래밍 언어를 배우기 시작할 때, 보통은 "hello-world"류의 튜토리얼이나, 초보자 가이드, 그리고 언어의 주요한 컨셉트, 문법, 심지어 표준 라이브러리에 대한 상세한 정보들 많이 접하게 된다. 하지만, 언어가 런타임 도중에 할당하는 주요한 데이터 구조의 레이아웃이라던지, 내장 함수를 호출할 때 어떤 어셈블리 코드가 발생하는지와 같은 정보를 얻는 것은 쉽지 않다. 물론, 답은 소스코드내에 자리잡고 있지만, 저자의 경험에 비추어 보면, 이렇다 할 성과없이 수많은 시간을 허비하는 일도 가능하다.  

이런 주제에 대해 전문가인 척 하지도 않을 거고, 모든 가능한 측면을 설명하려는 시도 또한 하지 않겠다. 대신, 목표하는 바는 독자들 스스로 Go 소스코드를 어떻게 해독해 나갈 수 있는 지를 보여주는 것이다.

시작하기 전에, 반드시 필요한 것은 각자의 Go 소스코드 복사본을 갖는 것이다. 소스코드를 다운 받는데 특별할 게 전혀 없다. 다음 명령을 실행해 보자.

```
git clone https://github.com/golang/go
```

메인 브랜치의 코드는 상시 변하는 점을 주목하자. 그래서 저자는 이 블로그에서 release-branch.go1.4 브랜치를 사용한다.

# 프로젝트 구조 이해하기

Go 레포의 `/src` 폴더를 보게 되면, 많은 폴더를 발견하게 된다. 대부분은 Go의 표준 라이브러리 소스 파일을 갖고 있다. 여기에도 표준 이름짓기 관행이 항상 적용되는데, 각 패키지는 그 이름에 상응하는 이름의 폴더 아래 있다. 표준 라이브러리외에 다른 것들을 살펴보자. 저자의 견해로는, 아래 폴더들이 가장 중요하고 유용하다.

<style type="text/css"><!--
.myTable { background-color:white;border-collapse:collapse; } .myTable th { background-color:#E0E0E0;color:black; } .myTable td, .myTable th { padding:5px;border:1px solid #989898; }
--></style>

<table class="myTable">
<tbody>
<tr>
<th><center>폴더</center></th>
<th style="width: 530px;" width="70%"><center>설명</center></th>
</tr>
<tr>
<td><a href="https://github.com/golang/go/tree/release-branch.go1.4/src/cmd" target="_blank">/src/cmd/</a></td>
<td>코멘드 라인 툴들을 보관한다.</td>
</tr>
<tr>
<td><a href="https://github.com/golang/go/tree/release-branch.go1.4/src/cmd/go" target="_blank">/src/cmd/go/</a></td>
<td>Go 툴의 소스 파일이 있는데, 이 툴들은 Go 소스코드를 다운받거나, 빌드하고, 설치하는데 사용된다. 툴이 실행되면서 전체 소스를 수집하고, Go 링커와 컴파일러 코멘드 라인 툴들을 호출한다.</td>
</tr>
<tr>
<td><a href="https://github.com/golang/go/tree/release-branch.go1.4/src/cmd/dist" target="_blank">/src/cmd/dist/ </a></td>
<td>다른 코멘트 라인 툴과 표준 라이브러리의 모든 패키지를 빌드하는 툴을 보관한다. 모든 특정한 툴이나 패키지에서 어떤 라이브러리가 사용되었는지를 알아 보려면 이 소스를 분석하고 싶을 것이다.</td>
</tr>
<tr>
<td><a href="https://github.com/golang/go/tree/release-branch.go1.4/src/cmd/gc" target="_blank">/src/cmd/gc/</a></td>
<td>Go 컴파일러내 (프로세서) 아키텍쳐에 의존하지 않는 부분이다.</td>
</tr>
<tr>
<td><a href="https://github.com/golang/go/tree/release-branch.go1.4/src/cmd/ld" target="_blank">/src/cmd/ld/</a></td>
<td>Go 링커내 (프로세서) 아키텍쳐에 의존하지 않는 부분이다. 아키텍쳐에 의존적인 부분들은 "l"로 끝나는 이름의 폴더에 위치하며 컴파일러와 같은 이름짓기 관행을 따른다.</td>
</tr>
<tr>
<td><a href="https://github.com/golang/go/tree/release-branch.go1.4/src/cmd/5a" target="_blank">/src/cmd/5a/</a>, 6a, 8a, and 9a</td>
<td>여러 아키텍쳐에 맞춘 Go 어셈블러 컴파일러들을 발견할 수 있다. Go 어셈블러는 일종의 어셈블리 언어로 하층 기계어와는 딱 맞아 떨어지는 것은 아니다. 대신 각 아키텍쳐마다 독특한 컴파일러들이 있어 Go의 어셈블러를 기계의 어셈블러도 번역한다. 더 자세한 내용은 다음 링크를 참조하라. <a href="https://golang.org/doc/asm" target="_blank">여기</a>.</td>
</tr>
<tr>
<td><a href="https://github.com/golang/go/tree/release-branch.go1.4/src/lib9" target="_blank">/src/lib9/</a>, <a href="https://github.com/golang/go/tree/release-branch.go1.4/src/libbio" target="_blank">/src/libbio</a>, <a href="https://github.com/golang/go/tree/release-branch.go1.4/src/liblink" target="_blank">/src/liblink</a></td>
<td>컴파일러, 링커, 그리고 런타임 패키지에 사용된 각종 라이브러리들.</td>
</tr>
<tr>
<td><a href="https://github.com/golang/go/tree/release-branch.go1.4/src/runtime" target="_blank">/src/runtime/</a></td>
<td>가장 중요한 Go 패키지로 모든 프로그램에 간접적으로 포함된다. 메모리 관리, 가비지 콜렉션, Go 루틴 생산등, 런타임 기능 전체를 포함하고 있다.</td>
</tr>
</tbody>
</table>

# 컴파일러 내부

위에서 언급한 것 처럼, 아키텍쳐에 무관한 Go 컴파일러는 `/src/cmd/gc` 폴더에 위치 한다. 시작 점은 `lex.c` 파일에 있다. 코멘드 라인 인수 파싱 같은 보편적인 기능들을 차치하고 들여다 보면, 컴파일러는 다음과 같은 일들을 한다.

  1. 어떤 공통의 데이터 구조를 초기화 한다.

  2. 주어진 모든 Go 파일을 차례로 읽어서 각 파일에 yyparse 메서드를 호출한다. 이때 실제로 파싱이 작동한다. Go 컴파일러는 `Bison`을 파서 발생기(parser generator)로 사용한다. 언어의 문법은 [go.y](https://github.com/golang/go/blob/release-branch.go1.4/src/cmd/gc/go.y) 완전히 서술되어 있다. (자세한 내용은 나중에 더 제공될 예정이다) 결과로, 이 단계는 완전한 파스트리(parse tree)를 생성하는데, 이때 트리의 각 노드는 컴파일된 프로그램의 요소들을 대표한다.

  3. 생성된 트리를 재귀적으로(Recursively) 방문하면서 약간의 수정을 가한다, 예를 들어, 암시적으로 타입이 주어진 노드에 타입 정보를 정의하거나, 타입 케스팅과 같은 언어요소들을 런타임 패키지내 어떤 함수을 호출하는 식으로 다시 재구성하기도 한다 그외 다른 일들도 실행한다.

  4. 파스트리(parse tree)가 완성되고 난 뒤 실제 컴파일을 실행한다. 노드들은 어셈블리 코드로 번역된다.

  5. 생성된 어셈블리 코드에 심볼 테이블과 같은 부수적인 데이터 구조를 함께 객체 파일 (object file)에 담아 만들고 디스크에 저장한다.

# Go 문법 들여다 보기

이제 두번째 단계를 좀 더 가까이 살펴보자. [go.y](https://github.com/golang/go/blob/release-branch.go1.4/src/cmd/gc/go.y) 파일은 언어 문법(grammar)을 가지고 있어 Go 컴파일러를 조사하고 언어의 구문론(syntax)을 이해하는 데 좋은 출발점이다. 파일의 주요한 부분은 선언문들로 구성되며, 다음과 유사하다.

```
xfndcl:
     LFUNC fndcl fnbody

fndcl:
     sym '(' oarg_type_list_ocomma ')' fnres
| '(' oarg_type_list_ocomma ')' sym '(' oarg_type_list_ocomma ')' fnres
```


이 선언에서는, *xfndcl* 와 *fundcl* 노드가 정의된다. *fundcl* 노드는 두가지 형식중에 하나이다. 첫번째는 다음의 언어 구성소(construct)에 상응하는 형식이다:

```
somefunction(x int, y int) int
```

그리고 두번째는 다음의 언어 구성소에 상응한다:

```
(t *SomeType) somefunction(x int, y int) int.
```

*xfndcl* 노드는 상수인 *LFUNC* 에 저장된 키워드 *func* 를 뒤 따르는 *fndcl* 와 *fnbodynodes* 로 구성되어 있다.

Bison(혹은 Yacc) 문법의 중요한 기능중에 하나는 무작위의 C 코드를 각 노드 정의옆에 갖다 붙일 수 있다는 것이다. 소스 코드안에 이런 노드의 정의가 매치될 때 마다 C 코드는 실행된다. 여기서, (실행된)결과의 노드는 $$ 사용해 표시하고 $1, $2 등등으로 자식 노드를 나타낸다.

예제를 통해 보면 다 쉽게 이해할 수 있다. 다음은 실제코드를 간소한 예다.

```
fndcl:
      sym '(' oarg_type_list_ocomma ')' fnres
        {
          t = nod(OTFUNC, N, N);
          t->list = $3;
          t->rlist = $5;

          $$ = nod(ODCLFUNC, N, N);
          $$->nname = newname($1);
          $$->nname->ntype = t;
          declare($$->nname, PFUNC);
      }
| '(' oarg_type_list_ocomma ')' sym '(' oarg_type_list_ocomma ')' fnres
```

우선, 새로운 노드가 만들어 지고 함수 선언을 위한 타입 정보를 갖는다. $3는 인수 리스트로 $5는 결과 리스트로 이 노드에서 레퍼런스된다. 그런 다음, $$ 결과 노드가 만들어 져서, 함수 이름과, 타입 노드를 저장한다. 보다시피 [go.y](https://github.com/golang/go/blob/release-branch.go1.4/src/cmd/gc/go.y)내 정의된 것들과 노드 구조사이에 직접적인 연결이 있을 수 없다.

# 노드 이해하기

Now it is time to take a look at what a node actually is. First of all, a node is a struct (you can find a definition here). This struct contains a large number of properties, since it needs to support different kinds of nodes and different nodes have different attributes. Below is a description of several fields that I think are important to understand.

Node struct field
Description
op	Node operation. Each node has this field. It distinguishes different kinds of nodes from each other. In our previous example, those were OTFUNC (operation type function) and ODCLFUNC (operation declaration function).
type	This is a reference to another struct with type information for nodes that have type information (there are no types for some nodes, e.g., control flow statements, such as if, switch, or for).
val	This field contains the actual values for nodes that represent literals.
Now that you understand the basic structure of the node tree, you can put your knowledge into practice. In the next post, we will investigate what exactly the Go compiler generates, using a simple Go application as an example.
