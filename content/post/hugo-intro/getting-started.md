+++
date = "2016-08-23T23:24:55-04:00"
draft = false
title = "시리즈 #1 - Hugo 시작하기"

tags = ["Blog", "Hugo"]
categories = ["How-to"]
series = ["Hugo 입문하기"]
authors = ["Jhonghee Park"]

toc = true
+++

# Hugo 시작하기

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

정적 사이트 제너레이터를 처음 접하시는 분들을 위해 Hugo를 간단하게 설명하자면, Hugo는 소스 폴더 아래 존재하는 파일과 컨텐츠 템플릿을 입력으로 사용해서 웹사이트 전체를 출력하는 시스템입니다. 보통 소스는 [Markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)을 이용한 컨텐츠이거나 템플릿 언어로 작성된 HTML에 자바스크립과 CSS스타일로 구성되어 웹 개발자에게 매우 친숙한 환경이라 할 수 있습니다.

```
+------------------+
|    Content       +--------+
|    (Markdown)    |        |
+------------------+        |           +------------+
+------------------+     +--v---+       |Full Website|
|Template          |     |      |       +-------+----+
|(text/template)   |     | Hugo |       |       |    |
|(Ace)             +----->      +------->       |    |
|(Amber)           |     |      |       |       |    |
+------------------+     +--^--^+       |       |    |
+------------------+        |  |        |       |    |
|Configuraton      |        |  |        +-------+----+
|(toml, yaml, json)+--------+  |
+------------------+           |
+------------------+           |
| Static           |           |
| (image)          |           |
| (javascript)     +-----------+
| (css)            |
+------------------+
```

Hugo는 커맨드라인 명령어 체계는 각종 보조 명령어와 POSIX를 준수하는 플래그로 구성되어 빌드와 유틸리티 기능을 제공합니다. 우선 Hugo가 제공하는 Scaffolding 명령어를 가지고 프로젝트 폴더를 만들어 보도록 합니다.

