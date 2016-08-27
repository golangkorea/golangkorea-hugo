+++
date = "2016-08-23T23:25:04-04:00"
draft = false
title = "시리즈 #2 - 컨텐츠 제작 기초"
description = "Hugo 입문 두번째 시리즈로 컨텐츠 제작과 관련해 꼭 알아야 할 개념들을 소개합니다"

tags = ["Blog", "Hugo"]
categories = ["How-to"]
series = ["Hugo 입문하기"]
authors = ["Jhonghee Park"]

toc = true

+++

# 컨텐츠 제작 기초

컨텐츠를 제작하면서 꼭 알아야 할 몇가지 개념을 정리하겠습니다.

# 컨텐츠의 조직적인 관리 (Organization)

사이트가 많은 양의 컨텐츠를 보유하게 되면서 조직적인 관리가 필요할 때 Hugo가 어떻게 도와주는지 알아 봅시다. [시리즈 1](/post/hugo-intro/getting-started/)에서 보았 듯이 Hugo의 `configuration`<sup>1</sup>에 특별한 세팅이 없는 한 모든 컨텐츠는 `content` 폴더 안에 위치하게 됩니다. Hugo를 통해 만들어질 사이트의 URL은 `content`내의 폴더 구조와 매우 밀접한 관계가 있습니다. 우선 `content` 바로 아래 위치하는 폴더는 `section`이라고 부르는데 매우 중요한 역활을 합니다. 다음의 예는 `section`이 사이트 URL과 어떤 상관이 있는지 암시합니다. 만들어진 사이트의 URL경로는 거울을 보듯이 컨텐츠 소스의 경로을 반영합니다.
``` ascii
.
|- content
   |- post
   |  |- firstpost.md   // <- http://1.com/post/firstpost/
   |  |- happy
   |  |  |- ness.md     // <- http://1.com/post/happy/ness/
   |  |- secondpost.md  // <- http://1.com/post/secondpost/
   |- quote
      |- first.md       // <- http://1.com/quote/first/
      |- second.md      // <- http://1.com/quote/second/

```
그렇다면 컨텐트가 소스의 경로와 다른 URL 경로를 가질 수는 없는 걸까요? 예를 들어,

 * 파일 이름 보다 좀 더 의미있는 단어가 URL에 나타나게 할 수는 없는가?
 * `section`을 다른 이름으로 대체할 수는 없는가?
 * 다른 `section`에 속한 컨텐트를 서로 조합해서 일관된 URL로 나타나게 할 수는 없는가?

이런 질문들에 대한 답을 얻기 위해서 다음의 개념들을 이해할 필요가 있습니다.

# 컨텐트 경로 (Destination)

이미 살펴본 바와 같이 특별한 변수가 없다면 Hugo를 통해 생성된 컨텐트의 경로는 소스파일의 경로에 의해 결정됩니다. 하지만 컨텐트의 경로는 앞으로 살펴볼 정면 변수들(Front Matter)을 통해 다양한 형태로 조정될 수 있습니다. 그럼 Hugo는 컨텐트의 경로를 어떻게 조립하는 것일까요? 우선 몇가지 경로의 부분을 지칭하는 이름을 소개하겠습니다.

``` ascii
            permalink
⊢--------------^-------------⊣
http://spf13.com/projects/hugo

    baseURL       section  slug
⊢-----^--------⊣ ⊢--^---⊣ ⊢-^⊣
http://spf13.com/projects/hugo

    baseURL       section          slug
⊢-----^--------⊣ ⊢--^--⊣        ⊢--^--⊣
http://spf13.com/extras/indexes/example

    baseURL            path       slug
⊢-----^--------⊣ ⊢------^-----⊣ ⊢--^--⊣
http://spf13.com/extras/indexes/example

    baseURL            url
⊢-----^--------⊣ ⊢-----^-----⊣
http://spf13.com/projects/hugo

    baseURL               url
⊢-----^--------⊣ ⊢--------^-----------⊣
http://spf13.com/extras/indexes/example
```
* **section** 컨텐트 타입의 기본값을 결정합니다.
  * 컨텐트 소스의 위치에 따라 값이 정해집니다.
  * `url` 정면변수의 값은 `section` 부분경로를 바꿀 수 있습니다.
* **slug** 확장자를 제외한 컨텐트 소스의 파일 이름으로 정해집니다.
  * `slug` 정면변수의 값을 통해 바꿀 수 있습니다.
* **path** `section`에서 시작하여 `slug`직전까지의 경로
  * 컨텐트 소스의 경로에 의해 결정됩니다.
* **url** basicURL 다음부터 `slug`까지 포함된 상대적인 URL
  * 정면변수에 의해 결정될 수 있으며 컨텐트 경로를 결정하는 다른 정면변수의 영향을 무력화 합니다.

