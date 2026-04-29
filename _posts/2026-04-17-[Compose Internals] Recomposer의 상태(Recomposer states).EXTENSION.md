---
title: "[Compose Internals] Recomposer의 상태(Recomposer states)"
date: 2026-04-17 13:30:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose runtime, recomposer]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞선 내용들은 책을 이미 읽었다는 가정하에 설명한다.  

`Recomposer`는 내부적으로 자신의 동작 상태를 나타내기 위해 다음과 같은 enum을 가진다.
```kotlin
enum class State {
    ShutDown,
    ShuttingDown,
    Inactive,
    InactivePendingWork,
    Idle,
    PendingWork
}
```
여기서 **enum의 선언 순서 자체가 의미를 가진다는 점**에 주목할 필요가 있다.  
ordinal 값이 작을수록 "더 종료에 가까운" 상태이고, 클수록 "더 활성화된" 상태를 의미한다.  
실제로 내부 코드에서는 `_state.value <= State.ShuttingDown` 같은 비교를 사용해 "이미 종료 절차가 시작되었는가?"를 판단한다.

각 상태의 의미는 다음과 같다.

## **1. ShutDown**
`Recomposer`가 취소되고 모든 정리 작업이 완료된 최종 종료 상태이다.   
이 상태에 들어가면 `Recomposer`는 더 이상 사용할 수 없다.  

---

## **2. ShuttingDown**
`Recomposer`가 취소되었지만 **아직 정리 작업이 진행 중인 상태**이다.   
이미 종료 절차에 들어갔기 때문에 이 상태에서도 `Recomposer`는 더 이상 사용할 수 없다.  

이 상태는 `effectJob`이 완료될 때 설정된다.

### **ShutDown vs ShuttingDown**
`ShutDown`, `ShuttingDown` 상태는 `effectJob`의 완료 시점에 결정된다.
```kotlin
private val effectJob = Job(effectCoroutineContext[Job]).apply {
    invokeOnCompletion { throwable ->
        val cancellation = CancellationException("Recomposer effect job completed", throwable)

        var continuationToResume: CancellableContinuation<Unit>? = null
        synchronized(stateLock) {
            val runnerJob = runnerJob
            if (runnerJob != null) {
                _state.value = State.ShuttingDown
                if (!isClosed) {
                    runnerJob.cancel(cancellation)
                } else if (workContinuation != null) {
                    continuationToResume = workContinuation
                }
                workContinuation = null
                runnerJob.invokeOnCompletion { runnerJobCause ->
                    synchronized(stateLock) {
                        closeCause = throwable?.apply {
                            runnerJobCause
                                ?.takeIf { it !is CancellationException }
                                ?.let { addSuppressed(it) }
                        }
                        _state.value = State.ShutDown
                    }
                }
            } else {
                closeCause = cancellation
                _state.value = State.ShutDown
            }
        }
        continuationToResume?.resume(Unit)
    }
}
```
`effectJob`이 완료되면 `invokeOnCompletion` 블록이 실행된다.  
여기서 분기는 크게 두 갈래다.  
1. **`runnerJob`이 존재하는 경우 (recomposition 루프가 돌고 있던 경우)**
    - 상태를 `ShuttingDown`으로 변경한다.
    - 이후 동작은 `isClosed` 여부에 따라 달라진다.
        - `Recomposer.close()`로 정상 종료된 경우(`isClosed == true`),    
        `runnerJob`은 `runRecomposeAndApplyChanges`의 내부 루프에서 자연스럽게 빠져나와 종료된다.  
        이때 `awaitWorkAvailable()`에 걸려 있던 `workContinuation`이 남아 있다면  
        resume 시켜줘야 루프가 풀리므로, 그 참조를 `continuationToResume`에 담아둔다.
    - 마지막으로 `runnerJob.invokeOnCompletion`을 등록해, **`runnerJob`이 실제로 종료된 뒤**에 상태를 `ShutDown`으로 전이시킨다.
2. **`runnerJob`이 존재하지 않는 경우 (한 번도 실행되지 않았거나 이미 끝난 경우)**
    - 별도의 정리 과정 없이 곧바로 `ShutDown`으로 전이된다.  

즉 정리하면,   
- **`ShuttingDown`** : recomposition 루프를 종료시키는 절차가 시작되었고, `runnerJob`이 실제로 끝나기를 기다리는 상태  
- **`ShutDown`** : `runnerJob`까지 모두 종료되고 `closeCause`가 기록된, 완전한 종료 상태

---

## **3. Inactive**
`Recomposer`가 recomposition을 수행하지 않는 비활성 상태이다.  
이 상태에서는 다음 동작이 일어나지 않는다.  
- composition invalidation 처리
- recomposition 트리거

