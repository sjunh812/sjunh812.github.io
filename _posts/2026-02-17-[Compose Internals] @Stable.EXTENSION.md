---
title: "[Compose Internals] @Stable"
date: 2026-02-17 13:00:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose compiler, stable, annotation]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞 절의 내용은, 책을 이미 읽었다는 가정하에 소개할 예정이다.  

Compose에서 안정성(Stability)을 보장한다는 것은 smart recomposition 관점에서 매우 중요하다.  
Compose는 변경이 발생한 지점만 다시 그리기 위해, **변경되지 않은 부분의 recomposition을 건너뛰는(skip) 전략**을   
사용한다. 이때 “이 값은 이전과 동일하다”라는 사실을 컴파일러와 런타임에 명확히 전달해줄 수 있어야 하는데,  
그 근거로 활용되는 개념이 바로 **안정성(Stability)**이다. 즉, 안정성은 불필요한 recomposition을 방지하기 위한  
신뢰 가능한 변경 판단 기준이다.  

이와 같은 안정성을 판별하기 위해 `@Immutable`과 `@Stable` 어노테이션이 각각 사용되는데,  
`@Stable` 어노테이션에 대해 알아보자.  

`@Stable`은 `@Immutable`보다는 조금 가벼운 약속이다.  
**가변적임(mutable)**을 의미하고, `@StableMarker`에 의한 상속의 의미만 지니게 된다.  
(`@StableMarker` 어노테이션의 요구사항에 대해 책에서 먼저 확인하자.)  

`@Stable` 어노테이션을 함수나 프로퍼티에 적용하면, 항상 동일한 입력값에 대해 동일한 결과를 반환하다는 사실을 컴파일러에    
알린다. 이는 함수의 매개변수가 `@Stable` 또는 `@Immutable`으로 마킹되어있거나, 기본 유형(primitive 타입)인 경우에만 가능하다.  

Composable 함수에 매개변수로 전달된 모든 타입이 안정적인 타입으로 마킹되면, **위치기억법을 기반으로   
이전 함수 호출과의 매개변수 값이 동일한지 비교하고, 모든값이 동일하다면 recomposition을 생략**한다.  
아래는 위치기억법에 따라 값이 같더라도 서로 다른 Composable 함수임을 보여주는 예시이다.   
```kotlin
val count = remember { 1 }

if (condition) {
    Text("$count")
else {
    Text("$count")
}

// 두 Text는 항상 같은 값을 가지지만, 위치기억법(호출 위치)를 기반으로 다르다.
```

한편 `@Stable`은 `@Immutable`과 다르게 클래스 외 함수 범위에도 사용 가능하다고 명시되어 있다.  
```kotlin
@MustBeDocumented
@Target(
    AnnotationTarget.CLASS,
    AnnotationTarget.FUNCTION,
    AnnotationTarget.PROPERTY_GETTER,
    AnnotationTarget.PROPERTY
)
@Retention(AnnotationRetention.BINARY)
@StableMarker
annotation class Stable
```
그렇다면 아래와 같이 일반 함수에 대해 `@Stable`를 명시할 수 있는 걸까?  
```kotlin
// 이런식으로 함수에도 @Stable 어노테이션을 사용 가능한걸까?
@Stable
fun foo(a: String): String { ... }
```
recomposition/skippable 관점에서 봤을 때, Composable 호출에서 파라미터(입력값)이 변경의 기준이기 때문에 
의미가 없다. 그렇다면 왜 `AnnotationTarget.FUNCTION`을 추가해두었을까?   

아래는 `ColumnScope` 내 `Modifier`의 확장 함수 `weight`이며, `@Stable`로 선언된 예시이다.    
입력에 따른 결과는 일정한 것으로 보이는데, 왜 `@Stable` 어노테이션으로 굳이 선언한걸까?  
단순히 계약상의 명시를 위해서인걸까?
```kotlin
internal object ColumnScopeInstance : ColumnScope {
    @Stable
    override fun Modifier.weight(weight: Float, fill: Boolean): Modifier {
        require(weight > 0.0) { "invalid weight $weight; must be greater than zero" }
        return this.then(
            LayoutWeightElement(
                // Coerce Float.POSITIVE_INFINITY to Float.MAX_VALUE to avoid errors
                weight = weight.coerceAtMost(Float.MAX_VALUE),
                fill = fill
            )
        )
    }
```
`weight` 함수는 `Modifier` 체인에 사용되는데, 이에 대한 안정성을 유지하기 위함으로 볼 수 있다.  
즉, `@Stable`이 함수에 붙는 경우는 Composable이 추적하는 값(예: `Modifier`)을 생성하는 함수가 안정적인 계약을 가진다는 것을 컴파일러에 명시하기 위해서다.  


