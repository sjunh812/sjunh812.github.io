---
title: "[Compose Internals] @DisallowComposableCalls"
date: 2026-02-15 14:00:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose compiler, disallowComposableCalls, annotation]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞의 내용들은 책을 이미 읽었다는 가정하에 소개할 예정이다.  

Compose 어노테이션 중에는 `@DisallowComposableCalls`라는 어노테이션이 있다.  
이 어노테이션은 **함수 내에서 Composable 함수의 호출이 발생하는 것을 방지**하기 위해 사용된다.  
Compose Runtime에서는 Composable 함수 간 호출로 UI 트리 구조를 생성하는데,   
이와 같이 Composable 호출을 방지하는 어노테이션이 왜 필요한걸까?  

Composable 함수를 안전하게 호출할 수 없는 Composable 함수의 인라인 람다 매개변수에서  
유용하게 사용될 수 있는데, 주로 recomposition마다 호출되면 안되는 람다식에 가장 적합하게 사용된다.  
➡️ **`@Composable` 인라인 함수의 람다는 상위 Composable로 호출 위치가 확장되기 때문이다.**  
```kotlin
@Composable
inline fun <T> remember(crossinline calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(false, calculation)
```
`remember`를 살펴보면, `calculation` 블록에 의해 제공된 값을 기억하게 되는데,  
이는 **최초 composition 단계에서만 수행**되어야 하며,   
이후의 모든 **recomposition 단계에서는 항상 이미 계산된 값을 반환**한다.  
그렇기 때문에 `calculation` 블록은 `@DisallowComposableCalls` 어노테이션으로 선언되어 있다.
> (Compose Internals 내용 중)  
> "`@DisallowComposableCalls` 어노테이션 덕분에 `calculation` 람다식에서 Composable 함수 호출이  
> 금지됩니다. 만약 Composable 함수 호출이 허용된다면, Composable 함수의 노드 방출 시 슬롯 테이블에서 
> 공간을 차지하고 람다가 더 이상 호출되지 않으므로 첫 composition 단계 후에 삭제됩니다."   
{: .prompt-info}
위와 같이 책에서 소개하고 있는데 어떤 내용을 전달하고자 했을까?  
아래 예시를 살펴보자.   
```kotlin
// 실제로 이렇게 구현되어있지 않다.
@Composable
inline fun <T> remember(calculation: () -> T): T =
    currentComposer.cache(false, calculation)
    
@Composable
fun FlowScreen() {
    val colaboSrno = remember {
        Text("최초 composition에만 불리는 친구")   
        100
    }
    Text("제목")
}
```
만약 위와 같이 인라인 함수 `remember`의 `calculation` 블록에서 Composable 함수의 호출을 허용한다면 어떻게 될까?   
아마 최초 composition에서만 `remember` 블록에서 선언한 UI 노드가 생기고,  
recomposition 이후에는 람다는 호출되지 않아 해당 노드는 삭제된다.  

Compose는 호출 순서에 따라 **위치기억법**이 동작하는데,  
recomposition 이후에 첫번째 위치에 있던 `Text`가 삭제되면서 **슬롯 테이블에 불필요한 변경**이 생긴다.   
따라서 **최초 composition 단계부터 UI 노드 생성을 막기 위해 `@DisallowComposableCalls`를 이용**한 것이다.
> 인라인 람다에서는 상위 호출 컨텍스트의 Composable 기능을 상속해야 한다.  
> 예를 들어 `forEach` 함수가 Composable 함수에서 호출되었다면, `forEach`의 람다는 Composable 함수를 호출할 수 있다.  
> 반면, `remember`와 같이 예외적으로 Composable 함수의 호출을 막아야할 때는 바람직하지 않다.   
> `@DisallowComposableCalls`는 이러한 조건부로 호출되는 인라인 람다에서 가장 적합하게 사용된다. 
{: .prompt-tip}
위와 같은 속성 때문에 `@DisallowComposableCalls`로 마킹된 인라인 람다 내부에서 또 다른 인라인 람다를 호출하는 경우,  
컴파일러에서 해당 람다도 `@DisallowComposableCalls`로 표시해야 한다(**전파성**).  

한편, Compose Runtime에 대한 자체적인 클라이언트 라이브러리(Compose UI와 유사한)를 작성해야 한다면,   
Compose Runtime 제약조건을 준수해야 한다(위치기억법, 호출 컨텍스트, 사이드 이펙트 처리 등).
