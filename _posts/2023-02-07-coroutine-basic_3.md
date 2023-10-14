---
title : Kotlin Coroutines basic
categories : 
  - Kotlin
tags :=
  - Kotlin
last_modified_at: 2023-02-07T21:34:00+09:00
---

<script src="https://unpkg.com/kotlin-playground@1" data-selector=".kotlin-code"></script>

## Kotlin Coroutines 기초

### 로컬 IDE 사용하기

https://play.kotlinlang.org/ 에서 간단하게 코틀린 코드를 짜고 실행해 볼 수 있지만, 자동완성도 안 되고 delay 등을 줬을 때 실행되는 것이 잘 보이지 않으므로 Intellij IDE에서 하려고 하는데...
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        println("launch: ${Thread.currentThread().name}")
    }
    println("runBlocking: ${Thread.currentThread().name}")
}

</div>
위 코드는 playground에서는 스레드 이름 뿐만 아니라 코루틴의 번호도 붙어서 출력이 되지만 로컬 IDE에서는 main 밖에 뜨지 않는다.
```
runBlocking: main
launch: main
```
Edit Configuration에 들어가서 VM Options에 `-Dkotlinx.coroutines.debug`를 추가하면 코루틴 번호까지 잘 뜨게 된다!

### Coroutine context and dispatchers
코루틴은 CoroutineContext 타입의 context 내에서 실행된다. Coroutine context에는 앞서 공부했던 `Job`이나 `Dispatcher` 등이 들어 있다.<br />

`launch`나 `async`같은 코루틴 빌더들을 사용할 때 추가적인 파라미터들 중 context에 디스패처를 넣어 줄 수 있다. 디스패처를 사용해서 블록 내 코드가 어떤 스레드에서 어떻게 사용될 지 정해줄 수 있다.<br /><br />

<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }    
}
</div>
결과를 하나씩 보면서 살펴보면, 
1. `Dispatcher.Default`는 JVM의 공유 스레드 풀에서 디스패처가 골라 주는 스레드를 사용하는 것이다. CPU의 코어 개수에 비례하는 스레드 풀에서 작업이 진행되고, 메인 스레드 밖에서 무거운 작업을 할 때 사용한다.<br />
2. `Dispatcher.IO`는 더 많은 스레드가 있는 스레드 풀에서 수행되도록 한다. 이름 그대로 CPU를 많이 사용하지 않는 IO 작업에 사용한다.<br />
3. 아무 파라미터도 넣지 않으면 부모 코루틴의 context를 상속받아 수행한다.
4. `newSingleThreadContext`는 새로운 스레드를 직접 만들어서 수행하는 것으로, 스레드가 비싸기도 하고 관리하기 까다롭기 때문에 `This is a delicate API and..`라고 워닝이 뜬다. 특별한 상황이 아니면 사용하지 않을 것 같다.
5. `Dispatcher.Unconfined`는 조금 특별한데, 처음에는 caller 스레드에서 코루틴을 수행하기 시작하지만 중단점(suspension point) 이후에는 어떤 스레드에서 수행될 지 알 수 없고 suspending function에 의해 그때그때 결정된다. 이 디스패처 또한 일반적인 상황에서는 사용하지 않는다.
6. `Dispatcher.Main`은 안드로이드의 경우 메인 스레드에서 수행하는 것이다. UI를 다루는 작업에 사용한다. 

