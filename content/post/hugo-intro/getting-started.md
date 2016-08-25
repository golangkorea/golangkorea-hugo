+++
date = "2016-08-23T23:24:55-04:00"
draft = true
title = "시리즈 #1 - Hugo 시작하기"

tags = ["Blog", "Hugo"]
categories = ["How-to"]
series = ["Hugo 입문하기"]
authors = ["Jhonghee Park"]

toc = true
+++

Hugo 시작하기
===========

[Hugo](https://gohugo.io)는 Go로 제작되고 하나의 실행파일로 배포됩니다. 다양한 설치 방법이 있지만 우선 Package Manager를 쓰시는 분들을 중심으로 살펴보겠습니다.

## Package Manager로 설치하기

MacOS를 쓰시는 분들은 brew를 이용해 쉽게 설치하실 수 있습니다.

```
brew update && brew install hugo
```

Windows에서 Chocolatey를 쓰시는 분들도 비슷한 방법으로 설치가 가능합니다.
```
C:\> choco install hugo
```

Linux에서는 조금 복잡해 집니다. 우분트를 쓰시는 분들은 우선 [Hugo 릴리즈 페이지](https://github.com/spf13/hugo/releases)로 가서 최신 deb 버전을 다운로드한 후에 다음 명령을 실행 시키면 됩니다.
```
sudo dpkg -i hugo*.deb
```

## 소스로 직접 빌드해 쓰는 방법
이미 Go로 개발 환경을 갖추고 계신 분들은 직접 소스를 빌드해 쓰시는 방법이 가장 편합니다. 간단히 `go get`툴을 이용해 설치하실 수 있습니다.
```
go get -v github.com/spf13/hugo
```

Hugo가 설치되었는지를 `version` 보조 명령어를 사용해 확인하십시요.
```
$ hugo version
Hugo Static Site Generator v0.17-DEV BuildDate: 2016-08-21T19:44:40-04:00
```

# 프로젝트 폴더 만들기

정적 사이트 제너레이터를 처음 접하시는 분들을 위해 Hugo를 간단하게 설명하자면, Hugo는 소스 폴더 아래 존재하는 파일과 컨텐츠 템플릿을 입력으로 사용해서 웹사이트 전체를 출력하는 시스템입니다. 보통 소스는 [Markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)을 이용한 컨텐츠이거나 템플릿 언어로 작성된 HTML에 자바스크립과 CSS스타일로 구성되어 웹 개발자에게 매우 친숙한 환경이라 할 수 있습니다. Hugo는 커맨드라인 명령어 체계는 각종 보조 명령어와 POSIX를 준수하는 플래그로 구성되어 빌드와 유틸리티 기능을 제공합니다. 우선 Hugo가 제공하는 Scaffolding 명령어를 가지고 프로젝트 폴더를 만들어 보도록 합니다.

```
$ hugo new site golangkorea-hugo
```