`@Stable`을 사용할 수 있는 타입의 예로는 **public 프로퍼티가 변경되지는 않지만 불변의 객체**로 간주될 수 없는 경우이다.  
예를 들어, private한 가변적인 상태를 소유하고 있거나, `MutableState` 객체에 대해서 내부적으로 프로퍼티를 위임  
(property delegation)하고, 외부에서 사용되는 형태는 불가변적(immutable) 상태인 경우이다.  

```kotlin
@Stable
class ScrollState(initial: Int) : ScrollableState {

    /**
     * current scroll position value in pixels
     */
    var value: Int by mutableIntStateOf(initial)
        private set
    ...
}
```
`LazyColumn`에서 많이 사용되는 `ScrollState`가 대표적인 `@Stable` 어노테이션의 사용 예시라 볼 수 있다.  
`value`는 가변적이며, Compose가 변경 여부를 알 수 있는 `MutableIntState`로 선언되어 있고, setter는 private으로 선언되어 있다.  

책에서는 `@Stable` 어노테이션의 의미를 Compose Compiler와 Runtime의 **데이터가 어떻게 진화할지**   
**(또는 진화하지 않을지)**에 대한 가정을 하고, 필요한 경우 숏컷과 같은 형태로 사용하는 경우라고 표현하고 있다.  
여기서 진화의 관점이라는 말을 아래와 같이 해석해보았다.  
1. 진화 가능
    ```kotlin
    data class DetailEventState(
        var isGoogle: Boolean
    )   
    ```
    `var`로 선언되어 있기 때문에, 컴파일러는 해당 값이 언제 바뀔지 예측할 수 없다.  
    그러므로 recomposition마다 항상 확인해야 한다(unstable).
2. 진화 불가능
    ```kotlin
    @Immtuable
    data class DetailEventState(
        val isGoogle: Boolean
    )
    ```
    `@Immutable` 어노테이션이 사용될 수 있는 상태로,  
    인스턴스 생성 후, 절대 변하지 않는 값을 보장하므로 recomposition의 관측 대상이 아니다.
3. 진화 일부 가능
    ```kotlin
    @Stable
    data class DetailEventState(...) {

        var funcCount by mutableIntStateOf(0)
            private set
    }
    ```
    `@Stable` 어노테이션이 사용될 수 있는 상태로, 값이 바뀔 수는 있지만 **Compose가 관측 가능**한 케이스이다.  
    여기서 Compose가 관측 가능한 케이스란 무엇일까?  
    앞서 위에서 언급한 `MutableState`가 이에 해당하는데, Compose에 변경을 알릴 수 있기 때문에,  
    Compose 환경에서 안정적이라고 판단할 수 있다.   
    만약 `MutableList`로 선언한다면, Compose는 해당 값이 변경됐을 때 인지하지 못한다.  
    아래 예시를 살펴보면 좀 더 이해가 쉬울 것이다.
    ```kotlin
    @Composable
    fun test(modifier: Modifier = Modifier) {
        var a by remember { mutableIntStateOf(0) }
        val b = mutableListOf<Int>()

        Column(
            modifier = modifier,
            verticalArrangement = Arrangement.spacedBy(4.dp)
        ) {
            Text("a: $a")
            Text("b: $b")
            Button(
                onClick = {
                    a++
                    b.add(1)
                }
            ) {
                Text("Click")
            }
        }
    }

    // 버튼을 클릭하면, a와 b에 대해 변경을 하고 있지만, 실질적으로 변경 관측이 가능한건
    // mutableIntStateOf로 선언한 a뿐이다.
    ```  

만약 `@Stable` 어노테이션의 의미가 충족될 것이라는 확신이 없다면 이 어노테이션을 절대 사용하면 안된다.   
그렇지 않으면 Compose Compiler에게 잘못된 정보를 제공하게 되어 쉽게 런타임 오류가 발생할 수 있다.  
(`@Immutable` 어노테이션도 동일)
> `@Immutable` 및 `@Stable` 어노테이션이 각자 다른 의미를 지닌 서로 다른 약속이더라도,  
> 오늘날 Compose Compiler는 smart recomposition과 recomposition을 생략하는 기능을 활성화하기 위해  
> **두 어노테이션 모두 동일한 방식으로 취급한다.**   
> Compose Compiler와 Runtime에서 향후 두 어노테이션을 원활하게 활용하기 위해 서로 다른 의미로 적용하고,  
> 미래의 Jetpack Compose 변화에 따른 차별 가능성을 열어두고 있는데,   
> 이것이 `@Immutable` 및 `@Stable` 어노테이션이 각자 따로 존재하는 이유이다.   
> → **즉, 현재는 skippable한 처리를 위해 공통점을 가지고 있지만, 변경 가능성에 대해서는 서로 그 정도가 다르기 때문에 미래의 새로운 최적화 기법에 나눠 사용될 수 있다는 관점으로 해석할 수 있다.**  
{: .prompt-tip}