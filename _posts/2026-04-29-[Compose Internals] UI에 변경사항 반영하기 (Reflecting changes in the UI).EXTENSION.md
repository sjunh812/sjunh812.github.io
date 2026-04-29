---
title: "[Compose Internals] UI에 변경사항 반영하기 (Reflecting changes in the UI)"
date: 2026-04-29 12:30:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose ui]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞선 내용들은 책을 이미 읽었다는 가정하에 설명한다.  

초기 composition과 후속 recomposition의 과정을 통해  
UI 노드가 어떻게 방출되고 런타임에 제공되는지 알아보았다.  

발생한 모든 변경사항을 실제 UI에 반영하여 사용자가 경험할 수 있도록 하는 통합이 필요하다.  
이 과정은 흔히 노드 트리의 **“구체화(materialization)“**라고 불리며,  
이는 Compose UI와 같은 클라이언트 라이브러리의 책임이다.  

이제부터 노트 트리의 구체화 과정에 대해 알아보자.  
(이전에 다뤘던 내용을 포함한다.)

## **1. Composition/Recomposition은 변경 사항을 계산한다.**
Composable 실행 결과로 Compose Runtime은  

- 어떤 노드를 새로 만들어야 하는지
- 어떤 노드를 삭제해야 하는지
- 어떤 속성이 바뀌었는지  

알아낸다.  

예를 들면, `Text("A")`가 `Text("B")`로 바뀔 때 새 화면을 통째로 다시 만드는 것이 아닌,  
기존 노드는 유지하고 텍스트만 B로 바꿔야 한다는 변경사항을 기록한다.  

## **2. 변경 사항을 Applier가 실제 노드 트리에 적용한다.**
`Applier`는 `Composer`가 계산한 결과를 받아서 실제 트리에 대해  

- 노드 삽입/이동/제거
- 속성 업데이트

를 수행한다.  

즉, Compose Runtime이
```
“여기 Box 하나 넣어”
“이 Text 내용 바꿔”
“이 노드는 제거해”
```
라고 한다면,  
`Applier`는 그걸 실제 UI 트리에 반영하는 역할을 한다.  

## **3. Compose UI에서는 LayoutNode가 실제 트리이다.**
안드로이드 Compose UI 쪽에서는 보통 변경사항들이 `LayoutNode`에 적용된다.  

예를 들면,
- `Box` 추가 → `LayoutNode` 삽입
- `Modifier` 변경 → 해당 노드 속성 갱신
- `Text` 변경 → 텍스트 관련 노드/측정/그리기 정보 갱신

으로 이어진다.  
이렇듯 추상적인 Composable의 결과가 실제 UI 노드 구조로 바뀐다.  

## **4. 측정/배치/그리기가 발생한다.**
노드 트리가 변경됐다면, 실제 화면에 그리기 위한 

- 측정(measure)
- 배치(layout)
- 그리기(draw)

과정을 거쳐 실제 픽셀이 그려진다.  

---

> Compose에서 `Modifier`는 미리 고정된 속성이 아니라,  
> Composable 실행 과정에서 계산되는 값이기 때문에 recomposition이 발생한다.  
> 예를 들어 아래와 같은 `Modifier`는  
> ```kotlin
> Modifier
>     .padding(8.dp)
>     .background(color)
>     .clickable { onClick() }
> ```
> 겉보기에는 하나의 값처럼 보이지만, 실제로는  
>
> - **레이아웃**에 영향을 주는 요소
> - **그리기**에 영향을 주는 요소
> - **입력 처리**에 영향을 주는 요소
>
> 들이 순서대로 연결된 선언적 체인이다.  
> 
> 따라서 이 중 하나의 값이라도 변경되면,  
> Compose는 먼저 recomposition을 통해 새로운 `Modifier` 체인을 다시 계산해야 한다.  
> 그리고 그 결과를 바탕으로 Compose UI가 실제 내부 노드 구조에 반영한다.  
> 만약 `drawBehind`나 `graphicsLayer`를 활용하면, Composition 및 Layout 단계를 생략할 수 있다.  
> → **smart recomposition**
{: .prompt-info} 