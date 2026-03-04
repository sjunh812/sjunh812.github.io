---
title: "[Compose Internals] 변경 목록(The list of changes)"
date: 2026-03-02 01:00:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose runtime]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞선 내용들은 책을 이미 읽었다는 가정하에 설명한다.   

앞서 책에서 슬롯 테이블에 대해 알아보았다.  
> **슬롯 테이블(Slot Table)?**   
> 슬롯 테이블은 **런타임이 composition의 현재 상태를 저장하는데 사용하는 최적화된 인메모리 구조**이다.  
> 최초의 composition이 발생하는 동안 데이터로 채워지고, recomposition이 발생할 때마다 업데이트된다.  
> 이를 소스의 위치, 매개변수, 기억된 값, `CompositionLocal` 등을 포함하여  
> 모든 Composable 함수의 호출을 추적하는 것으로 볼 수 있다.  
{: .prompt-info}  

그렇다면, Compose Runtime 환경에서 **변경 목록**은 무엇을 의미하는걸까?  

composition(또는 recomposition)이 발생할 때마다 Composable 함수들이 실행되고, **방출(emit)**된다.   
여기서 방출이란, 슬롯 테이블을 업데이트하고 구체화된 트리를 만들기 위해  
궁극적으로 **지연 중인 변경 사항**을 생성하는 과정을 의미한다.  

이러한 변경 사항들은 목록의 형태로 저장된다.  
새로운 변경 목록은 슬롯 테이블에 이미 저장된 값을 바탕으로 생성되며,  
**트리의 어떤 변경이든 composition의 현재 상태에 의존**해야 한다는 점이 중요하다.  

> 가령, 노드를 이동하는 것을 떠올려보자.  
> 일련의 Composable 함수 호출 순서를 재정렬한다고 가정하면,  
> 슬롯 테이블에서 해당 노드가 이전에 어디에 배치되어 있었는지를 먼저 확인해야 한다.  
> 이후 해당 위치에 작성된 모든 슬롯을 제거한 뒤, 새로운 위치에서 다시 슬롯을 작성해야 한다.  
> → **즉, 변경 사항은 composition의 현재 상태에 의존해야 한다.**  
{: .prompt-tip}  

다시 말해, Composable 함수가 발행될 때마다 슬롯 테이블을 확인하고,  
현재 사용 가능한 정보에 따라 지연 중인 변경 사항을 생성하고, 해당 변경 사항을 모두 변경 목록에 추가한다.   
나중에 composition이 끝나면 변경 목록에 기록된 내용들이 실제로 실행되면서 구체화되는데,  
그때가 슬롯 테이블을 composition의 가장 최신 정보로 실제 업데이트하는 순간이다.  

**이처럼 실행을 기다리는 작업을 미리 생성해 두는 것만으로도 방출 과정은 매우 빠르게 동작할 수 있다.**  
composition이 끝나고 실제로 반영되는 즉, **UI를 그리는 작업을 즉시 수행하지 않고,**    
**실행을 기다리는 변경 사항을 목록으로 관리**하여 빠르게 방출할 수 있음을 말한다.  

매 순간마다 UI를 즉시 그린다면, `invalidation`이 계속해서 발생할 것이고,    
recomposition 시, 특정 값에 의해 기존 UI의 수정이 발생하거나, 또는 취소된다면 불필요한 작업을 수행하게 될 것이다.  

변경 목록이 적용된 이후에는 구체화된 노드 트리를 업데이트하기 위해 `Applier`에게 해당 사실을 알린다.  
`Applier`가 하는 역할이 무엇일까?  

## **Applier**
아래는 Applier 인터페이스이다.  
```kotlin
@JvmDefaultWithCompatibility
interface Applier<N> {
    val current: N

    fun onBeginChanges() {}
    
    fun onEndChanges() {}

    fun down(node: N)

    fun up()

    fun insertTopDown(index: Int, instance: N)

    fun insertBottomUp(index: Int, instance: N)

    fun remove(index: Int, count: Int)

    fun move(from: Int, to: Int, count: Int)

    fun clear()
}
```
각 함수의 역할을 간단히 정리하면 아래와 같다.   
- `current`: 현재 변경 사항이 적용되고 있는 노드
- `onBeginChanges()`: 변경 사항의 적용 시작을 알린다.
- `onEndChanges()`: 변경 사항의 적용 종료를 알린다.
- `down()`: 현재 노드에서 자식 노드로 내려간다.
- `up()`: 현재 노드에서 부모 노드로 올라간다.
- `insertTopDown()`: 새로 생성할 인스턴스 생성 전, 현재 노드의 자식으로 삽입한다.(부모 → 자식 순서)   
**부모가 트리에 들어갈 때 ‘자식에게 통지’가 발생하는 구조**에서 효율적이다.  
(부모가 이미 트리에 있으므로, 자식 추가 시 불필요한 전체 재통지가 없다.)
```
1           2           3
R           R           R
|           |           |
B           B           B
             /           / \
            A           A   C
```
- `insertBottomUp()`: 새로 생성할 인스턴스 생성 후, 현재 노드의 자식으로 삽입한다.(자식 → 부모 순서)  
**자식이 트리에 들어갈 때 ‘부모/조상에게 통지’가 발생하는 구조**에서 효율적이다.  
(부모가 아직 트리에 없으므로, 자식 추가 시 조상 통지 폭증을 방지한다.)
```
1           2           3
B           B           R
|          / \          |
A         A   C         B
                         / \
                        A   C
```
- `remove()`: 현재 노드의 자식에서 일정 범위 만큼 노드를 제거한다.
- `move()`: 현재 노드의 자식 일부의 위치를 변경한다.
- `clear()`: 루트로 이동 후 모든 노드를 제거한다.(새 composition 준비 상태로 초기화)  

