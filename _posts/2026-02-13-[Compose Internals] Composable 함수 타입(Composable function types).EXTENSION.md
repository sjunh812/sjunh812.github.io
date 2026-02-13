---
title: "[Compose Internals] Composable 함수 타입(Composable function types)"
date: 2026-02-13 13:30:00 +0900
categories: [Compose]
tags: [compose internals, compose, composable]
---
이번 장에서는 Composable 함수 타입에 대해 개념적으로 다룬다.  

`@Composable` 어노테이션은 컴파일 시점에서 함수의 타입을 효과적으로 바꾼다.  
Composable 함수 타입을 아래와 같이 다양한 관점에서 살펴보자.  
1. **호출 컨텍스트**
- Compose Runtime의 고급 기술을 처리하기 위해 필요한 값들이 담겨있는 **Composer** 를 생성한다.
- Composable 간 호출만 허용한다.
2. **함수의 구문(syntax)**
- `@Composable (T) -> A`으로 표현된다.
- `@Compoasble Scope.() -> A`와 같이 특정 Composable로만 정보 범위를 지정할 수 있다.
    ```kotlin
    @Composable
    inline fun Column(
        modifier: Modifier = Modifier,
        verticalArrangement: Arrangement.Vertical = Arrangement.Top,
        horizontalAlignment: Alignment.Horizontal = Alignment.Start,
        content: @Composable ColumnScope.() -> Unit,
    ) {
        val measurePolicy = columnMeasurePolicy(verticalArrangement, horizontalAlignment)
        Layout(
            content = { ColumnScopeInstance.content() },
            measurePolicy = measurePolicy,
            modifier = modifier,
        )
    }
    ```
    위 예시는 `Column` 컴포넌트 구현부이다.  
    `content` 파라미터를 살펴보면, `ColumnScope`로 범위가 지정되어있음을 알 수 있다.  
    굳이 범위를 지정해서 구현한 이유가 있을까?  
    `content`에 선언되는 Composable들은 부모 노드인 `Column`의 성질을 그대로 전파 받는다.  
    예를 들면, 정렬 및 배치 속성이 있는데 `content` 내에서 `Row`의 성질을 사용할 수는 없기 때문에,    
    이를 강제하기 위함으로 해석할 수 있다.
3. **언어적 관점**
- 타입은 컴파일러에 정보를 제공하여 빠른 정적 검증을 수행하고, 때로는 편리한 코드를 생성하며,  
런타임에 의해 활용되는 데이터 사용 방식을 제한 및 정제하기 위해 존재한다.
- 런타임 시, Composable 함수의 유효성을 검사하고 사용하는 방법을 변경한다.

이렇듯 이러한 관점에서 **Composable 함수가 Kotlin 표준 함수와 다른 타입으로 간주**되는 이유이다.

이번 장은 다소 개념론적인 내용이 많아 다소 딱딱했던 것 같다.  
Compose Compiler와 Compose Runtime의 세부 역할이 생략되어있어 더 어렵다고 느꼈을 수 있는데,  
정리하면 아래와 같다.   
- Composable 함수는 일반 함수와 비교했을 때, **구문적 타입**도 다르고, **컴파일러가 해석하는 방법** 자체가 달라지며  
또한 **Compose Runtime 기준**으로 해석될 수 있도록 수정된다.  
- 앞선 주제에서도 언급된 것 처럼 함수 컬러링에 의해 채색된 함수 즉, **개념론적으로 아예 다른 함수**임을 강조한다.

> **함수 컬러링?**  
> 앞선 주제에서 다뤘던 내용으로 개념은 아래와 같다.  
> Google의 Dart 팀, Bob Nystrom이 2015년에 작성한 “What color is your function?"이라는  블로그 포스트에서 소개된 개념이다.   
> Bob은 비동기(async)와 동기(sync)함수가 잘 결합(compose)되지 않는다고 설명했는데, 동기 함수에서는 비동기 함수를 호출할 수 없기 때문이다.  
> 이를 해결하기 위해 일부 라이브러리와 언어에서는 `Promise`와 `async`/`await`를 도입했다.  
> 이는 함수의 결합을 다시 가져오려는 시도였으며, Bob은 이 두 가지 함수 범주를 다른 “함수 색상(function coloring)”으로 묘사했다.
{: .prompt-info}