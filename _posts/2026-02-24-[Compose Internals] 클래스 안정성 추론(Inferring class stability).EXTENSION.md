---
title: "[Compose Internals] 클래스 안정성 추론(Inferring class stability)"
date: 2026-02-24 13:00:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose compiler]
---
Compose를 개발해본 사람이라면, smart recomposition에 대해 들어본 적이 있을 것이다.  
smart recomposition은 Composable의 입력값이 변경되지 않았고,  
해당 입력값이 **안정적(Stable)**으로 간주될 때, Composable의 **recomposition을 생략**하는 것을 
의미한다.  
그러므로 Compose Runtime은 필요에 따라 recomposition을 생략하기 위해,  
해당 입력값을 안전하게 읽고 비교할 필요가 있다.  

안정적인 타입이 충족해야하는 특성은 아래와 같다.   
- 두 인스턴스에 대한 `equals` 함수의 호출은 항상 동일한 두 인스턴스에 대해 동일한 결과를 반환한다.
- 타입의 `public` 프로퍼티가 변경될 때마다, composition은 항상 변경 사항에 대한 알림을 받는다.  
→ 그렇지 않다면, Composable의 입력값과 본문에 반영되는 **최신 상태 사이에 비동기화(불일치)가 발생**할 수 있다.   그래서 항상 recomposition이 수행되며, smart recomposition은 이런 불안정한 입력값에 의존할 수 없기 때문이다.
- 모든 `public` 프로퍼티는 원시 타입 또는 안정적으로 간주되는 타입을 가진다.  

위 특성은 책 내 어디선가 본 특성과 매우 닮아있다.  
바로 `@StableMarker` 어노테이션으로 마킹된 타입의 안정성을 결정짓는 요구사항이다.  

모든 원시타입들은 `String`을 포함하여 모든 함수형과 함께 기본적으로 안정적인 타입으로 간주하는데,  
그 이유는 해당 타입들은 정의상 **불변**으로 간주되기 때문이다.  
불변 타입은 변하지 않기 때문에 composition에게 변화를 알릴 필요가 없다.  
불변 타입이 아님에도 Compose에서 안정적인 타입으로 간주할 수 있는 타입에는 `@Stable` 어노테이션을 추가하여 정의할 수 있다.  

`MutableState`는 값이 변경될 때, Compose에 변경 알림을 전달하여 가변 상태를 **안전하게 추적**할 수 있다.  

만약 개발자가 직접 만든 커스텀 타입이라면,   
위에 나열한 특성을 준수하는지 직접 판단하고 수동으로 `@Immutable` 또는 `@Stable` 어노테이션을 사용하여    
안정적인 타입을 표시할 수 있다.  
하지만 이와 같이 개발자가 직접 판단하고 정의하는 행위는 코드의 유지보수성을 낮출 수 있기 때문에,  
클래스의 안정성은 Compose가 스스로 추론할 수 있도록 하는 것이 바람직하다.  

안정성을 추론하는 알고리즘은 지속적으로 발전하고 있지만,  
기본적으로 모든 클래스를 방문하고, 이에 대해 `@StabilityInferred`라는 주석을 합성하는 방식을 따른다.  
또한 클래스에 대해 안정성 정보를 인코딩하는 아래와 같은 합성값을 추가한다.  
```java
static final int $stable
```
이 합성값은 컴파일러가 런타임에서 클래스 안정성을 결정 짓기 위한 추가적인 기계 장치를 생성하는데 도움이 된다.  
따라서 Compose는 해당 클래스에 의존하는 Composable 함수에 대해 recomposition 실행 여부를 결정할 수 있다.  

> 클래스의 안정성 추론은 모든 클래스마다 적용되는 것이 아니다.  
> 열거형(enum), 열거형 항목(enum entry), 내부 클래스(inner class), 컴패니언 클래스(companion class),  
> 또는 인라인 클래스(inline class)가 아닌 모든 `public` 클래스에 대해 적용된다.    
> 따라서 `@Immutable`과 `@Stable` 어노테이션이 추가되지 않은 일반적인 클래스 및 데이터 클래스(data class)에  
> 대한 추론을 수행한다.  
{: .prompt-info}

---   

클래스의 안정성을 추론하기 위해 Compose는 여러가지 요소를 고려한다.  
만약 클래스의 모든 필드(field)가 읽기 전용이고 안정적일 때,   
해당 타입은 Compose Compiler에 의해 안정적이라고 추론된다.  
(여기서 말하는 필드란, 컴파일 이후 JVM 바이트 코드에 실제로 생성되는 결과물을 의미한다.)    

