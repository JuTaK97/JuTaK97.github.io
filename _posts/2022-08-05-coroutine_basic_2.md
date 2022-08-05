---
title : Kotlin Coroutines basic 2
categories : 
  - Android
tags :
  - Android 
  - Kotlin
last_modified_at: 2022-08-04T15:53:00+09:00
---

<script src="https://unpkg.com/kotlin-playground@1" data-selector=".kotlin-code"></script>


## Kotlin Coroutines 기초

### suspend functions

suspend function 여러 개의 결과가 필요할 경우 어떻게 할 수 있을까? 가장 단순한 방법은 차례대로(sequentially) 수행하는 것이다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {
    val times = measureTimeMillis {
        val one = one()
        val two = two()
        println(one+two)
    }
    println("${times}ms")
}
suspend fun one(): Int {
    delay(1000L)
    return 29
}
suspend fun two(): Int {
    delay(1000L)
    return 13
}
</div>
두 개의 suspend function을 차례대로 계산하고, 시간은 2000ms 정도 나온다. <br /><br />
하지만 두 함수가 서로 의존관계가 없다면 병렬적으로 계산하는 것이 훨씬 효율적이다. 이 경우 `async{}`라는 좋은 함수가 있다. async는 기본적으로 `launch{}`와 동일하지만 Job을 반환하지 않고 `Deferred<T>`를 반환한다. Deferred는 계산이 끝날 때까지 말 그대로 결과값의 반환을 미룬다. 수행하는 연산이 complete되면 그때서야 결과 값을 얻을 수 있고, 이는 `await()`을 통해 얻을 수 있다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {
    val times = measureTimeMillis {
        val one = async { one() }
        val two = async { two() }
        println(one.await() + two.await())
    }
    println("${times}ms")
}
suspend fun one(): Int {
    delay(1000L)
    return 29
}
suspend fun two(): Int {
    delay(1000L)
    return 13
}
</div>
val one과 val two의 타입은 async{}가 반환한 Deferred<Int>이다. await()을 통해서 연산이 끝날 때 Int 타입으로 결과를 얻을 수 있다. 수행 시간은 sequential할 때의 절반 수준으로 아주 효율적이다.
  
### lazy
`async{}` 내의 연산은 lazy하게 할 수 있다. 반환값이 await()에 의해 요구되거나, `.start()`를 통해 invoke할 때 연산이 시작된다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {
    val times = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { one() }
        val two = async(start = CoroutineStart.LAZY) { two() }
        // some computation
        one.start()
        two.start()   // check what happens when erase this line
        println(one.await() + two.await())
    }
    println("${times}ms")
}
suspend fun one(): Int {
    delay(1000L)
    return 29
}
suspend fun two(): Int {
    delay(1000L)
    return 14
}
</div>
start()로 두 함수를 invoke하기 전까지는 연산이 수행되지 않는다. start()를 하지 않아도 await()에 의해 값이 요구되어 결과 값이 나오기는 하지만 이렇게 되면 sequential하게 진행된다. 따라서 start를 하지 않으면 약 2000ms의 시간이 소요된다.
  
### GlobalScope
[GlobalScope]는 어느 job에도 매여 있지 않은 최상위의 코루틴으로, 어플리케이션의 전체 생명 주기동안 쭉 동작하고 cancel되지 않는다. 지금까지 했던 structured concurrency에 구애받지 않는 코루틴을 만들고 싶다면 GlobalScope를 사용하면 되지만 무척 주의해서 사용해야 한다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() {
    val result = somethingUsefulAsync()
    // what if program aborts here?
    runBlocking {
        println(result.await())
    }
}

@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulAsync() = GlobalScope.async {
    one()
}
suspend fun one(): Int {
    delay(1000L)
    return 1
}
</div>
`somethingUsefulAsync` 함수는 suspend function이 아니다. 하지만 계산 결과를 얻으려면 runBlocking{}을 사용해야 한다.<br /><br />
 
 GlobalScope의 사용이 위험한 이유는 다음과 같다. `val result = somethingUsefulAsync()`와 `result.await()` 사이에 exception이 발생해서 해당 작업이 멈춘다면 보통은 error-handler가 exception을 catch한 후, 다른 해야 할 일을 하게 되지만 somethingUsefulAsync는 GlobalScope이므로 백그라운드에서 계속 돌아가게 된다고 한다. 예시가 있으면 이해가 더 잘 될텐데... 잘몰?루겠다.
  
### structured concurrency
이전 예제에서는 `async{}`를 main의 runBlocking{}을 통해 사용했지만, 별도의 coroutineScope에 담아 사용하면 structured concurrency의 혜택을 볼 수 있다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {
    val time = measureTimeMillis {
        println(add())
    }
    println(time)
}

suspend fun add(): Int = coroutineScope {
    val one = async { one() }
    val two = async { two() }
    one.await() + two.await()
}

suspend fun one(): Int {
    delay(1000L)
    return 13
}

suspend fun two(): Int {
    delay(1000L)
    return 29
}
</div>
이제 add() 함수는 별도의 coroutineScope를 가지기 때문에 one()이나 two()에서 exception이 발생하면 add()의 coroutineScope 내의 모든 코루틴이 취소된다. 그리고 물론 one()과 two()는 병렬적으로 수행되기 때문에 걸린 시간은 1000ms 정도가 나온다. <br /><br />
 
Cancellation도 코루틴 계층을 따라 전파된다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> { 
        try {
            delay(Long.MAX_VALUE) // Emulates very long computation
            42
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> { 
        println("Second child throws an exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}
</div>
two에서 exception이 발생해서 `failedConcurrentSum`의 coroutineScope 내 모든 코루틴(one)이 취소되는 것을 확인할 수 있다.
  
[GlobalScope]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/
