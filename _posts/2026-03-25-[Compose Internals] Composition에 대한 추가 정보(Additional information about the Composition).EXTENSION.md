---
title: "[Compose Internals] Composition에 대한 추가 정보(Additional information about the Composition)"
date: 2026-03-25 13:30:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose runtime, composition]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞선 내용들은 책을 이미 읽었다는 가정하에 설명한다.  

## **Composition에서 invalidation은 어떻게 처리될까?**
`Composition`은 recomposition을 위해 **보류 중인 invalidation**을 관리한다.  
또한 `isComposing` 플래그를 통해 현재 composition이 진행 중인지 여부도 알고 있다.  
덕분에 invalidation이 발생했을 때, 그 변경을 지금 진행 중인 composition 흐름 안에서 즉시 반영할지,  
아니면 pending 상태로 보류해 두었다가 이후 recomposition에서 처리할지를 판단할 수 있다.

> 여기서 말하는 즉시 반영은 invalidation이 발생한 순간 recomposition 전체를 다시 수행한다는 뜻이 아니다.  
> 정확히는, 현재 composition이 진행 중이라면 해당 invalidation 정보를 내부 상태에 기록하여   
> 이번 또는 다음 recomposition 흐름에서 놓치지 않도록 반영한다는 의미에 가깝다.  
> 반대로 composition이 진행 중이 아니라면 invalidation은 pending 상태로 보관되었다가,  
> 이후 적절한 시점에 recomposition이 수행될 때 함께 처리된다.
{: .prompt-tip}

또한 `Recomposer`는 특정 composition이 이미 수행 중인지 파악할 수 있다.  
따라서 composition 도중 발생한 추가 invalidation을 매번 새로운 recomposition으로 즉시 중첩 실행하지 않고,  
현재 작업이 끝난 뒤 이어지는 recomposition 흐름에서 처리할 수 있도록 조율한다.

즉, composition 중 발생한 invalidation은 새로운 recomposition을 즉시 재귀적으로 시작하는 방식이 아니라,  
현재 composition과 이후 recomposition 사이에서 안전하게 병합되고 조정되는 방식으로 다뤄진다.

---

## **Composition은 어떻게 실행될까?**
런타임은 일반적인 `Composition`보다 더 많은 제어 기능을 제공하는 `ControlledComposition`에 의존한다.

`ControlledComposition`은 `Recomposer`가 composition에 대해  
**invalidation을 반영하고, recomposition을 수행하며, 변경사항을 실제로 적용할 수 있도록 만든 인터페이스**이다.  
대표적인 예가 `composeContent`와 `recompose`이다.

- `composeContent` :  
초기 composition을 시작할 때 사용된다. 주어진 content를 이 composition에 연결하고,  
이를 기반으로 composition을 구성하는 진입점이라고 볼 수 있다.
- `recompose` :  
이미 존재하는 composition에 대해 invalidated scope를 기준으로 필요한 부분만 다시 계산하는 작업이다.  
State 변경 등으로 특정 scope가 invalid 되면, 해당 범위를 다시 계산하고 이후 변경사항을 반영하는 흐름으로 이어진다.

아래는 `Recomposer` 내부의 `composeInitial()` 함수이다.

```kotlin
internal override fun composeInitial(
    composition: ControlledComposition,
    content: @Composable () -> Unit
) {
    val composerWasComposing = composition.isComposing
    try {
        composing(composition, null) {
            composition.composeContent(content)
        }
    } catch (e: Exception) {
        processCompositionError(e, composition, recoverable = true)
        return
    }

    if (!composerWasComposing) {
        Snapshot.notifyObjectsInitialized()
    }

    // ...

    try {
        performInitialMovableContentInserts(composition)
    } catch (e: Exception) {
        processCompositionError(e, composition, recoverable = true)
        return
    }

    try {
        composition.applyChanges()
        composition.applyLateChanges()
    } catch (e: Exception) {
        processCompositionError(e)
        return
    }

    if (!composerWasComposing) {
        // Ensure that any state objects created during applyChanges are seen as changed
        // if modified after this call.
        Snapshot.notifyObjectsInitialized()
    }
}
```

`composition.composeContent(content)`가 호출되면,  
`Recomposer`는 해당 composition에 content를 연결하고 슬롯 테이블과 그룹 구조를 구성하면서 필요한 변경사항을 기록한다.  
이 단계는 실제 UI 변경을 즉시 반영하는 단계라기보다,  
**composition 결과를 계산하고 이후 적용할 변경사항을 수집**하는 단계에 가깝다.

