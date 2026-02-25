---
title: "[Compose Internals] 타입 검사(Type checks)와 선언 검사(Declaration checks)"
date: 2026-02-20 13:00:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose compiler]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞 절의 내용은, 책을 이미 읽었다는 가정하에 소개할 예정이다.  

앞서 책에서 Compose Compiler가 다양한 목적을 위해 일련의 컴파일러 익스텐션을 등록한다고 했다.  
개발자가 코딩하는 동안 문법의 옳고 그름을 안내해주는 **정적 검사기** 형태로 제공되는 것들이 있는데,  
함수 호출, 타입 및 선언에 대한 검사기가 그 예시이다.  
이 중 **타입 검사(Type checks)**와 **선언 검사(Declaration checks)**에 대해 알아보자.  
# 타입 검사(Type checks)
---
`@Composable` 어노테이션을 함수 외 타입에 사용하는 경우가 있다. 그래서 Compose Compiler에는  
타입 추론과 관련된 검사가 있어, `@Composable` 어노테이션이 달린 타입을 예상했지만, 실제로는 어노테이션이  
없는 경우에 대해 오류를 보고할 수 있다.  
아래 예시를 살펴보자.
```kotlin
val foo: () -> Unit = { 
    run {
        run {
            Text("Hello")   
        }
    }
}
```
함수 `foo`의 기대 타입은 `() -> Unit`이지만, 람다 내부에서는 Composable 함수 `Text`를 호출하고 있다.  
**방문자 패턴**을 이용하여 노드를 순회하는데, 최종적으로 `Text` 호출을 방문하면서, 추론 타입은 `@Composable () -> Unit`이 된다.  
> **방문자 패턴(Visitor Pattern)?**  
> 방문자 패턴은 객체 지향 디자인 패턴 중 하나로, 객체 구조 내에서 새로운 기능을 추가할 때 사용되는 패턴이다.  
> 방문자 패턴의 주요 목적은 기존의 객체 구조를 변경하지 않고 새로운 연산을 추가하는 것으로,     
> **데이터 구조 및 처리 기능을 분리**하기 위함이다.  
> 컴파일러 관점에서 노드의 종류는 함수, 람다, 호출, 조건문, 타입 등 다양하며, 적용해야 할 동작 역시  
> 여러 종류의 분석(타입 검사, 선언 검사 등)일 수 있기 때문에 방문자 패턴을 사용한다.  
{: .prompt-info}

# 선언 검사(Declaration checks)
---
호출 위치 및 타입 검사 외에도 element의 선언 위치와 관련해서 프로퍼티, 프로퍼티 접근자, 함수 선언, 함수 매개변수와 같은 것들도 검사되어야 한다.  
호출 및 타입 검사로 모든 처리가 가능할 수도 있다는 생각이 들 수도 있지만, **재정의(override)** 시 발생할 수 있는 **선언 오류**에 대한 검사가 필요하다.  
아래는 잘못된 재정의 선언에 대한 예시이다.  
```kotlin
interface ComposeNode {
    @Composable
    fun node()
}

internal class ComposeNodeImpl : ComposeNode {
    override fun node() {
        Text("node")
    }
}
```
`node` 함수를 오버라이딩하는 과정에서 `@Composable` 어노테이션이 생략되었다. 이러한 경우, 선언 검사를 통해 방지할 수 있다.  
Compose Compiler는 이러한 `KtElements` 중 어느 것이든 오버라이드될 경우,  
`@Composable` 어노테이션이 선언되어 있는지를 확인하여 일관성을 유지하는 검사를 수행한다.  
> **KtElements?**  
> PSI 노드의 구성 요소로, Kotlin 소스코드를 구성하는 모든 문법 요소가 여기에 포함된다.  
> Kotlin 파일, 클래스, 프로퍼티, 프로퍼티 접근자, 함수 호출 등 모든게 포함된다.  
{: .prompt-info}

이외에도 `main` 함수를 Composable 함수로 만들거나, Composable 속성의 backing field도 선언 검사를 통해 금지된다. 
1. **`main` 함수** :  
메인 함수는 JVM 단 프로젝트 시작점으로 생각될 수 있다.  
이러한 환경을 Composable 함수로 변경하게 된다면, Composable 컨텍스트를 제공하지 않기 때문에 사용이 불가하다.  
따라서 Composable 함수는 독립적인 진입점이 될 수 없고, 반드시 composition을 생성하는 런타임 진입 API를 통해 호출되어야 한다.  
아래 예시는 `Activity` 또는 `Fragment`에서 `ComposeView`를 사용할 때 composition을 생성하는 단계의 일부이다.  
    ```kotlin
    // Activity, Fragment에서 ComposeView 사용 시
    // 최초 composition 생성 단계(시작점)
    fun setContent(content: @Composable () -> Unit) {
        shouldCreateCompositionOnAttachedToWindow = true
        this.content.value = content
        if (isAttachedToWindow) {
            createComposition()
        }
    }
    ```
    `Activity`나 `Fragment` 같은 안드로이드 컴포넌트는 `Composer`를 보유하고 있지 않지만,  
    `setContent()`와 같은 API를 통해 최초 composition이 발생한다.     
    이 시점에서 Compose Runtime이 내부적으로 `Composer`를 생성하고, 이후부터 Composable 함수 호출이 가능해진다.
2. **backing field** : 
    > **Backing Field?**   
    > 프로퍼티에 대한 접근을 감시하고, 제어하는 숨겨진 필드로,  
    > 기본적으로 getter와 setter를 가진다. getter와 setter에서는 직접 값을 접근하지 않고,  
    > backing field를 통해 값을 저장하고 처리한다.
    {: .prompt-info}

    backing field에 대한 저장 및 처리는 Compose 바깥 환경에서도 발생할 수 있다.  
    그렇기 때문에 recomposition, skip 등 Compose Runtime에서 안정성 및 일관성을 보장받지 못한다.  
    아래와 같이 초기값을 선언한 변수에 `@Composable` 어노테이션을 붙이면 오류가 발생하는데,  
    이는 **초기값을 생성한 시점에서 backing field가 생성**되기 때문이다.  
    ```kotlin
    // 초기값이 있어 백킹 필드 생성 -> 오류
    @Composable
    var count: Int = 0
    ```
    getter는 접근자 특성상 값을 변경하지 않으며, 필요에 따라 `@Composable` 어노테이션을 사용할 수 있다.  
    만약 getter가 composition이나 snapshot 상태를 변경하지 않고, 오직 읽기 전용으로만 동작함을 보장할 수 있는 경우,  
    `@ReadOnlyComposable` 어노테이션을 함께 사용할 수 있다.
    ```kotlin
    val color: Color 
        @Composable
        @ReadOnlyComposable 
        get() = MaterialTheme.colorScheme.primary
    ```


