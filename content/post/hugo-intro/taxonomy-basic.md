+++
date = "2016-08-23T23:25:44-04:00"
draft = false
title = "시리즈 #4 - 분류(Taxonomy)기능 사용하기"

tags = ["Blog", "Hugo"]
categories = ["How-to"]
series = ["Hugo 입문하기"]
authors = ["Jhonghee Park"]

toc = true
+++

# 분류(Taxonomy)기능 사용하기

사이트에 컨텐트가 많아 질 수록 고민이 생깁니다. 비숫한 주제의 컨텐트를 한 곳에 나열해 주는 페이지를 만들수는 없는가? 순진한 저자는 자기가 포스트한 내용을 모두 한 포스트에 링크할 지도 모릅니다. 주제 별로 나열해 주는 포스트를 만들어 새로운 포스트가 올라올 때 마다 링크를 걸어 줄 수도 있겠죠. 손이 많이 가고 더군다나 다른 저자들의 비슷한 포스트는 포함되지 못하는 일도 허다할 겁니다.

Hugo는 이런 문제를 분류(taxonomy)라는 기능으로 간단히 햬결해 드립니다. 기본적으로 Hugo에는 `tags`와 `categories`라는 분류변수를 지원합니다. 컨텐트 저자가 할 일은 단지 Front Matter에 명시해 주기만 하면 됩니다.

```
+++
date = "2016-08-23T23:25:44-04:00"
draft = false
title = "시리즈 #4 - 분류(Taxonomy)기능 사용하기"

tags = ["Blog", "Hugo"]
categories = ["How-to"]
series = ["Hugo 입문하기"]
authors = ["Jhonghee Park"]

toc = true
+++
```
예를 들어 `How-to` category의 경우 Hugo는 모든 컨텐트를 스캔하면서 해당 컨텐트들을 `/categories/how-to/index.html`로 나열해 줍니다. 
