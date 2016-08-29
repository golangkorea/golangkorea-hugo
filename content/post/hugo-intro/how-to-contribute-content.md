+++
date = "2016-08-23T23:25:55-04:00"
draft = false
title = "시리즈 #5 - 사이트에 블로그 올리는 방법"

tags = ["Blog", "Hugo"]
categories = ["How to"]
series = ["Hugo 입문하기"]
authors = ["Jhonghee Park"]

+++

# 사이트에 블로그 올리는 방법

[Golang Korean Community 사이트](https://golangkorea.github.io)는 깃헙의 [golangkorea](https://github.com/golangkorea) Organization의 웹사이트로 [GitHub Pages](https://pages.github.com/)를 이용해 제작되고 있습니다. 현존하는 Static Site Generator중 가장 빠른 Hugo를 엔진으로 사용하고 주로 Go언어에 관련된 포스트와 글로벌 기술 동향및 최신의 개발 기법등을 소개하는 포스트를 다루고 있습니다.

# 참여자격

깃헙에 계정이 있는 개발자라면 누구나 제작에 참여하실 수 있습니다.

# Fork it!
[Golang Korean Community 사이트](https://golangkorea.github.io)는 다음과 같이 두개의 Repo를 가지고 개발됩니다.

* **[golangkorea-hugo](https://github.com/golangkorea/golangkorea-hugo)** Hugo로 제작하는 golangkorea.github.io의 소스 프로젝트입니다.
* **[golangkorea.github.io](https://github.com/golangkorea/golangkorea.github.io)** golangkorea-hugo의 submodule로 Hugo로 빌드된 웹사이트입니다. 직접 이 repo에서 작업하지는 않습니다.

포스트를 하기 위해서 우선 githubkorea-hugo를 fork하신다음 본인의 repo를 clone하십시요.

```
$ git clone https://github.com/myaccount/golangkorea-hugo.git
```

# clone hugo-octopress
미래에는 어떻게 바뀔지 모르겠지만 현재 golangkorea.github.io는 [hugo-octopress](https://github.com/parsiya/Hugo-Octopress) 테마를 사용하고 있습니다. `themes` 폴더에 clone하십시요.

```
$ cd themes
$ git clone https://github.com/parsiya/Hugo-Octopress.git
$ cd ..
```

# Start Hugo
이제 로컬에서 사이트를 열어볼 차례입니다. 다음 명령을 사용해서 Hugo의 웹서버를 시작하십시요.
```
$ hugo server
```
[http://localhost:1313](http://localhost:1313)를 브라우저에 열어서 사이트가 뜨는 걸 확인하십시요.

# 첫번째 포스트
`hugo new`명령을 사용해서 포스트의 작성을 시작하십시요.

```
$ hugo new post/my-frist-blog.md
```
golangkorea.github.io는 `authors` taxonomy를 지원합니다. 포스트의 Front Matter에 다름과 같이 저자의 영어 이름을 입력해 주십시요.

```
+++
date = "2016-08-28T23:01:25-04:00"
draft = true
title = "my first post"

authors = ["Your Name"]
+++
```
golangkorea.github.io는 저자의 소개 페이지를 지원합니다. 다름과 같이 본인의 이름을 hyphenated, lower-cased된 형태로 만들어 주십시요.

```
$ hugo new author/your-name.md
```

# Pull Request하기
포스트의 작성이 끝나면 다음 과정을 거쳐 Pull Request해 주십시요.

```
$ git add -A
...
$ git commit -m"My first post"
...
$ git push
```
Pull Request에 대한 자세한 내용은 [GitHub의 도움말](https://help.github.com/articles/about-pull-requests/)을 참조 하세요.

# 최신의 golankorea-hugo repo와 싱크하기

포스트를 한 지 좀 시간이 지나다 보면 그 사이에 사이트에 많은 변화가 있을 수 있습니다. 그때는 본인의 로컬 repo를 최신의 golangkorea-hugo와 싱크 시킬 필요가 생깁니다. 새로운 포스트를 시작하기 전에 우선 로컬의 repo에 golangkorea-hugo를 upstream remote repo로 만드시고 나머지 단계를 따라 싱크 시키십시요.

```
# Add the remote, call it "upstream":

git remote add upstream https://github.com/golangkorea/golangkorea.git

# Fetch all the branches of that remote into remote-tracking branches,
# such as upstream/master:

git fetch upstream

# Make sure that you're on your master branch:

git checkout master

# Rewrite your master branch so that any commits of yours that
# aren't already in upstream/master are replayed on top of that
# other branch:

git rebase upstream/master
```

일단 싱크되면 본인의 forked repo에 다시 푸쉬하십시요.
```
git push -f origin master
```

이제 새로운 포스트를 작성하십시요.

<br/>
<br/>