이후 `Snapshot.notifyObjectsInitialized()`가 호출된다.  
이 시점까지 생성된 state 객체를 이후부터는 **snapshot의 변경 추적 대상에 포함시키는 역할**로 이해할 수 있다.

그다음 `performInitialMovableContentInserts(composition)`를 통해  
movable content와 관련된 초기 삽입 작업을 수행한 뒤,  
`composition.applyChanges()`와 `composition.applyLateChanges()`를 호출해  
앞서 기록한 변경사항을 실제 composition 상태와 `Applier` 쪽에 반영한다.

마지막으로 다시 `Snapshot.notifyObjectsInitialized()`를 호출하는데,  
이는 `applyChanges()` 과정에서 새롭게 생성된 state 객체들 역시  
이후 변경이 발생했을 때 올바르게 추적될 수 있도록 하기 위함이다.

정리하면 `composeInitial()`은 다음과 같은 흐름을 담당한다.

```
초기 composition 수행 → movable content 초기 처리 → 변경사항 적용 
→ 새롭게 생성된 state 객체가 이후 변경 추적 대상에 포함되도록 처리
```

또한 composition 수행 중 예외가 발생하면 현재 composition pass는 중단될 수 있다.  
이는 단순히 에러가 발생했다는 의미를 넘어,  
`Composer`가 해당 pass 동안 유지하던 **임시 위치 정보, 스택, 참조 상태 등을 정리하고 현재 작업을 중단**한다는 뜻에 가깝다.

---

## **Composition은 언제 recomposition을 skip할 수 있을까?**
`Composition`은 자신이 의존하는 외부 문맥과 값들을 추적할 수 있으며,  
그 값이 변경되면 해당 범위를 다시 recomposition 대상으로 만들 수 있다.  
대표적인 예가 `CompositionLocal`이다.

상위 composition에서 제공하는 `CompositionLocal` 값이 변경되면,  
그 값을 읽고 있던 하위 composition 역시 다시 실행되어야 한다.  
이때 각 composition은 `CompositionContext`를 통해 연결되어 있으므로,  
상위 문맥의 변화가 하위 composition으로 전파될 수 있다.

예를 들어 다음과 같은 코드가 있다고 하자.

```kotlin
CompositionLocalProvider(LocalTheme provides theme) {
    Child()
}
```

만약 `Child()`가 `LocalTheme.current`를 읽고 있었다면,  
상위에서 제공하는 theme 값이 바뀌는 순간 `Child()`는 다시 실행되어야 한다.  
즉, 입력 파라미터가 그대로이더라도 **provider를 통해 전달되는 외부 문맥이 변경되었다면 skip할 수 없다.**

반대로 현재 scope가 참조하는 외부 문맥에 변화가 없고,  
해당 `RecomposeScope` 자체도 invalid 상태가 아니며,  
현재가 재사용이나 강제 재실행과 같은 특수 경로가 아니라면  
`Composer`는 해당 recomposition을 skip할 수 있다.  

여기서 흔히 말하는 **invalid provider**는  
`CompositionLocal` provider의 값이 바뀌어  
해당 값을 읽는 하위 scope를 다시 실행해야 하는 상황을 의미한다.

이렇듯 recomposition의 skip 여부는 단순히 **파라미터 비교만으로 결정되지 않는다.**  
`Composer`는 다음 요소들을 함께 고려한다.  

- 현재 `RecomposeScope`가 invalid 되었는지
- `CompositionLocal`과 같은 외부 문맥에 변경이 있었는지
- 현재 실행이 재사용 또는 강제 재실행과 같은 특수 경로에 놓여 있는지

결국 skip은  
“이 scope를 다시 실행해야 할 명확한 이유가 있는가?”를 기준으로 판단되며,  
파라미터 변경 여부는 그 판단 요소 중 하나일 뿐이다.

---

## **마무리**
정리하자면, `Composition`은 단순히 Composable 함수를 다시 실행하는 객체가 아니다.  
현재 composition이 진행 중인지, 어떤 invalidation이 보류 중인지,  
그리고 어떤 변경이 실제로 적용되어야 하는지를 함께 관리하면서  
**recomposition이 안전하고 효율적으로 수행되도록 조율하는 역할**을 맡는다.

또한 recomposition이 항상 전체를 다시 실행하는 것도 아니다.  
필요한 범위만 다시 계산하고,  
외부 문맥이나 현재 scope 상태에 따라 어떤 부분은 skip할 수도 있다.  

이러한 구조 덕분에 Compose Runtime은  
상태 변경을 단순히 “다시 그린다”는 수준이 아니라,  
**어디를 다시 계산해야 하고 어디는 재사용할 수 있는지 판단하면서 composition을 수행**할 수 있다.  
