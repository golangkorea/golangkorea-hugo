+++
date = "2016-08-23T23:25:30-04:00"
draft = false
title = "시리즈 #3 - 사이트 테마 개발하기"

tags = ["Blog", "Hugo"]
categories = ["How-to"]
series = ["Hugo 입문하기"]
authors = ["Jhonghee Park"]

toc = true
+++

# 사이트 테마 개발하기

[시리즈 1](/post/hugo-intro/getting-started) 마지막에 [hugo-octopress](http://themes.gohugo.io/hugo-octopress/) 테마를 사용하여 처음으로 사이트를 구축한 기억을 하실 겁니다. 사이트를 구축하기 전에 Hugo에서 사용할 수 있는 테마가 어떤것 있는지 한번 살펴보고 생각하고 있는 사이트와 잘 맞는 테마를 선택하는 일도 중요합니다. Hugo의 테마 쇼케이스에서 한번 감상하시길 바랍니다.

* [Hugo 테마 쇼케이스](http://themes.gohugo.io/)

자신만의 테마를 개발하기 위한 첫걸음은 남이 개발해 놓은 테마를 사용하는 것에서 부터 시작합니다.

# 테마 설치하는 법
테마의 사용법은 대단히 간단합니다. `hugo new site`명령으로 사이트 프로젝트를 초기화하면 `themes` 폴더가 생기는 것을 이미 아실 것입니다. 사용하고자 하는 테마를 선택하고 난 뒤 `themes` 폴더 밑에 다운로드 받은 테마 패키지를 설치해 주면 됩니다.

**테마의 깃헙주소를 아는 경우**
```
$ cd themes
$ git clone https://github.com/parsiya/Hugo-Octopress.git
```

**테마 쇼케이스의 모든 테마를 설치하는 경우**
```
$ git clone --recursive https://github.com/spf13/hugoThemes.git themes
```

# 테마 사용법
테마를 `themes`에 설치하고 난 뒤 다음과 같이 선택한 테마로 사이트를 구축할 수 있습니다.
```
$ hugo -t ThemeName
```
ThemeName은 `themes` 폴더내 설치된 테마의 디렉토리 이름과 일치하여야 합니다.

# 테마 커스터마이징
설치된 테마를 사용하다 보면 맘에 들지 않는 부분이 생기거나 부족한 부분을 발견할 지도 모릅니다. **_어떤 경우가 생기더라도 `themes`밑에 설치된 테마에 속한 파일들을 직접 편집하지 마십시요_**. Hugo는 이러한 경우가 생길때 보충하거나 기존의 템플릿을 오버라이드할 수 있도록 허락합니다. 설치된 테마의 구조를 살펴보시면 힌트를 얻을 수 있습니다.

```
.
└── Hugo-Octopress
    ├── LICENSE.md
    ├── README.md
    ├── images
    │   ├── Thumbs.db
    │   ├── codecaption1.png
    │   ├── screenshot.png
    │   └── tn.png
    ├── layouts
    │   ├── 404.html
    │   ├── _default
    │   │   ├── list.html
    │   │   ├── single.html
    │   │   └── terms.html
    │   ├── index.html
    │   ├── indexes
    │   │   ├── category.html
    │   │   └── tag.html
    │   ├── license
    │   │   └── single.html
    │   ├── partials
    │   │   ├── disqus.html
    │   │   ├── footer.html
    │   │   ├── header.html
    │   │   ├── navigation.html
    │   │   ├── octo-header.html
    │   │   ├── pagination.html
    │   │   ├── post_footer.html
    │   │   ├── post_header.html
    │   │   └── sidebar.html
    │   ├── post
    │   │   └── single.html
    │   └── shortcodes
    │       ├── codecaption.html
    │       └── imgcap.html
    ├── sample-config.toml
    ├── static
    │   ├── css
    │   │   └── hugo-octopress.css
    │   └── favicon.png
    └── theme.toml
```

지금 개발하고 있는 사이트 프로젝트 폴더 밑으로 `layouts`, `static`, `archetypes`가 있듯이 테마도 같은 구조를 가지고 있습니다.

## 정적 리소스를 교체하는 법
테마에 따라온 jQuery버전이 맘에 안 드시나요? 교체하는 법은 기존의 것은 그대로 두고 새로운 jQuery버전을 다운로드 받아 테마에 위치한 장소와 같은 상대적 경로에 배치하면 Hugo는 테마의 jQuery를 사용하지 않고 프로젝트에 배치된 것을 사용해서 사이트를 구축합니다.

**테마 밑**
```
/themes/themename/static/js/jquery.min.js
```

**프로젝트 Root**
```
/static/js/jquery.min.js
```

## 템플릿 교체하는 법
Hugo는 탬플릿을 찾을 때 항상 프로젝트 밑 `layouts`폴더를 먼저 검색하고 없으면 테마의 `layouts`을 찾아봅니다. 이런 Hugo의 특성을 이용하여 테마의 템프렛을 수정해야 할 필요가 생길 때 `layouts`폴더 밑으로 같은 경로와 이름의 템플릿으로 교체하면 됩니다.

**테마 밑**
```
/themes/themename/layouts/_default/single.html
```

**프로젝트 Root**
```
/layouts/_default/single.html
```
특히 부분 템플릿(partial template)을 잘 활용한 테마의 경우 이런 교체법은 사이트의 보수유지를 최소화하고 미래에도 호환성을 보장하게 해 주는 장점을 가지고 있습니다.

## archetype 교체하는 법
Archetype을 제공하는 테마의 경우 `archetypes`폴더로 교체하고자 하는 archetype 파일을 복사한 다음 필요에 맞게 수정하면 됩니다.

## Default 템플릿 사용 주의
`layouts/_default`폴더내의 템플릿들은 테마에 비슷한 파일들을 가리지 않고 교체하는 효과가 있어 사용을 자제해야 합니다. 항상 default 템플릿을 사용하기 보다는 특정한 템플릿 교체가 더 낫다는 사실을 명심하십시요.

# 새 테마 만들기
새로 테마를 만들기 위한 명령어는 다음과 같습니다.
```
$ hugo new theme golangkorea
```
이 명령은 `themes`폴더 안에 다음과 같이 테마구조를 발생시킵니다.
```
golangkorea
├── LICENSE.md
├── archetypes
│   └── default.md
├── layouts
│   ├── 404.html
│   ├── _default
│   │   ├── list.html
│   │   └── single.html
│   ├── index.html
│   └── partials
│       ├── footer.html
│       └── header.html
├── static
│   ├── css
│   └── js
└── theme.toml
```
템플릿은 Go의 템플릿 언어로 만들어 집니다. [Go template primer](https://gohugo.io/layout/go-templates/)은 Go 템플릿 언어를 숙지하기 위한 좋은 출발점입니다.

## 테마 콤퍼넌트

* **Layouts** 근본적으로 웹사이트는 두가지 형식의 페이지를 통해 컨텐트를 제공합니다: 컨텐트 자체가 하나의 페이지인 경우와 여러 항목을 나열해 놓은 페이지. Hugo의 테마는 이 두가지 페이지를 처리하는 기본(default) 템플릿에서 시작해서 컨텐트 타입(type)과 section을 통해 추가로 layout을 제공하는 템플릿을 준비합니다.
* **Single Content** 기본 템플릿은 `layouts/_default/single.html`에 위치합니다.
* **List of Contents** 기본 템플릿은 `layouts/_default/list.html`에 위치합니다.
* **Partial Templates** 부분 템플릿은 테마 제작에 있어 매우 중요한 요소입니다. 부분 템플릿을 통해 코드 재사용이 가능하고 아주 작은 부분만 교체하거나 삽입할 수 있게 해 주는 메카니즘이어서 테마의 유지보수가 간단해 지고 미래의 호환성을 보장해 줍니다.
* **Static** 테마내 정정 리소스의 구조는 전적으로 개발자에게 달려 있습니다. 보통은 `/css`, `js`, `img`와 같은 폴더를 이용해 정적 자원을 관리합니다.
* **Archetypes** 특정한 컨텐트 타입의 정면 변수를 정의하는 archetype을 테마에 포함시킬 수 있습니다. 자세한 내용은 [Archetype Guideline](https://gohugo.io/content/archetypes/)을 확인할 수 있습니다.
* **Generator meta tag** 테마 개발자들에게 HTML의 `<head>`에 Generator meta tag, `.Hugo.Generator`을 포함시킬 것을 권유합니다. Hugo의 사용과 인기도를 가늠하는데 도움을 줍니다.

<br/>
<br/>