`Recomposer`가 실제로 동작을 시작하려면 `runRecomposeAndApplyChanges()`가 호출되어야 하며,  
**이 함수가 호출되기 전의 초기 상태**이기도 하다.  
또한 recomposition 도중 처리되지 못한 예외가 발생해 `errorState`에 저장된 경우에도 `Inactive`로 빠진다.

---

## **4. InactviePendingWork**
`Recomposer`가 비활성 상태이지만, **프레임을 기다리는 awaiter가 이미 존재하는 상태**이다.  
구체적으로는 `BroadcastFrameClock`에 걸린 awaiter가 있을 때 해당된다.  
`withFrameNanos` 등을 통해 프레임을 기다리는 코루틴이 이에 해당한다.  

이 경우, `Recomposer`가 실행되면 즉시 프레임을 생성해 대기 중이던 작업을 처리할 수 있다.

---

## **5. Idle**
`Recomposer`가 실행 중이면서, composition/snapshot invalidation을 추적하고 있지만 **현재 처리할 작업이 없는 상태**이다. recomposition 루프는 계속 살아 있고, 다음 작업이 들어오기를 기다린다.

---

## **6. PendingWork**
`Recomposer`가 **처리해야 할 작업이 존재하는 상태**이다.  
이미 작업을 수행 중이거나, 곧 수행할 기회를 기다리는 상태라고 볼 수 있다.  

### **deriveStateLocked()**
`Inactive`, `InactivePendingWork`, `Idle`, `PendingWork` 상태는 `deriveStateLocked()`에서 결정된다.
```kotlin
private fun deriveStateLocked(): CancellableContinuation<Unit>? {
    if (_state.value <= State.ShuttingDown) {
        clearKnownCompositionsLocked()
        snapshotInvalidations = MutableScatterSet()
        compositionInvalidations.clear()
        compositionsAwaitingApply.clear()
        compositionValuesAwaitingInsert.clear()
        failedCompositions = null
        workContinuation?.cancel()
        workContinuation = null
        errorState = null
        return null
    }

    val newState = when {
        errorState != null -> State.Inactive
        runnerJob == null -> {
            snapshotInvalidations = MutableScatterSet()
            compositionInvalidations.clear()
            if (hasBroadcastFrameClockAwaitersLocked) State.InactivePendingWork
            else State.Inactive
        }
        compositionInvalidations.isNotEmpty() ||
            snapshotInvalidations.isNotEmpty() ||
            compositionsAwaitingApply.isNotEmpty() ||
            compositionValuesAwaitingInsert.isNotEmpty() ||
            concurrentCompositionsOutstanding > 0 ||
            hasBroadcastFrameClockAwaitersLocked -> State.PendingWork
        else -> State.Idle
    }

    _state.value = newState
    return if (newState == State.PendingWork) {
        workContinuation.also { workContinuation = null }
    } else null
}
```
함수의 맨 앞에서 `_state.value <= State.ShuttingDown` 조건을 먼저 체크한다는 점이 중요하다.  
**이미 종료 절차에 들어간 상태에서는 `deriveStateLocked()`가 상태를 바꾸지 않고,**     
단지 내부 자료구조를 정리한 뒤 return한다.  
즉, 한 번 `ShuttingDown`으로 넘어가면 다시 `Idle`이나 `Inactive`로 돌아오지 않는다.

그 외의 경우 상태는 다음 규칙에 따라 결정된다.
- **`Inactive`** : `errorState`가 있거나, `runnerJob`이 아직 존재하지 않으면서   (=`runRecomposeAndApplyChanges()`가 실행되기 전) frame clock awaiter도 없는 경우
- **`InactivePendingWork`** : `runnerJob`은 없지만 frame clock awaiter가 이미 존재하는 경우
- **`PendingWork`** : 다음 중 하나라도 만족
    - snapshot invalidation 존재
    - composition invalidation 존재
    - `applyChanges` 대기 작업 존재
    - composition insert 대기 작업 존재
    - concurrent composition 진행 중
    - frame clock awaiter 존재
- **`Idle`** : 위 조건을 모두 만족하지 않는 경우 (처리할 작업 없음)

## **정리**
`Recomposer`의 전체 생명주기는 다음과 같이 그릴 수 있다.
![Recomposer state machine](/assets/img/20260417_1.svg){: w="700" h="700" }    

정리하자면, `Recomposer`는 실행되는 동안에는 **작업 유무에 따라 `Idle`과 `PendingWork` 사이를 오가며** 동작하다가,  
`effectJob`이 완료되는 순간 **`ShuttingDown → ShutDown` 순서로 완전히 종료된다.**   
실행 전에는 frame clock awaiter 유무에 따라 `Inactive` 또는 `InactivePendingWork` 상태에 머무른다.
