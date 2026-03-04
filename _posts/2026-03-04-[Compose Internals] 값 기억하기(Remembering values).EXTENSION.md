---
title: "[Compose Internals] 값 기억하기(Remembering values)"
date: 2026-03-04 22:00:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose runtime, rememberobserver]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞선 내용들은 책을 이미 읽었다는 가정하에 설명한다.  

composition으로부터 `remember` 함수가 호출되면,   
이전 composition 대비 값이 변경되었는지에 대한 비교는 즉시 수행된다.   
다만 업데이트 작업은 `Composer`가 삽입 중이 아닌 경우에만 `Change`로 기록된다.  
(삽입 중이라면, 슬롯 테이블 쓰기 및 `Applier`에 즉시 반영한다.)  

만약 업데이트할 값이 `RememberObserver`인 경우,  
`Composer`는 composition의 기억 작업을 추적하기 위해 **암시적** `Change`를 기록한다.  
이는 나중에 기억하고 있는 모든 값을 다시 잊어야할 때 사용하기 위함이다.  
여기서 `RememberObserver`에 대해 다소 생소할 수 있는데,  
`RememberObserver`는 `remember`로 생성된 객체의 **composition 생명주기**를 직접 받을 수 있게 해주는 인터페이스이다.  

한편, 위 단락의 내용 중 암시적 `Change`를 기록하는 이유에 대해    

> *"이는 나중에 기억하고 있는 모든 값을 다시 잊어야할 때 사용하기 위함이다."*   

라고 설명하고 있는데, 이건 무슨 뜻일까?  
간단한 예시를 통해 살펴보자.  

아래 `remember` 함수는 상태가 아닌 단순 값을 반환한다.    
```kotlin
val x = remember { 10 }
```
이런 경우, 생명주기와 상관없이 일관된 값을 가지기 때문에 생명주기를 관리할 필요가 없다.  
하지만 아래와 같다면 어떨까?  
```kotlin
val scope = remember { CoroutineScope(...) }
```
composition이 발생하면 scope의 사용이 시작되고,  
composition에서 제거되면 scope를 `cancel()`해줘야 한다.  
이렇듯 **composition 생명주기**에 맞게 처리가 필요할 때 `RememberObserver`를 사용한다.  

## **RememberObserver**
아래는 실제 `RememberObserver` 인터페이스의 모습으로,  
각 함수의 역할에 대해 알아보자.  
```kotlin
@Suppress("CallbackName")
public interface RememberObserver {
    /**
     * Called when this object is successfully remembered by a composition. This method is called on
     * the composition's **apply thread.**
     */
    public fun onRemembered()

    /**
     * Called when this object is forgotten by a composition. This method is called on the
     * composition's **apply thread.**
     */
    public fun onForgotten()

    /**
     * Called when this object is returned by the callback to `remember` but is not successfully
     * remembered by a composition.
     */
    public fun onAbandoned()
}
```
- `onRemembered()`: 객체가 composition에 정상적으로 들어왔을 때 호출되며,  
apply 단계에 실행되는 스레드를 사용한다.
- `onForgotten()`: 객체가 composition에서 제거될 때 호출되며,  
apply 단계에 실행되는 스레드를 사용한다. (`remember` 블록이 사라지는 시점)
- `onAbandoned()`: composition이 완료되기 전에 취소된 경우 호출된다. (apply 전)  

앞서 `remember` 함수에서 composition 생명주기 관리가 필요한 예시로 `CoroutineScope`를 들었는데,  
**Compose는 `CoroutineScope`를 `RememberObserver`로 구현한 `rememberCoroutineScope`라는 함수를 제공한다.**  
```kotlin
@Composable
public inline fun rememberCoroutineScope(
    crossinline getContext: @DisallowComposableCalls () -> CoroutineContext = {
        EmptyCoroutineContext
    }
): CoroutineScope {
    val composer = currentComposer
    return remember { createCompositionCoroutineScope(getContext(), composer) }
}

@PublishedApi
@OptIn(InternalComposeApi::class)
internal fun createCompositionCoroutineScope(
    coroutineContext: CoroutineContext,
    composer: Composer,
): CoroutineScope =
    if (coroutineContext[Job] != null) {
        CoroutineScope(
            Job().apply {
                completeExceptionally(
                    IllegalArgumentException(
                        "CoroutineContext supplied to " +
                            "rememberCoroutineScope may not include a parent job"
                    )
                )
            }
        )
    } else {
        val applyContext = composer.applyCoroutineContext
        RememberedCoroutineScope(applyContext, coroutineContext)
    }
```
`rememberCoroutineScope`가 `remember` 블록으로 감싸여 있는 모습을 확인할 수 있고,    
이는 내부적으로 `RememberedCoroutineScope` 라는 새로운 `CoroutineScope` 객체를 생성하고 있다.

아래는 그 예시이다.  
```kotlin
internal class RememberedCoroutineScope(
    private val parentContext: CoroutineContext,
    private val overlayContext: CoroutineContext,
) : CoroutineScope, RememberObserver {
    private val lock = makeSynchronizedObject(this)

    @Volatile private var _coroutineContext: CoroutineContext? = null

    override val coroutineContext: CoroutineContext
        get() {
            // ...
        }

    fun cancelIfCreated() {
        synchronized(lock) {
            val context = _coroutineContext
            // ...
        }
    }

    override fun onRemembered() {
        // Do nothing
    }

    override fun onForgotten() {
        cancelIfCreated()
    }

    override fun onAbandoned() {
        cancelIfCreated()
    }

    companion object {
        @JvmField val CancelledCoroutineContext: CoroutineContext = CancelledCoroutineContext()
    }
}
```
`remember` 블록 내에서 composition 생명주기에 맞게 `CoroutineScope`를 관리하기 위해,  
`RememberObserver`를 상속 받아 구현하고 있다.  

결론적으로 위 `CoroutineScope`와 같이 composition 생명주기 관리 대상인 경우,  
당장의 값 업데이트가 발생하지 않았더라도 `Composer`는 이를 추적할 필요가 있다.  
이러한 이유로 `Composer`는 실제 값 변경이 없더라도 `RememberObserver`를 **암시적 Change로 기록해**    
**composition 생명주기를 안정적으로 추적**한다.