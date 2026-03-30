---
title: "[Compose Internals] 노드 연결 및 그리기(Attaching and drawing the nodes)"
date: 2026-03-25 13:30:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose runtime, compose ui, layoutnode]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞선 내용들은 책을 이미 읽었다는 가정하에 설명한다.  

Compose에서 UI는 `LayoutNode`라는 노드 트리로 구성된다.  
흥미로운 점은, 이 노드들이 단순한 데이터 구조가 아니라는 것이다.  

> **노드는 스스로 어떻게 연결되고, 어떻게 그려지는지 알고 있다.**  

`UiApplier`가 노드 삽입을 위임하면, `LayoutNode`는 다음과 같은 과정을 스스로 수행한다.

## **LayoutNode 삽입 과정**
![LayoutNode 삽입 과정](/assets/img/20260327_1.png){: w="700" h="400" }    
### 1. 삽입 가능 여부 확인
- 부모가 이미 있는지
- 트리 구조가 올바른지
즉, 잘못된 트리 구조를 방지하는 단계

### 2. Z Index 정렬 리스트 invalidate
Compose는 자식들을 **Z Index 기준으로 정렬된 별도 리스트**로 관리한다.
- 그리기 순서 보장 (낮은 Z → 높은 Z)
- 하지만 매번 정렬하지 않고 **삽입 시점에는 invalidate만 하고, 실제로 필요할 때 정렬한다.**  
이 방식을 통해 불필요한 비용을 제거하고, 여러 변경을 한번에 처리 가능하다.

### 3. 부모 및 Owner 연결
- 부모 `LayoutNode`에 연결
- 동시에 Owner도 함께 설정  
여기서 **Owner**라는 중요한 개념이 등장한다. (뒤에서 좀 더 자세히 다룬다.)  
이 단계는 "화면에 그릴 수 있는 상태"로 가는 핵심이다.

### 4. invalidate 호출
- 최종적으로 Owner를 통해 invalidate 발생
- 다음 프레임에서 다시 그리도록 요청

---  

## **Owner란?**
Owner는 Compose UI 트리를 Android View 시스템과 연결하는 루트이자 창구이다.  
실제로 `AndroidComposeView`가 이에 해당한다.  

![Compose UI 레이아웃](/assets/img/20260327_2.jpeg){: w="400" h="700" }    

Owner가 하는 일은 아래와 같다.

1. **`LayoutNode`를 "화면에 그릴 수 있는 상태"로 만든다.**   
    `LayoutNode`는 단독으로 화면에 나타날 수 없기 때문에, 반드시 Owner에 attach되어야 한다.  

2. **Android 렌더링 파이프라인과 연결**  
    Owner를 통해 measure, layout, draw 작업이 수행된다.  
    실제 렌더링은 전부 Owner를 통해 이루어진다.
3. **invalidate의 출발점**  
    invalidate를 호출하여, Android 프레임 시스템(Choreographer)에  
    "다음 프레임에 다시 그려라"라고 요청한다.
4. **입력 및 시스템 이벤트 전달**  
    Owner는 렌더링 뿐만 아니라, 터치 이벤트, 키보드 상태, 포커스, 접근성 등  
    모든 플랫폼 이벤트를 Compose로 전달하는 역할도 한다.

---

## **setContent**
Activity, Fragment, 또는 ComposeView에서 호출되는 `setContent()`를 본 적 있을 것이다.  
`setContent()`가 호출되는 시점에 Owner라고 불리는 `AndroidComposeView`가 생성된다.  

아래 코드를 살펴보자.  

```kotlin
internal fun AbstractComposeView.setContent(
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
    GlobalSnapshotManager.ensureStarted()
    val composeView =
        if (childCount > 0) {
            getChildAt(0) as? AndroidComposeView
        } else {
            removeAllViews(); null
        } ?: AndroidComposeView(context, parent.effectCoroutineContext).also {
            addView(it.view, DefaultLayoutParams)
        }
    return doSetContent(composeView, parent, content)
}

private fun doSetContent(
    owner: AndroidComposeView,
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
    if (isDebugInspectorInfoEnabled && owner.getTag(R.id.inspection_slot_table_set) == null) {
        owner.setTag(
            R.id.inspection_slot_table_set,
            Collections.newSetFromMap(WeakHashMap<CompositionData, Boolean>())
        )
    }
    val original = Composition(UiApplier(owner.root), parent)
    val wrapped = owner.view.getTag(R.id.wrapped_composition_tag)
        as? WrappedComposition
        ?: WrappedComposition(owner, original).also {
            owner.view.setTag(R.id.wrapped_composition_tag, it)
        }
    wrapped.setContent(content)

    // When the CoroutineContext between the owner and parent doesn't match, we need to reset it
    // to this new parent's CoroutineContext, because the previous CoroutineContext was cancelled.
    // This usually happens when the owner (AndroidComposeView) wasn't completely torn down during a
    // config change. That expected scenario occurs when the manifest's configChanges includes
    // 'screenLayout' and the user selects a pop-up view for the app.
    if (owner.coroutineContext != parent.effectCoroutineContext) {
        owner.coroutineContext = parent.effectCoroutineContext
    }

    return wrapped
}
```

코드와 같이 Owner 역할을 수행하는 `AndroidComposeView`가 생성되며,    
Composable 실행 결과로 생성된 UI 변경 사항은  
`Composition`을 통해  `UiApplier(owner.root)`로 전달되어 Owner가 관리하는 `LayoutNode` 트리에 반영된다.    
또한 `Composition`을 Android 환경에 맞게 wrapping한 어댑터 `WrappedComposition`를 활용하여  
Android 생명주기 및 환경을 연결한다. (ex. `dispose()` 처리)  

마지막으로 `wrapped.setContent(content)`를 통해 실제 UI가 실행되며, 
- Composable 실행
- `LayoutNode` 생성
- 트리 구성 시작  

이 발생한다.

이외에도 `CoroutineContext`를 동기화하는 코드도 볼 수 있는데,  
이는 `Recomposer`/`LaunchedEffect` 등이 실행되는 Coroutine 환경에 사용된다.

---

## **마무리**
지금까지의 내용을 하나의 프로세스로 간단히 정리하면 아래와 같다.
```
Composer
  ↓
UI 변경 사항 계산
  ↓
Applier
  ↓
LayoutNode 생성 및 트리 구성
  ↓
Owner(AndroidComposeView)에 attach
  ↓
Owner.invalidate()
  ↓
Android 렌더링 시스템 (Choreographer)
```
결론적으로 Owner는 `setContent()` 호출 시 생성되는 `AndroidComposeView`로,  
Compose UI 트리를 Android View 시스템에 연결하여 실제 화면에 그려지도록 만드는 렌더링의 진입점이다.