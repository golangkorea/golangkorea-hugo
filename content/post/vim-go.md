+++
date = "2016-08-22T22:57:14+09:00"
draft = false
title = "vim-go를 이용한 go 개발 환경 구축"
tags = ["Development", "vim"]
categories = ["Development", "vim"]
authors = ["Sangbae Yun"]
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
Vim의 플러그인들을 편리하게 관리하기 위해서 몇 가지 패키지 매니저들이 있다. 보통 Vundle 이나 **pathogen**을 사용한다. 나는 pathogen을 사용하고 있다. 아래와 같이 설치하자.
```sh
# mkdir -p ~/.vim/autoload ~ 
/.vim/bundle
# cd ~/.vim/autoload
# curl -LSso pathogen.vim https://tpo.pe/pathogen.vim
```

.vimrc 파일을 수정한다.
```sh
cat ~/.vimrc
execute pathogen#infect()
syntax on
filetype plugin indent on
```

이제 vim-go를 설치하자.
```sh
# cd ~/.vim/bundle
# git clone https://github.com/fatih/vim-go.git
```

Go 개발을 위한 환경 설정은 다음과 같다.
```sh
# export GOPATH=$HOME/golang 
# export PATH=$PATH:$GOPATH/bin
# mkdir $HOME/golang
# echo $GOPATH
/home/yundream/golang
# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/yundream/golang/bin....
```

vim-go 프로젝트는 구글의 mercurial에서 관리하고 있다. mercurial도 설치해야 vim-go를 빌드 할 수 있다.
```
# apt-get install mercurial
```

vim을 실행 한후 명령모드에서 **:GoInstallBinaries**를 수행하면, 자동으로 vim-go를 빌드해서 설치해준다.
```
# vim
~
~
:GoInstallBinaries
vim-go: gocode not found. Installing github.com/nsf/gocode to folder /home/yundream/.vim-go/
vim-go: goimports not found. Installing code.google.com/p/go.tools/cmd/goimports to folder /home/yundream/.vim-go/
vim-go: godef not found. Installing code.google.com/p/rog-go/exp/cmd/godef to folder /home/yundream/.vim-go/
vim-go: oracle not found. Installing code.google.com/p/go.tools/cmd/oracle to folder /home/yundream/.vim-go/
vim-go: golint not found. Installing github.com/golang/lint/golint to folder /home/yundream/.vim-go/
vim-go: errcheck not found. Installing github.com/kisielk/errcheck to folder /home/yundream/.vim-go/
vim-go: gotags not found. Installing github.com/jstemmer/gotags to folder /home/yundream/.vim-go/
계속하려면 엔터 혹은 명령을 입력하십시오
```
## Vim-go 기능 빠르게 살펴보기
Go 코드의 실행
```
:GoRun
```

빌드
```
:make
:GoBuild
```

에러체크
```
:GoErrCheck
```

패키지 임포트
```
:GoImport fmt
```

심볼에 대한 정의로 이동. 해동 심볼에서 :GoDef
```
:GoDef
```

대략 이런 식이다. 나머지 명령들은 직접 실행해 보자.

## 자동완성
자동완성은 IDE의 가장 쓸만한 기능 중 하나일 것이다. vim의  **YCM(YouCompleteMe)**를 이용해서 자동완성 기능을 추가 할 수 있다. 컴파일을 하기 때문에 python-dev와 cmake 패키지를 미리 설치해야 한다.
```sh
# cd ~/.vim/bundle
# git clone https://github.com/Valloric/YouCompleteMe.git
# cd YouCompleteMe
# ./install.sh
```

이제 자동완성 기능을 사용 할 수 있다. 아래 화면을 보자.

![YCM 자동완성](https://lh5.googleusercontent.com/-n9eUbylZw50/U_C_wglq1aI/AAAAAAAAEQI/IsIWi6MYrdc/s640/golang-2.png)

YCM은 C, C++, Python, Java 등에도 사용 할 수 있다. 
## TagBar 설치
ctags는 코드에 포함된 패키지, struct, 메서드의 목록을 한눈에 보여주는 애플리케이션이다. ctags를 설치하자. tagbar는 ctags를 기반으로 작동하는 플러그인이다.
```sh
# apt-get install ctags
```

tagbar 플러그인을 설치한다.
```sh
# cd ~/.vim/bundle
# git clone https://github.com/majutsushi/tagbar.git
```
이제 **:TagbarToggle** 명령으로 tagbar 네비게이션 창을 열고 닫을 수 있다. 

![TagBar 적용](https://lh5.googleusercontent.com/-xO-ZcWBjqfQ/U_NybrcP-FI/AAAAAAAAEQ8/VjcKCUsrIE0/s640/golang-3.png)

명령어를 입력하기 귀찮다면, 단축 키를 만들자.
```sh
# cat .vimrc
......
map <F8> :TagbarToggle<CR>
```

## NerdTree 설치
NerdTree는 파일 네비게이션을 만들어주는 플러그인다.
```sh
# cd ~/.vim/bundle
# git clone https://github.com/scrooloose/nerdtree.git 
```

NerdTree와 TagBar를 적용한 화면이다.

![NerdTree와 TagBar 적용](https://lh6.googleusercontent.com/-yfhTJc0-xMc/U_N0PxZXmEI/AAAAAAAAERE/2n5LUOmtQGw/s640/golang-4.png)

명령을 일일이 입력하기가 귀찮아서 단축키를 등록했다.
```sh
# cat ~/.vimrc
set ts=4
 
map <F8> :NERDTreeToggle<CR>
map <F2> :GoDef<CR>
map <F4> :TagbarToggle<CR>
```
