---
title: "[Compose Internals] 통제되지 않은 사이드 이펙트 방지(Free of uncontrolled side effects)"
date: 2026-02-12 13:00:00 +0900
categories: [Compose]
tags: [compose internals, compose, composable, side effects]
---
개발을 하면 "사이드 이펙트(Side Effect)"라는 말을 종종 사용하곤 한다. 사이드 이펙트가 뭘까?  
> **사이드 이펙트란(Side Effect)?**  
> 함수(또는 코드)가 자신의 반환값 외에 외부 상태를 변경하거나, 관찰 가능한 변화를 일으키는 것    
> 즉, 순수 함수가 아닌 행동을 일컫는다.(ex. 전역 변수 변경, 네트워크 호출, 로그 출력 등)
{: .prompt-info }
책에서도 이와 같은 맥락으로 호출된 함수의 제어를 벗어나서 발생할 수 있는 예상치 못한 모든 동작을 의미한다.  
다시 말해, 함수는 **결과를 생성하기 위해서 입력값에만 의존하는 것이 아니라, 외부 요인에도 의존**하게 된다.  

이러한 사이드 이펙트가 모호함의 근원이다.  
Compose Runtime은 Composable 함수가 예측 가능하도록(결정론적인) 기대하기 때문에 사이드 이펙트가  
포함된 Composable 함수는 예측이 어려워지고, 결과적으로 Compose에게 좋지 않다.  
이 말은 즉, 사이드 이펙트가 Compose 내에서 아무 통제를 받지 않고 여러번 실행될 수 있음을 의미한다.  
Compose Runtime에게 필수적인 **멱등성을 따르지 않게 된다.**  
(**멱등성** : 같은 연산을 여러번 수행해도 결과가 한 번 수행한 것과 동일한 성질)
> 코드 상으로는 사이드 이펙트로 구분되는 로직을 작성할 수는 있지만, 확실성을 보장해야하는 독립적인  
> UI 트리 구조를 작성하는데 필요한 Composable 함수에서는 개념론적으로 사이드 이펙트를 사용해서는 안된다.  
> (ViewModel에 접근하는 DB 로직이 Composable 안에 있다면, recomposition 발생마다 중복 호출 발생)
{: .prompt-tip}

```kotlin
@Composable
fun EventsFeed(networkService: EventsNetworkService) {
    val events = networkService.loadAllEvents()
		
    LazyColumns {
        items(events) { event ->
                Text(text = event.name)
        }
    }
}
```
위 예시를 살펴보자. 매우 위험한 상황이다.  
Composable 함수는 근본적으로 Compose Runtime에 의해 짧은 시간 내 여러번 다시 실행될 수 있으며(recomposition),  
이로 인해 네트워크 요청이 여러번 수행되어 제어를 벗어날 수 있다.  
더 최악의 상황은 이러한 사이드 이펙트가 아무 조건 없이 다른 스레드에서 실행될 수 있다는 것이다.
> **Compose Runtime은 Composable 함수에 대한 실행 전략을 선택할 권한을 보유한다.**    
> 이는 하드웨어의 멀티 코어의 이점을 활용하기 위해 recomposition을 다른 스레드로 이전시킬 수 있거나,  
> 필요성이나 우선순위에 따라 임의로 순서를 실행할 수 있다.    
> (ex. 화면에 보이지 않는 Composable은 낮은 우선순위로 할당 가능)
{: .prompt-info}

결국 이론상으로는 Composable 함수를 stateless(무상태, 상태를 보존하지 않음)하게 만들려고 노력해야 한다.  
하지만 **대부분의 애플리케이션이 stateful(상태 유지, 상태를 보유하는)하기 때문에, 사이드 이펙트가 필요하다.**    
Jetpack Compose에서는 함수에게 안전하고 통제된 환경에서 이펙트(effect)를 호출할 수 있는  
**이펙트 핸들러(effect handlers)**와 같은 매커니즘을 제공한다.  

이펙트 핸들러는 사이드 이펙트가 **Composable의 lifecycle을 인식**하도록 하여, 해당 lifecycle에 의해  
제한되거나 실행될 수 있게 한다. 또한, Composable 노드가 트리를 떠날 때 자동으로 이펙트를 해제하거나 취소  
할 수 있게 하고, 이펙트에 주어진 입력값이 변경되면 재실행시키거나, 심지어 동일한 이펙트를 recomposition   
과정에서 유지시키고 한번만 호출되게 할 수 있다.  

즉, 이펙트 핸들러는 **Composable 함수 내에서 아무런 제어를 받지 못하고, 직접적으로 이펙트가 호출되는 것을 방지하는**     
**역할을 수행**한다. 아래는 Compose를 써본 사람이면 모두가 알고 있는 코드이다.  
```kotlin
@Composeable
internal fun DetailRoute(
    viewModel: DetailViewModel = hiltViewModel(),
    // ...
) {
    // ...
    LaunchedEffect(Unit) {
        viewModel.sideEffect.collect {
            // ...
        }
    }
}
```
`LaunchedEffect`는 Composable 생명주기에 맞게 사이드 이펙트를 처리해주는 핸들러이다.  
매개변수 `key1`을 지정하면, `key1`가 변경될 때 기존 작업을 재실행한다.   
예시에서는 `Unit`을 키로 사용하여 최초 composition에만 collect를 시작한다. 

이외에도 아래와 같은 핸들러가 있고, 각각 고유한 특성을 가진다.  
- `DisposableEffect` : Composable 생명주기에 맞게 사이드 이펙트 처리(트리를 떠날 때, `onDispose` 실행)
- `SideEffect` : recomposition 될 때마다 1회 실행