한편, `Recomposer`는 어떤 스레드에서 composition 하거나, recomposition 할지,  
그리고 변경 목록에 있는 변경 사항을 적용하기 위해 어떤 스레드를 사용할지 결정한다.  
**변경 사항을 적용하기 위한 스레드는 `LaunchedEffect`가 사이드 이펙트를 실행하기 위해 사용하는 디폴트 컨텍스트가 되기도 한다.**  

## **변경 사항 적용 스레드 vs LaunchedEffect 실행 스레드**
아래는 `LaunchedEffect` 코드이다.  
```kotlin
@Composable
@NonRestartableComposable
@OptIn(InternalComposeApi::class)
fun LaunchedEffect(
    key1: Any?,
    key2: Any?,
    block: suspend CoroutineScope.() -> Unit
) {
    val applyContext = currentComposer.applyCoroutineContext
    remember(key1, key2) { LaunchedEffectImpl(applyContext, block) }
}
```
`currentComposer`의 `applyCoroutineContext`를 `LaunchedEffect`의 컨텍스트로 사용하는 것을 볼 수 있다.  
`applyCoroutineContext`는 정확히 어떤 값을 가리키는걸까?  
```kotlin
sealed interface Composer {

    // ...
    /**
     * A Compose internal function. DO NOT call directly.
     *
     * The coroutine context for the composition. This is used, for example, to implement
     * [LaunchedEffect]. This context is managed by the [Recomposer].
     */
    @InternalComposeApi
    val applyCoroutineContext: CoroutineContext
        @TestOnly
        get
    // ...
}
    
internal class ComposerImpl : Composer { 	
		
    // ...
    /**
     * Parent of this composition; a [Recomposer] for root-level compositions.
     */
    private val parentContext: CompositionContext,

    // ...	
    override val applyCoroutineContext: CoroutineContext
        @TestOnly 
        get() = parentContext.effectCoroutineContext
    // ...
}
```
`ComposerImpl`에서 `applyCoroutineContext`는 `parentContext`의 `effectCoroutineContext`를 그대로 반환한다.  
여기서 `parentContext`는 해당 composition의 부모 실행 환경이며,  
루트 composition의 경우, 위 주석 그대로 `Recomposer`가 된다.  
즉, **`LaunchedEffect`가 사용하는 기본 CoroutineContext는 궁극적으로 `Recomposer`가 제공하는 `effectCoroutineContext`에 의해 결정된다.**  

`Recomposer`의 `effectCoroutineContext`는 다음과 같이 구성된다.
```kotlin
class Recomposer(
    effectCoroutineContext: CoroutineContext
) : CompositionContext() {
		
    // ...
    override val effectCoroutineContext: CoroutineContext =
        effectCoroutineContext + broadcastFrameClock + effectJob
    // ...

    suspend fun runRecomposeAndApplyChanges() = recompositionRunner { parentFrameClock ->
        // ...			
    }
    // ...
}
```
`LaunchedEffect`는 `Recomposer`가 구성 및 관리하는 effect 전용 `CoroutineContext`(dispatcher + broadcastFrameClock + effectJob)에서 실행된다.  
또한 `Recomposer`는 `runRecomposeAndApplyChanges()`를 통해 recomposition과 변경 사항을 적용하며,  
이 또한 `Recomposer` 내 동일한 실행 환경 아래에서 동작하기 때문에,  
결과적으로 변경 사항을 적용하는 작업과 `LaunchedEffect`는 같은 실행 환경(Context)을 공유한다.  
(단, 같은 코루틴 블록에서 실행되는 것은 아니고, `LaunchedEffect`는 별도의 하위 코루틴으로 `launch` 된다.)

