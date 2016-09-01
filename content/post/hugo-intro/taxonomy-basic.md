+++
date = "2016-08-23T23:25:44-04:00"
draft = false
title = "시리즈 #4 - 분류(Taxonomy)기능 사용하기"

tags = ["Blog", "Hugo"]
categories = ["How to"]
series = ["Hugo Introduction"]
authors = ["Jhonghee Park"]

toc = true
+++

# 분류(Taxonomy)기능 사용하기

사이트에 컨텐트가 많아 질 수록 고민이 생깁니다. 비숫한 주제의 컨텐트를 한 곳에 나열해 주는 페이지를 만들수는 없는가? 순진한 저자는 자기가 포스트한 내용을 모두 한 포스트에 링크할 지도 모릅니다. 주제 별로 나열해 주는 포스트를 만들어 새로운 포스트가 올라올 때 마다 링크를 걸어 줄 수도 있겠죠. 손이 많이 가고 더군다나 다른 저자들의 비슷한 포스트는 포함되지 못하는 일도 허다할 겁니다.

Hugo는 이런 문제를 분류(taxonomy)라는 기능으로 간단히 햬결해 드립니다.<a href="#footnote-1"><sup>1</sup></a> 기본적으로 Hugo에는 `tags`와 `categories`라는 분류변수를 지원합니다. 컨텐트 저자가 할 일은 단지 Front Matter에 명시해 주기만 하면 됩니다.

```
+++
date = "2016-08-23T23:25:44-04:00"
draft = false
title = "시리즈 #4 - 분류(Taxonomy)기능 사용하기"

tags = ["Blog", "Hugo"]
categories = ["How-to"]
series = ["Hugo Introduction"]
authors = ["Jhonghee Park"]

toc = true
+++
```
예를 들어 `How-to` category의 경우 Hugo는 모든 컨텐트를 스캔하면서 해당 컨텐트들을 `/categories/how-to/index.html`로 나열해 줍니다. 다시 말하면 taxonomy기능이 컨텐트내에 사용되면 taxonomy에 지정된 template을 이용해 Hugo는 모든 taxonomy페이지를 자동으로 만들고 사용된 term과 각 term을 사용한 컨텐트를 나열합니다.

# Taxonomy 용어 정리

* **Taxonomy** 같은 카테고리의 컨텐트를 분류하는 방식 (예: tag, category, author 등)
* **Term** taxonomy에 속하는 키워드 (author의 예를 들면 `Jhonghee Park`, `Sangbae Yun`, `Kookheon Kwon`)
* **Value** term이 사용된 컨텐트의 내용

# Taxonomy Organization
Taxonomy의 관점에서 본 Organization은 다음과 같습니다.
``` ascii
author                                     <-- Taxonomy
    Jhonghee Park                          <-- Term
        ReadMe First                       <-- Content
        시리즈 #1 - Hugo 시작하기           <-- Content
        시리즈 #2 - 컨텐츠 제작 기초         <-- Content
        ...
    Sangbae Yun                            <-- Term
        Go언어 시작하기                    <-- Content
        Golang 프로젝트에 TDD 도입하기      <-- Content
        vim-go를 이용한 go 개발 환경 구축   <-- Content
        ...
```

컨텐트의 관점에서 본 Organization은 다음과 같습니다.
``` ascii
시리즈 #1 - Hugo 시작하기      <-- Content
    author                    <-- Taxonomy
        Jhonghee Park         <-- Term
    series                    <-- Taxonomy
        Hugo 입문             <-- Term
Go언어 시작하기                <-- Content
    author                    <-- Taxonomy
        Sangbae Yun           <-- Term
    series                    <-- Taxonomy
        Go 시작하기            <-- Term
```

# 분류변수(taxonomies)의 정의

분류변수는 사용하기 전에 사이트 configuration에 정의되어야 만 합니다. 단수형과 복수형을 사용해 다음과 같이 정의합니다.

```
[taxonomies]
author = "authors"
category = "categories"
tag = "tags"
series = "series"
```

# 컨텐트에 분류변수 지정하기

일단 분류변수가 사이트 레벨에 정의된 후에는 컨텐트 타입과 `section`을 막론하고 어떤 컨텐트에도 사용 가능합니다. Front Matter에 복수형의 taxonomies를 써서 원하는 모든 Term을 나열하면 됩니다.

# Taxonomy 템플릿

Hugo는 일정한 규칙에 따라 Taxonomy 리스트 페이지를 만들어 냅니다. 주어진 Term의 모든 Taxonomy Value 리스트는 다음과 같은 URL 형식을 따릅니다.

```
/{{복수형 Taxonomy 이름}}/{{Term}}/
```

이때 Term은 다음과 같은 변환을 거칩니다

  * **hyphenated** `How to`는 `/categories/how-to`
  * **lower-cased** `Jhonghee Park`는 `/authors/jhonghee-park`
  * **normalized** `Gérard Depardieu`는 `/actors/gerard-depardieu`<a href="#footnote-2"><sup>2</sup></a>

Taxonomy 페이지를 렌더링하기 위해서는 해당 Taxonomy의 템플릿이 필요합니다. 템프릿의 위치는 다음의 규칙을 따릅니다.

```
/layouts/taxonomy/{{단수형 Taxonomy 이름}}
```

Taxonomy, `authors`의 경우, 템플릿은 `/layouts/taxonomy/author.html`에 위치하게 합니다.



<br/>
<hr/>
<br/>
<ol>
  <li><a id="footnote-1"></a>Hugo v0.11이전에는 taxonomies를 indexes로 불렀습니다. 그런 이유로 만들어 진지 오래된 테마의 taxonomies 템플릿들은 `layouts/indexes`에 위치 할 수도 있습니다.</li>
  <li>Special Character를 보존하고자 하면 사이트 Configuraton에 `preserveTaxonomyNames = true`를 지정해야 합니다.</li>
</ol>
<br/>
<br/>
