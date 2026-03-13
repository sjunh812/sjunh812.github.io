---
title: "[Compose Internals] 노드 탐색(Navigating the nodes)"
date: 2026-03-13 13:30:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose runtime, composerchangelistwriter]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞선 내용들은 책을 이미 읽었다는 가정하에 설명한다.  

노드 트리의 탐색은 실제로 `Applier` 에 의해 수행되지만,   
**탐색 명령은 즉시 실행되지는 않는다.**   

Compose Runtime은 노드 트리를 이동할 때, `Applier.down()`과 `Applier.up()`을 바로 호출하지 않고,  
**탐색 경로를 먼저 기록**한 뒤 필요할 때만 실제 탐색을 수행한다.  
이 과정에서 하향 이동 경로는 `downNodes` 라는 스택(배열)에 저장된다.  

---

## **하향 탐색 이전에 상향 탐색이 발생하는 경우**

만약 실제 하향 노드 탐색이 수행되기 전에 상향 탐색이 발생한다면,  
Compose는 아직 실행되지 않은 하향 이동을 취소할 수 있다.  

이 경우, 단순히 **`downNodes` 스택에서 마지막 노드를 제거**하여 탐색 경로를 단축한다.  

아래와 같은 탐색이 수행된다고 가정해보자.  

```kotlin
applier.down(A)
applier.down(B)
applier.up()
```

이를 즉시 실행한다면 실제 탐색 경로는 다음과 같다.

```
Root → A → B → A
```

하지만 최종적으로는 `A` 위치에 머물게 되므로,  
`B` 노드를 실제로 탐색할 필요는 없다.    

이처럼 Compose Runtime은 이러한 불필요한 이동을 방지하기 위해,  
`Applier`의 호출을 지연시키고 탐색 경로만 먼저 기록하는데,  
앞서 언급한 `downNodes` 스택에서 제거하는 것만으로 탐색을 최적화할 수 있다.  

```
1. down(A) → [A]
2. down(B) → [A, B]
3. up()    → [A]   // B 제거
```

결과적으로 실제로 수행되는 이동은 다음과 같이 최소화된다.

```kotlin
applier.down(A)
```

## **ComposerChangeListWriter**
`ComposerChangeListWriter`는 Composition 과정에서 발생한 노드 변경과 탐색 정보를 ChangeList에 기록하는 역할을 한다.  

즉, Composition 단계에서 발생한 노드 이동(down, up)과 노드 변경(insert, remove 등)을  
바로 `Applier`에 전달하지 않고 변경 목록(ChangeList)으로 수집하는 중간 계층이다.  
이 과정에서 노드 탐색 역시 즉시 수행되지 않고, **지연된 탐색 형태로 기록된다.**  

아래는 `ComposerChangeListWriter` 코드 예시다.  
```kotlin
internal class ComposerChangeListWriter(
    private val composer: ComposerImpl,
    var changeList: ChangeList
) {
    // ..
    // Navigation of the node tree is performed by recording all the locations of the nodes as
    // they are traversed by the reader and recording them in the downNodes array. When the node
    // navigation is realized all the downs in the down nodes is played to the applier.
    //
    // If an up is recorded before the corresponding down is realized then it is simply removed
    // from the downNodes stack.
    private var pendingUps = 0
    private var pendingDownNodes = Stack<Any?>()
    // ..
}
```
앞서 설명했던 것처럼 `pendingDownNodes`는 아직 **실행되지 않은 하향 탐색 경로**를 저장하고,  
`pendingUps`는 아직 **실행되지 않은 상향 탐색 횟수**를 기록한다.  

하향 탐색이 기록될 때는 `moveDown()`이 호출된다.  

```kotlin
    fun moveDown(node: Any?) {
        realizeNodeMovementOperations()
        pendingDownNodes.push(node)
    }
```
이때 `Applier.down()`을 바로 호출하지 않고, 단순히 `pendingDownNodes` 스택에 노드를 기록한다.

반대로 상향 탐색이 발생하면 `moveUp()`이 호출된다.

```kotlin
    fun moveUp() {
        realizeNodeMovementOperations()
        if (pendingDownNodes.isNotEmpty()) {
            pendingDownNodes.pop()
        } else {
            pendingUps++
        }
    }
```

여기서도 앞서 살펴본 것처럼 아직 실행되지 않은 down이 있다면 단순히 스택에서 제거하여 탐색을 취소한다.  
이를 통해 `down → down → up`과 같은 불필요한 탐색을 실제 `Applier` 호출 없이 제거할 수 있다.  

이렇게 기록된 탐색 정보는 이후 `pushPendingUpsAndDowns()`에서 ChangeList로 전달된다.
```kotlin
    private fun pushPendingUpsAndDowns() {
        if (pendingUps > 0) {
            changeList.pushUps(pendingUps)
            pendingUps = 0
        }

        if (pendingDownNodes.isNotEmpty()) {
            changeList.pushDowns(pendingDownNodes.toArray())
            pendingDownNodes.clear()
        }
    }
```
결국 이 단계에서 지연되어 있던 탐색 명령들이 ChangeList에 기록되고,  
이후 `applyChanges()` 단계에서 실제 `Applier` 호출로 변환된다.  

---

결론적으로 Compose Runtime은 노드 탐색을 즉시 수행하지 않고 먼저 기록한 뒤,  
변경을 적용하는 시점에 최소한의 이동만 실제로 수행하도록 최적화한다.  
