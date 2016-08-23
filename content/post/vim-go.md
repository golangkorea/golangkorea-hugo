+++
date = "2016-08-22T22:57:14+09:00"
draft = true
title = "vim-go를 이용한 go 개발 환경 구축"
tags = ["Development", "vim"]
categories = ["Development", "vim"]
author = "Sangbae Yun yundream@gmail.com"
+++
## Vim
**Vim**은 Emacs와 함께 (적어도 리눅스에서는) 가장 널리 사용하는 에디터일 것이다. 가볍고 빠르며, 어디에서나 실행되기 때문에 그 단순함에도 불구하고 여전히 사랑받고 있다. GUI 환경에서 사용하는 IDE에 익숙한 개발자라면 "요즘 같은 시대에 왠 구닥다리 터미널 기반 에디터냐"라고 생각할 지도 모르겠다. 아래 그래프를 보자. 

![Go 에디터 사용율](https://i.redditmedia.com/Zemj1bdTRcBwW8bF_UFEVSNZ9S1VrS4tsD4HC1b9jeI.jpg?w=844&s=1fbbaa5fe7f8ba1ba0942191327ffd70)

Go언어를 대상으로 조사한 결과인데, Vim이 거의 40% 정도를 차지하고 있다. Emacs까지 하면 터미널 기반 에디터를 사용하는 개발자가 절반이 넘는다. 물론 Go 언어가 시스템과 네트워크 분야의  백앤드 프로그램의 개발에 특화된 측면을 고려해야 겠지만 말이다. 

## Vim-go
Vim은 다양한 플러그인을 제공한다. **Vim-go**는 Go 개발환경을 지원하는 플러그인이다. 지원하는 기능은 아래와 같다.

  * 함수, 오퍼레이터, 메서드들에 대한 Syntax highlighting 
  * **gocode**를 이용한 자동완성
  * **:GoDef**를 이용해서 메서드, 변수들의 선언 위치를 네비게이션 할 수 있다. 
  * **:GoImport**를 이용한 패키지 임포트
  * **:GoTest**와 **:GoTestFunc**를 이용한 유닛 테스트
  * 테스트 커버리지를 위한 **:GoCoverage**
  * **:GoBuild**, **:GoInstall**을 이용한 패키지 컴파일과 설치
  * **:GoRun**을 이용한 빠른 실행
  * 소스 분석을 위한 **:GoImplements**, **:GoCallee**, **:GoReferrer**
  * Lint툴 **:GoLint**
  * **:GoPlay**로 코드를 [play.golang.org](https://play.golang.org) 로 공유
등 개발 환경을 만들기 위한 거의 모든 기능들을 제공한다. 여기에 파일 네비게이션 플러그인, 자동완성 플러그인들을 추가로 설치하면, IDE 부럽지 않은 개발 환경을 만들 수 있다. 

## Vim-go 설치
