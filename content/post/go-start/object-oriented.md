+++
authors = ["Sangbae Yun"]
date = "2016-10-02T00:51:06+09:00"
draft = false
categories = ["How-to"]
tags = ["Object Oriented","struct"]
series=["Go 시작하기"]
title = "object oriented"
toc = false
+++

## 객체지향 프로그래밍
Go는 클래스(Class)와 객체(Object) 이 아닌 타입(types)과 값(values)를 가지고 있다. Struct가 Class 비슷한 역할을 하지만 함수도 Struct로부터 분리되는 이상한 구조를 가지고 있다. 다중상속은[[FootNote(파이슨이나 루비 같은 언어도 다중 상속을 제공하지는 않는다.)]] 말 할 필요도 없고, 단일 상속도 없다. 뭔가 굉장히 객체지향 스럽지 않은 언어로 보일 수 있겠지만 '''충분히 객체지향적'''이다. 그냥 좀 다른 방법으로 객체를 지향하고 있을 따름이다.
  * 네임스페이스(namespacing)는 exports로 대신한다.  
  * struct가 클래스를 대신한다. 다른 OOP에서의 클래스와는 달리 non-virtual 메서드로만 구성할 수 있다.
  * receiver로 메서드를 만들 수 있다.
  * 인터페이스(interfaces)로 다형성을 구현할 수 있다. 다른 OOP에서는 필드 없이, virtual 메서드로만 구성된 클래스 형태로 구현된다.
  * embedding으로 상속을 대신한다. 객체지향의 composition 모델과 비슷하다.
Go 언어를 이용한 객체지향 프로그래밍 기술에 대해서 살펴보자.

## struct와 메서드
  * struct와 리시버(receiver)를 이용한 메서드 구현 
  * 포인터 리비서에 대해서
  * Public과 Private
  * 생성자

## Composition 모델 
  * Composition 모델에 대한 일반적인 설명
  * composition 예제 

## Interface
  * interface를 이용한 다형성
  * empty interface

## 참고
  * [golang oop](https://github.com/luciotato/golang-notes/blob/master/OOP.md)