```
$ hugo new site golangkorea-hugo
```
Hugo의 모든 명령은 `hugo`로 시작하고 보조 명령어가 뒤를 따릅니다. 여기서 `new`는 보조 명령어로서 `site` 보조 명령어와 함께 프로젝트를 초기화합니다. 초기화된 프로젝트의 폴더 구조는 다음과 같습니다.
```
$ cd golangkorea-hugo
$ tree -a
.
├── archetypes
├── config.toml
├── content
├── data
├── layouts
├── static
└── themes

6 directories, 1 file
```
초기화된 프로젝트에는 텅빈 폴더 6개와 `config.toml` 파일 하나가 만들어 집니다. 각 폴더의 용도를 간단히 나열하면 다음과 같습니다.

 * archetypes: `hugo new`명령으로 컨텐트 생성시 [Front Matter](https://gohugo.io/content/front-matter/)<sup>1</sup> 에 컨텐트 타입에 따른 기본 값들을 어떻게 정해줄 것인가를 결정하는 파일들을 저장합니다.
 * content: 컨텐츠가 저장됩니다.
 * data: 템플랫으로 불러쓸 수 있는 데이터 파일을 저장하는 공간입니다. 데이터의 타입은 toml, yaml, 과 json이 지원됩니다.
 * layouts: 테마를 커스터마이징할 때 기존의 테마내 탬플릿의 내용을 수정하거나 덧씌우기를 하는 템플릿을 저장하는 공간입니다.
 * static: 이미지, 자바스크립, CSS등을 저장하는 공간
 * themes: 사이트의 테마를 저장하는 공간.

config.toml의 내용을 보면 다음과 같습니다.
```
baseurl = "http://replace-this-with-your-hugo-site.com/"
languageCode = "en-us"
title = "My New Hugo Site"
```
`baseurl`은 말 그대로 사이트내 모든 리소스의 URL의 베이스를 형성합니다. 예를 들어 `content/post/my-first-blog.md`라는 컨텐트가 있으면 Full URL은 `http://replace-this-with-your-hugo-site.com/post/my-first-blog`이 됩니다.

# 첫번째 컨텐트 만들기

그럼, 다음 Scaffolding 명령을 써서 첫번째 블로그 포스트를 만들어 보도록 합니다.
```
$ hugo new post/my-first-blog.md
```
`content/post/my-first-blog.md`가 만들어 지면 아래와 같이 편집을 하고 저장하십시요.
```
+++
date = "2016-08-24T21:51:10-04:00"
draft = true
title = "my first blog"

+++

# Hello, Hugo! <- 편집 부분
```
`hugo server`명령을 써서 Hugo가 제공하는 웹서버를 구동한 다음 `http://localhost:1313`을 브라우저로 열어 보십시요. 텅빈 페이지로 나타날 겁니다. 왜 그럴까요? 답 부터 말씀드리면 Hugo의 입장에서는 무엇으로 페이지를 렌더링할 지 아무런 정보가 없는 경우인 것입니다. `layouts` 폴더안에 `index.html`이라는 파일을 만들고 다음과 같이 편집해 저장하신 다음 다시 `http://localhost:1313`을 열어 보십시요.
```
<h1>Hello, Hugo!</h1>
```
**Hello, Hugo!**라고 크게 나타나는 것을 보게 될 것입니다.

이제 `http://localhost:1313/post/my-first-blog`를 열어 보십시요. 심지어 `404 page not found`라고 나옵니다. 텅빈 페이지가 아니고 왜 404일까요? 이유는 포스트의 Front Matter에 `draft = true`라고 명시되어 있어서 Hugo의 입장에서는 렌더링을 할 이유가 없는 것이죠. `Ctrl-C`로 Hugo 웹서버를 중단시킨 다음 `hugo server -D=true`명령을 써서 다시 웹서버를 가동시키시고 `http://localhost:1313/post/my-first-blog`를 열어 보십시요. 이번에는 404가 아니고 텅빈 페이지가 보일 겁니다. Hugo를 의인화해서 다시 설명을 드리면, `-D=true` 플래그를 보고 드래프트 포스트도 렌더링을 해야 하는데 어떻게 해야 할 지 몰라 백지를 낸 상황인 겁니다. 이건 어떻게 고쳐야 할까요?

`layouts/post/single.html`라는 파일을 만드시고 다음의 내용을 저장하신 다음, `http://localhost:1313/post/my-first-blog`을 열어 보세요.
```
<p>Before content</p>
{{ .Content }}
<p>After content</p>
```
첫번째 포스트가 이제 보이십니까?

# 이렇게 힘들게 만들어야 하나?
이런 질문이 당연히 생기실 겁니다. 사이트의 구조와 컨텐츠의 템플릿을 하나씩 만들어 나가야 한다면 사이트 발생기라고 부를 이유가 없겠죠. 누군가 그런 힘든 노동을 통해 `layouts`의 구조와 템플랫을 모두 작성했다면 공유할 수 있는 메카니즘이 필요합니다. 그런 공유의 매카니즘을 `테마(theme)`이라고 부릅니다.

이제 `layouts/index.html`과 `layouts/post/single.html`을 제거하시고 테마를 사용하는 방법을 배워 봅시다. 다음과 같이 `hugo-octopress` 테마를 설치하십시요.
```
$ rm layouts/index.html
$ rm layouts/post/single.html
$ cd themes
$ git clone https://github.com/parsiya/Hugo-Octopress.git
$ cd ..
```
테마가 설치된 후에는 Hugo의 웹서버를 다음과 같이 시작해 보십시요.
```
$ hugo server -D=true -t=hugo-octopress
```
Hugo로 만든 당신의 첫번째 포스트가 보일 겁니다.

![image](https://cloud.githubusercontent.com/assets/211484/17955233/9990f3c8-6a4e-11e6-8d3e-0c824453ba1f.png)




<br/>

<hr/>
`1. 컨텐트 인스턴스의 메타데이터로 템플릿에서 호출해 쓸 수 있습니다.`
<br/>
<br/>
<br/>
