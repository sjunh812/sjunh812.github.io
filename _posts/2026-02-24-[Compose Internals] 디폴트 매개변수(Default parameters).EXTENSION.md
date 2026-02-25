---
title: "[Compose Internals] 디폴트 매개변수(Default parameters)"
date: 2026-02-25 13:00:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose compiler]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞 절의 내용은, 책을 이미 읽었다는 가정하에 소개할 예정이다.

Composable 함수는 컴파일 타임에 `$default`라는 메타데이터가 매개변수로 추가된다.  
이번 글에서는 이 디폴트 매개변수에 대해 알아보고자 한다.  

**Kotlin의 디폴트 매개변수 기능은 Composable에서는 사용할 수 없다.**    
Composable에서도 디폴트 매개변수를 선언할 수 있기 때문에 동일한 기능으로 생각할 수도 있다.  
하지만, Composable 함수는 매개변수에 대한 기본 표현식을 **함수의 범위(생성된 그룹)** 내에서 실행해야 하기 때문에,  
두 디폴트 매개변수 기능은 분리된다.  
> **그룹(Group)?**    
> 그룹과 관련해서 책에서는 이전에 언급한 바있다. 뒤에서 더 자세하게 다루겠지만 간단하게 설명하자면,  
> Composable 함수의 호출을 식별하고, 재실행, 스킵, 그리고 상태 복원을 가능하게 하기 위해,  
> **Restartable(재시작 가능한)**, **Replaceable(교체 가능한)**, **Movable(이동 가능한)** 그룹으로 분류할 수 있다.  
{: .prompt-info}  

아래는 Kotlin의 디폴트 매개변수를 사용하는 예시이다.  
```kotlin
fun foo(a: Int = bar()) {
    println(a)
}

fun main() {
    foo()
}
```
`foo()`를 호출한 호출자 `main()`에서 디폴트 매개변수 `bar()`는 미리 계산된다.  
즉, 일반 함수의 디폴트 매개변수의 표현식은 **함수 본문 안에서 실행되지 않고, 호출 시점에서 실행**된다.  
이에 반해 Composable 함수는 아래와 같이 내부적으로 그룹을 생성하기 때문에,  
디폴트 매개변수는 **그룹 내부에 선언되어, 다음 recomposition의 생략(skip) 여부나 상태 추적에 영향**을 준다.  
```kotlin
composer.startGroup(...)
// 디폴트 매개변수는 여기서 실행
composer.endGroup(...)
```

만약 아래와 같이 `remember` 람다식을 Composable 함수의 디폴트 매개변수로 선언하는 경우는 어떨까?  
```kotlin
@Composable
fun Foo(text: String = remember { "hi" }) {
    Text(text)
}
```
`remember` 람다는 **Composable의 그룹 안에서만 호출이 가능**하기 때문에,  
일반함수에서 디폴트 매개변수를 계산하는 방식으로는 호출할 수 없다.  

만약 호출자도 Composable 함수라면 상관없는게 아닐까?  
하지만, **호출자 Composable과 피호출자 Composable의 Context가 서로 다르기 때문에**,  
앞서 말한대로 피호출자 Composable의 그룹 안에서만 호출이 가능하다.

Composable 함수에서는 각 매개변수의 인덱스를 **비트마스킹 기법**으로 매핑하는 `$default` 매개변수를 통해    
디폴트 매개변수를 나타낸다. 이는 앞 절에서 살펴본 `$changed` 매개변수와 동일한 방식이다.  
`$default` 매개변수는 Composable 함수의 각 입력 매개변수에 대해 호출자 쪽에서 제공하는 값을 사용할지,  
또는 디폴트 값을 사용할지에 대해 결정한다.  
> **`$changed`?**   
> `$changed`는 **Composable의 입력 파라미터가 이전 recomposition 대비 변경되었는지**    
> 비트마스크로 표현한 값이다. recomposition을 생략(skip)하는 목적으로 사용되며,  
> 비트마스킹에는 변경 여부 및 안정성에 대한 정보를 저장한다.  
> 아래와 같이 조건을 만족한다면, 전체 함수의 실행을 건너뛰게 된다.  
> ```kotlin  
> if (($changed & 0b1011) == 0b0010 && $composer.skipping) {
>     $composer.skipToGroupEnd()
>     return
> }
> ```
> Composable 함수가 호출자에 의해 전달된 `$changed` 매개변수를 전달받는 것과 마찬가지로,  
> 모든 Composable 함수는 **트리 아래로 전달된 모든 매개변수에 대한 정보를 전달**해야 할 책임이 있다.   
> 이것을 책에서는 **“비교 전파(comparison propagation)“**라고 소개하고 있다.  
{: .prompt-info}

아래 예시는 Compose 라이브러리 문서에 등장하는데,  
`$default` 매개변수가 주입되기 전후의 Composable 함수의 형태와 디폴트 매개변수 값을 알고 사용하는 사례를 보여준다.  
```kotlin
// Before compiler (sources)
@Composable 
fun A(x: Int = 0) {
    f(x)    
}

// After compiler
@Composable(x: Int, $changed: Int, $default: Int) {
    // ...
    val x = if ($default and 0b1 != 0) 0 else x
    f(x)
    // ...
}
```
`$changed` 매개변수의 사례와 마찬가지로 비트의 조작이 수행된다.  
디폴트 값에 대한 비교는 단순하게 `$default` 매개변수를 사용한 비트마스킹을 통해 확인하고,  
`x`에 전달된 값을 디폴트 값인 `0`으로 설정하거나, 원래 값으로 유지한다.