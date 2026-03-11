---
title: "[Compose Internals] 현재 상태 스냅샷에 접근(Accessing the current State snapshot)"
date: 2026-03-10 20:30:00 +0900
categories: [Compose]
tags: [compose internals, compose, compose runtime, snapshot]
---
> Compose Internals 책을 읽고 발표한 내용을 정리한 글이다.  
> 앞선 내용들은 책을 이미 읽었다는 가정하에 설명한다.  

`Composer`는 현재 composition 실행 시점의 상태에 대한 **스냅샷**을 참조한다.  
이 스냅샷은 현재 스레드에서 관찰 가능한 상태 값들의 **일관된 시점**을 제공한다.  
즉, Compose는 단순히 최신 값을 읽는 것이 아니라 **특정 시점의 상태 집합을 기준으로 UI를 계산**한다.  

만약 상태 값을 항상 "최신 값"으로 읽는 방식이라면,   
멀티스레드 환경에서 **race condition(경쟁 상태)**이 발생할 수 있다.

예를 들어, 다음과 같은 코드가 있다고 가정하자.  
(실제로 Compose에서는 이렇게 상태를 쓰지 않지만, 개념 설명을 위한 예시다.)

```kotlin
var a = 0
var b = 0

Text("$a $b")
```
위와 같이 변수 `a`, `b`가 정의되어 있고, 이를 `Text` Composable에서 사용하고 있다.  

만약 다른 스레드에서 아래와 같이 변수의 값을 변경하다면,  

```kotlin
a = 10
b = 20
```

UI가 값을 읽는 타이밍에 따라 의도와 다른 결과가 나올 수 있는데,  
다음과 같은 상황이 가능하다.  

```
1. read a → 10
(write b 아직 반영되지 않음)
2. read b → 0
```

결과적으로 UI에는,

```
10 0
```

처럼 **의도하지 않은 중간 상태(inconsistent state)**가 표시될 수 있다.  
이 문제는 여러 상태를 조합해 UI를 구성하는 선언형 UI에서 특히 치명적이다.  
이러한 이유로 Compose는 이를 해결하기 위해 스냅샷을 사용한다.

한편, Compose에서 사용하는 `MutableState` 역시 내부적으로 스냅샷 시스템 위에서 동작한다.

```kotlin
@StateFactoryMarker
public fun <T> mutableStateOf(
    value: T,
    policy: SnapshotMutationPolicy<T> = structuralEqualityPolicy(),
): MutableState<T> = createSnapshotMutableState(value, policy)
```

---
**스냅샷의 읽기는 특정 스냅샷을 기준으로 이루어지고, 쓰기는 새로운 스냅샷에서 수행된다.**    
각 스냅샷은 내부적으로 `SnapshotId`를 가지는데,  
이를 통해 Compose는 **읽기와 쓰기를 논리적으로 분리**한다.

예를 들어,

```
Reader → snapshot #10
Writer → snapshot #11
```

- Reader는 `SnapshotId` `10`을 기준으로 데이터를 읽고,
- Writer는 새로운 `SnapshotId` `11`에 변경 사항을 기록한다.

결과적으로 읽는 동안 값이 바뀌어도 영향이 없고, composition 중 **데이터 일관성**을 유지할 수 있다.  
이를 통해 앞서 살펴본 경쟁 상태 문제를 방지할 수 있다.  