`slug`나 `url` 정면변수들을 통해 컨텐트의 목적지 경로(Destination)를 부분적으로 수정하거나 전면적으로 교체할 수 있다는 걸 알 수 있습니다. 이제 목적지 경로 변경 기능외에 컨텐트 처리와 HTML변환시 정면 변수들이 어떤 역활을 하는지 알아봅시다.

# 정면 변수들(Front Matter)
`Front Matter`는 컨텐트의 메타 데이터라고 할 수 있습니다. 컨텐트보다 먼저 나타난다는 의미로 `front matter`라는 이름을 지었을 것으로 추측해 봅니다. 시작과 끝을 나타내는 문자열에 따라 여러가지 포맷이 지원됩니다.

* `+++`로 시작과 끝이 표시되면 [TOML](https://github.com/toml-lang/toml)을 사용해 `Front Matter`를 정의할 수 있습니다.
* `---`로 시작과 끝이 표시되면 [YAML](http://yaml.org)을 사용해 `Front Matter`를 정의할 수 있습니다.
* `{`로 시작하고 `}`로 끝이 표시되면 [JSON](http://www.json.org)을 사용해 `Front Matter`를 정의할 수 있습니다.

이 글에서는 TOML의 예만을 살려보도록 하겠습니다.
```
+++
date = "2016-08-23T23:25:04-04:00"
draft = true
title = "시리즈 #2 - 컨텐츠 제작 기초"
description = "Hugo 입문 두번째 시리즈로 컨텐츠 제작과 관련해 꼭 알아야 할 개념들을 소개합니다"

tags = ["Blog", "Hugo"]
categories = ["How-to"]
series = ["Hugo 입문하기"]
authors = ["Jhonghee Park"]

toc = true
+++
```
`Front Matter`로 정의될 수 있는 변수에 특별한 제약사항은 없습니다. 어떤 변수라도 템플렛안에서 `.Params.varname`형식으로 접근할 수 있습니다. 템플릿 안에서 변수이름은 항상 소문자로 표현됩니다. 예를 들어 `camelCase = true`라고 정의된 변수는 템플릿안에서는 `.Params.camelcase`<sup>2</sup>로 값을 출력할 수 있습니다. 다음은 컨텐트 제작에 필수적인 변수들입니다.

* **title** 컨텐트의 제목
* **description** 컨텐트에 대한 설명
* **date** 컨텐츠를 정열할 때 사용할 날짜
* **taxonomies** 항상 복수형으로 표현되는 분류변수로 위의 예제에 나와있는 `tags`, `categories`, `series`, 그리고 `authors`

이외에 다음과 같은 선택적으로 사용할 수 있는 변수들도 있습니다.

* **aliases** 하나 이상의 이름들이 나열된 정렬(예를 들면 이름을 바꾼 컨텐트가 과거에 사용했던 URL)로 현재의 컨텐트 URL로 리디랙트 되는 별명들. 자세한 내용은 다음 링크를 참조, [Aliases](https://gohugo.io/extras/aliases/).
* **draft** 만약 값이 true이면, 컨텐트는 HTML로 만들어 지지 않습니다. 하지만 --buildDrafts 플래크를 써서 강제할 수 있습니다.
* **publishdate** 만약 날짜가 미래로 잡혀 있으면, 컨텐트는 HTML로 변환되지 않습니다. --buildFuture 플래그를 써서 강제할 수 있습니다.
* **type** 컨텐트 타입 (없는 경우는 컨텐트가 속한 디렉토리를 통해 값이 정해집니다. 즉, `type`의 기본값은 `section`을 통해 정해집니다.)
* **isCJKLanguage** 값이 true인 경우, 컨텐트를 한중일 언어로 작성된 것으로 간주하고, `.Summary`와 `WordCount`와 같은 값들이 한중일 언어에 맞게 생성됩니다.
* **weight** 컨텐트의 차례를 정렬하는데 사용됩니다.
* **markup** (실험적변수) `rst`는 reStructuredText (rst2html 툴을 사용합니다) or `md` (기본값) 은 Markdown
* **slug** URL의 말단에 위치하는 토큰(token)
* **url** 웹 루트에서 컨텐트까지의 전체 경로.

사이트 개발자의 입장에서 사이트의 기능을 확장하고 컨텐트 제작자에게 그 기능을 조종할 수 있는 인터페이스를 제공하려고 할 때 정면 변수를 유용하게 사용할 수 있습니다. 위에 정면변수 예를 보면, 렌더링된 컨텐트의 상단에 목차를 구현하고 그 기능을 컨텐트 제작시 `toc = true`를 이용해 나타나게 하는 예가 있습니다. 또 다른 예는 `authors`를 분류변수(taxonomies)로 등록하고 테마를 통해 구현한 뒤 정면변수의 하나로 컨텐트 제작자에게 자신의 이름을 입력할 수 있게 합니다. 컨텐츠 제작자는 항상 동일한 이름을 사용함으로서 사이트가 제공하는 컨텐츠 목록의 자동 발생을 사용할 수 있습니다.

# 지원되는 컨텐트의 포맷

컨텐트 상단에 정면변수들의 정의되고 그 뒤로 컨텐트의 내용이 따라옵니다. Hugo의 컨텐트는 다양한 포맷으로 제작될 수 있습니다. [asciidoc](http://www.methods.co.nz/asciidoc/), [reStructuredText](http://docutils.sourceforge.net/rst.html)는 외부 프로그램의 도움을 얻어 지원되는 포맷들입니다. 외부 툴에 의존하지 않고 Hugo 자체적으로 컨텐트를 제작할 수 있는 포맷은 [Markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)입니다. Markdown을 이용해 새로운 컨텐트를 제작하고 싶으면 `hugo new`명령에 `md`확장자를 가지는 파일이름을 사용해 시작할 수 있습니다.

```
$ hugo new post/my-first-blog.md
```
이 명령이 실행되면 Markdown 파일이 만들어 지고 기본적인 정면변수들의 셑업이 이루어 집니다. 이 Scaffolding 과정에서 Hugo는 post라는 archetype을 찾기 위해 테마의 archetypes 폴더나 프로젝트 내 archetypes 폴더안을 검색하고 정면변수들의 기본 셑업을 진행합니다. 만약에 post archetype이 정의되어 있지 않을 때는 Hugo의 기본 값들을 사용합니다.

# 컨텐트 타입과 전형 (Content Types and archetypes)
컨텐트 타입은 기본적으로 소스의 위치가 어디에 있느냐에 따라 결정 됩니다. `post/my-first-blog.md`에 작성된 컨텐트는 정면변수 `type`이 존재하지 않는 한 post 컨텐트 타입으로 간주되어 타입에 맞는 렌더링이 이루어 집니다. 만약에 전혀 새로운 컨텐트 타입을 도입하고자 하면 어떻게 해야 할까요? 예를 들어 musician이라는 컨텐트 타입을 통해 유명한 음악가들의 소개를 하고자 하는 가정을 합시다. `hugo new musician/bach.md`로 컨텐트를 초기화 했을때 Hugo의 입장에서는 musician이라는 컨텐트 타입으로 bach.md의 HTML을 렌더링하려고 할 것입니다. musician 컨텐트에 특화된 렌더링을 제공하려면 다음과 같이 새로운 탬플릿을 layouts에 추가해야 합니다.

* 우선 컨텐트 자체의 렌더링을 위해 `layouts/musician/single.html`을 추가합니다.
* `section` 리스트 페이지의 렌더링을 지원하기 위해서는 `layouts/section/musician.html`을 추가합니다.
* 음악가의 시대에 따라 조금씩 다른 페이지 뷰(View)를 제공하려면 `layouts/musician`안에 변형된 템플릿을 추가하고 컨텐트의 정면변수로 `layout`을 사용해 지정해 줍니다.

템플릿을 준비하면서 musician 컨텐트 타입에 필요한 새로운 정면변수들이 생길 수 있습니다. 저자의 입장에서는 새로운 musician 컨텐트를 `hugo new`명령으로 발생 시킨 후 첨가된 정면변수들을 일일이 **기억**해서 입력해야 하는 불편함이 생깁니다. 이런 불편함을 해소하기 위해 musician 컨텐트 타입의 전형(archetype)을 정의해 줄 필요가 있습니다. musician archetype은 `archetypes/musician.md`를 사용해 정의되고 이 archetype 문서에 필요한 기본값들을 지정해 줄 수 있습니다.

**archetypes/musician.md**
```
+++
name = ""
bio = ""
genre = ""
+++
```
이 archetype을 사용해 새 musician 컨텐트를 만들어 봅시다.
```
$ hugo new musician/mozart.md
```
Hugo는 musician 타입을 인지하고 준비된 archetype을 이용하여 정면변수들을 자동으로 입력해 줍니다.
**content/musician/mozart.md**
```
+++
title = "mozart"
date = "2015-08-24T13:04:37+02:00"
name = ""
bio = ""
genre = ""
+++
```

컨텐츠 저자의 관점에서 보면 이제 어느 정도 Hugo를 이용해 정적사이트를 건설할 준비가 끝난 셈입니다. 이어지는 시리즈에서는 사이트 개발자의 관점에서 어떻게 새로운 테마를 만들 수 있는지, 정면변수들과 사이트 구성변수(configuration)들이 템플릿안에서 어떻게 접근할 수 있는지를 알아보겠습니다.

<br/>

<hr/>
<br/>
  <ol>
    <li>Hugo의 configuration은 특별한 조치가 없는 경우 프로젝트 폴더내 config.toml에 정의됩니다. TOML외 YAML과 JSON 포맷이 지원됩니다.</li>
    <li>Hugo는 여러 템플릿 엔진을 지원합니다. 이 글에서는 Go언어의 자체적 `text/template`을 사용하는 것을 전제합니다.</li>
  </ol>
<br/>
<br/>
