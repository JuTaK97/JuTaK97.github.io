---
title : Kotlin Coroutines basic
categories : 
  - Android
tags :
  - Android 
  - Kotlin
last_modified_at: 2022-08-04T15:53:00+09:00
---

<script src="https://unpkg.com/kotlin-playground@1" data-selector=".kotlin-code"></script>


## Kotlin Coroutines 기초

### runBlocking 1
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() {
    println("main start")
    runBlocking {
        launch {
            delay(1000L)
            println("1000 ms")
        }
        launch {
            delay(500L)
            println("500 ms")
        }
    }
    println("main end")
}
</div>

[runBlocking]의 안쪽은 코루틴 세상이고, 바깥은 그렇지 않은 세상이다. 코루틴 세상이 아닌 main 함수 내에서 runBlocking은 새로운 coroutineScope를 열고, 내부의 일이 끝날 때까지 이름 그대로 blocking하게 된다. 

### launch
runBlocking에 의해 열린 코루틴 세상 속에서는 `launch{}`를 이용해 새로운 코루틴을 열 수 있다. `launch{}`는 현재 쓰레드를 block하지 않는 새로운 코루틴을 만들고 `Job`의 형태로 반환한다. 따라서 아래의 두 Job은 동시에(concurrently) 실행된다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() {
    println("main start")
    runBlocking {
        launch {
            delay(1000L)
            println("1000 ms")
        }
        launch {
            delay(500L)
            println("500 ms")
        }
    }
    println("main end")
}
</div>

### suspend function
코루틴 세상에서는 suspend function을 실행할 수 있다. 위 코드에서 suspend function을 사용해 각 `launch{}`의 body를 별개의 함수로 extract할 수 있다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() {
    println("main start")
    runBlocking {
        launch { sus1() }
        launch { sus2() }
    }
    println("main end")
}

suspend fun sus1() {
    delay(1000L)
	println("1000 ms")
}
suspend fun sus2() {
    delay(500L)
	println("500 ms")
}
</div>

### scope builder
coroutine 내에서는 [coroutineScope]를 사용해서 새로운 coroutine scope를 만들 수 있다. coroutineScope{} 로 coroutine scope를 만들고, 이 scope에서 suspend block을 실행하게 된다.

<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() {
    runBlocking {
    	  scope1()
        scope2()
    }
}

suspend fun scope1() = coroutineScope {
    println("scope 1 start")
    delay(1000L)
    println("scope 1 end")
}

suspend fun scope2() = coroutineScope {
    println("scope 2 start")
    delay(2000L)
    println("scope 2 end")
}
</div>

scope1 이 끝날 때까지 scope2는 실행되지 않는다. 이런 점에서 `runBlocking{}`과 유사하지만 runBlocking은 쓰레드를 block하는 반면 coroutineScope는 단지 suspend하고, 쓰레드는 돌아갈 수 있게 풀어 놓는다.


[runBlocking]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[coroutineScope]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