아래와 같은 제네릭 타입의 매개변수가 포함된 클래스는 어떨까?  
```kotlin
class Foo<T>(val value: T)
```
예시의 제네릭 타입 `T`는 클래스 매개변수 중 하나에 사용되므로,  
`Foo`의 안정성은 `T`에 전달된 타입의 안정성에 의존하게 된다.  
하지만, `T`가 **실제화된 타입이 아니기 때문에 런타임이 실행되기 전까지는 알 수 없다.**   

`T`에 전달된 타입을 알게될 때, 런타임에서 클래스의 안정성을 결정하는 일종의 기계 장치가 필요하다.  
Compose Compiler는 위 경우의 안정성 추론을 돕기 위해, 해당 타입 매개변수의 안정성을 의존해야 함을 나타내는  **`@StabilityInferred` 어노테이션에 비트마스크를 계산**하여 넣는다.  

하지만, 제네릭 형식이 있다고 반드시 불안정한 것은 아니다.  
아래 예시를 살펴보자.  
```kotlin
class Foo<T>(val a: Int, b: T) {

    val c: Int = b.hashCode()
}
```
제네릭 타입으로 선언된 `b`는 프로퍼티가 아닌 단순 파라미터로 사용된다.  
코드 상에서는 `hashCode()`를 실행하며, 이는 **같은 인스턴스에 대해 항상 동일한 결과를 반환**할 것을 기대하기 때문에,  
안정적이라는 사실을 이미 알고 있다.  

```kotlin
class Foo(val bar: Bar, val bazz: Bazz)
```
위와 같은 클래스처럼 다른 클래스로 매개변수를 구성한 경우,  
안정성은 모든 매개변수에 대한 안정성의 조합으로 추론된다.  
이는 `Foo → Bar → Bazz`와 같은 형태처럼 **재귀적**으로 해결된다.  

한편 클래스 내부적으로만 사용되는 가변적 상태 또한 클래스를 불안정하게 만든다.  
아래 예시는 `private`하게 선언된 가변한 매개변수를 가지고 있다.  
```kotlin
class Counter {

    private var count = 0
    
    fun getCount(): Int = count
    fun increment() { count++ }
}
```
`count`는 내부적으로만 값을 변경할 수 있지만, 결국 시간이 지남에 따라 상태는 변할 것이다.  
이는 Compose Runtime이 해당 값을 일관성 측면에서 신뢰할 수 없음을 의미한다.  
즉, **Compose Runtime은 증명할 수 있는 것만 안정적인 타입으로 간주**한다.  
인터페이스의 경우, 어떻게 구현될지 모르기 때문에 불안정하다고 가정하는게 바로 그 예이다.  

`List`는 읽기 전용 컬렉션 인터페이스이다.  
그렇다면, `List`를 매개변수로 받는 클래스는 안정적일까?  
```kotlin
@Composable
fun <T> MyListOfItems(items: List<T>) {
    // ...
}
```
`List`는 가변적인 구현체인 `ArrayList`로 구현될 수 있기 때문에 불안정한 타입으로 간주한다.  

`public` 프로퍼티들이 불가변적으로 사용되는 경우는 어떨까?  
Compose Compiler는 해당 프로퍼티가 불가변성을 가질 가능성에 대해 추론할 수 없기 때문에,  
기본적으로 불안정한 타입으로 간주한다. 하지만 이는 **단점으로 작용**할 수 있는데,  
해당 프로퍼티들이 **불변 타입으로 구현될 가능성이 높고, Compose Runtime이 추론해도 될 가치가 충분히 있기 때문**이다.  
그래서 개발자는 `@Stable` 어노테이션을 마킹하여 안정적임을 컴파일러에게 명시할 수 있다.  

아래 예시를 살펴보자.  
```kotlin
@Stable
interface UiState<T: Result<T>> {
    val value: T?
    val exception: Throwable?
    
    val hasError: Boolean
        get() = exception != null   
}
```
`value`는 제네릭 타입에 의해 조건부 안정성 즉, 불안정성을 가지고 있고, `exception` 프로퍼티는 `Throwable`  
(open class)으로 선언되어 있어 이 또한 불안정하다.  
또한 `UiState` 자체가 인터페이스로 선언되어 있어, 이 자체만으로도 불안정하다.  
하지만, 개발자는 해당 클래스를 불가변적으로 사용할 수 있다.     
이런 경우에는, 개발자의 책임 아래 `@Stable` 어노테이션을 통해 안정적임을 컴파일러에게 알릴 수 있다.  
> 일반적으로 `UiState`는 화면의 상태를 표시하는 값의 개념으로 사용되기 때문에,   
> 특정 이벤트가 발생하기 전까지는 변하지 않는다.  
> 만약 이벤트에 의해 변하는 경우, **새로운 인스턴스를 생성하기 때문에 사용 관점에서 불변**할 수 있다.  
{: .prompt-tip}


