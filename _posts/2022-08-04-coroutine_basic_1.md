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
    	println("main block")
        delay(3000L)
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

## explicit job

`launch{}`는 그 coroutine을 refer하는 Job을 반환한다. 그리고 `Job.join()` 을 이용해서 명시적으로 해당 코루틴이 끝날 때까지 기다리도록 할 수 있다. 아래 코드를 실행해 보자. <br />

<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("main start")
    launch {
        delay(2000L)
        println("main checkpoint")
    }
    child()
    println("main end")
}

suspend fun child() = coroutineScope {
    println("\t child suspend function start")

    val job = launch {
        println("\t\t child job start")
        delay(2500L)
        println("\t\t child job end")
    }

    delay(1000L)
    println("\t child checkpoint 1")
    delay(1000L)
    println("\t child suspend function end")
}
</div>
세 개의 concurrent한 코루틴이 있다. 첫 번째는 main에서 2초를 기다려 checkpoint를 print하는 코루틴이다. 두 번째는 child에서 child job을 시작하고 2.5초를 기다린 후 끝내는 코루틴이다. 마지막으로 child 자체의 코루틴이 있다. 이들이 모두 병렬적으로 실행되기 때문에 출력 결과가 이렇게 나타난다.<br /><br />
하지만 여기에 `job.join()`을 섞으면 상황이 복잡해진다. `val job`이 선언되는 순간은 병렬적으로 `launch{}` 내부의 코드가 실행되지만 `job.join()`을 하는 순간 마치 block을 하는 것처럼 되는 것이다.

<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("main start")
    launch {
        delay(2000L)
        println("main checkpoint")
    }
    child()
    println("main end")
}

suspend fun child() = coroutineScope {
    println("\t child suspend function start")

    val job = launch {
        println("\t\t child job start")
        delay(2500L)
        println("\t\t child job end")
    }

    delay(1000L)
    println("\t child checkpoint 1")
    job.join()		// Explicitly wait!!
    delay(1000L)
    println("\t child suspend function end")
}
</div>
출력 순서가 달라진 것을 확인할 수 있다. Join 하지 않았을 때는 job이 끝나기 전에 child의 두 번째 delay까지 끝나서 child suspend function end가 먼저 나오지만 `job.join()`으로 인해 job이 끝날 때 까지 기다리게 된다.

### cancel
`job.cancel()`은 해당 job을 cancel한다. `job.join()`과 합친 `job.cancelAndJoin()`도 있다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        repeat(5) { i ->
            println("job: I'm sleeping $i ...")
            delay(1000L)
        }
    }
    delay(2500L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    job.join() // waits for job's completion
    println("main: Now I can quit.")
}
</div>
cancel 후 join으로 끝날 때 까지 기다린다. <br />
이는 suspend function들이 cancellable하고, 이들은 코루틴의 cancel을 확인하고 cancel됐을 경우 `CancellationException`을 throw 한다. 다음 예시에서 이를 확인할 수 있다.

<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*
fun main() = runBlocking {
    val job = launch {
        repeat(5) { i ->
            try {
                println("job: I'm sleeping $i ...")
                delay(1000L)
            } catch (e:CancellationException) { 
                println(e)	// Compare the result with `throw e` 
            }
        }
    }
    delay(2500L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    job.join() // waits for job's completion
    println("main: Now I can quit.")
}
</div>
i=3, 4일 때는 코루틴이 cancel되었기 때문에 CancellationException이 발생한다. catch 후 throw하지 않으면 실행 결과와 같이 i=4까지 반복문이 돌게 되고, throw를 해 주면 i=2까지만 출력되고 job이 끝나게 된다.<br />

코루틴이 계속 계산중이라면 cancel을 체크하지 않기 때문에 계속해서 돌아간다. 아래 예시를 확인해 보자.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    var startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var i = 0
        while (i < 5) {
            if (System.currentTimeMillis() > startTime) {
                println("job: I'm sleeping ${i++} ...")
                startTime += 1000L
            }
        }
    }
    delay(2500L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}
</div>
분명 job.cancel()은 실행된 것 같지만 지쳤다고 말하는 main을 제쳐 두고 반복문은 5회를 모두 돌게 된다. try-catch를 이용해서 CancellationException을 잡아 던지도록 코드를 짜 봐도 동일한 결과가 나타난다. 이는 코루틴에서 1초가 경과할 때까지 while문에 의해 계산이 계속되고 있기 때문이다.<br />
계산하는 코드를 cancel하는 방법은 두 가지가 있다. 첫 번째는 반복문 내에서 `yield()`를 사용해서 cancellation을 확인하는 방법이다. 
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    var startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var i = 0
        while (i < 5) {
            if (System.currentTimeMillis() > startTime) {
                yield()						// yield the thread to other coroutines
                println("job: I'm sleeping ${i++} ...")
                startTime += 1000L
            }
        }
    }
    delay(2500L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}
</div>
	
yield()에 의해 잠시 쓰레드 사용권이 다른 코루틴에게 넘어가게 되고, 이 사이에 cancel 신호가 도착하게 된다. <br />
두 번째 방법은 명시적으로 확인하는 방법이다. 해당 coroutineScope의 isActive property를 직접 체크한다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    var startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var i = 0
        while (this.isActive) {		// check whether the coroutine is cancelled
            if (System.currentTimeMillis() > startTime) {
                println("job: I'm sleeping ${i++} ...")
                startTime += 1000L
            }
        }
    }
    delay(2500L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}
</div>

try-catch-finally를 사용하는 경우 보통 `finally{}`에는 file close, job cancel 등 각종 리소스를 정리하는 코드를 넣게 되고, blocking하는 suspend function을 쓸 일은 잘 없다. 애초에 job이 cancel된 후 `finally{}`에서의 suspend function 사용은 CancellationException을 일으키기 때문에 실행되지 않는다. 하지만 finally에서 suspend function을 사용해야 하는 특별한 경우에는 다음 방법을 사용할 수 있다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        try {
            repeat(5) { i ->
                println("job: I'm sleeping $i ...")
                delay(1000L)
            }
        } catch (e: CancellationException) {
            throw e
        } finally {
            withContext(NonCancellable) {		// Always active job
                println("finally...(기 모으는 중)")
                delay(3000L)
                println("finished!!")
            }
        }
    }
    delay(2500L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}
</div>
cancel되어 i=3에서의 반복문 수행은 CancellationException을 일으키고, catch되어 throw된 후 finally가 실행된다. 그리고 finally에서는 suspend function인 delay()가 사용되었다.

### Timeout

코루틴에서 cancel을 사용하는 가장 주된 이유는 실행 시간이 timeout을 초과했기 때문이다. <br />
어떤 job을 계속 따라가면서, 또다른 코루틴으로는 특정 시간이 지나면 그 job을 취소하는 일을 하는 `withTimeout` 함수가 있다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    withTimeout(3500L) {
        repeat(10) { i ->
            println(i)
            delay(1000L)
        }
    }
}
</div>
실행해 보면 전과는 다르게 콘솔에 stack trace가 뜬다. 이전까지 뜨던 CancellationException은 취소된 코루틴 내에서 일어났고, 이는 정상적인 코루틴 completion으로 고려하기 때문에 따로 trace가 뜨지 않았다. 하지만 지금은 main 함수에서 CancellationException이 일어났기 때문에 stack trace가 표시된 것이다.<br />
TimeoutCancellationException은 CancellationException의 subclass이고, 물론 try-catch를 이용해 timeout 후 추가적으로 할 일을 넣을 수 있다. 또는 `withTimeoutOrNull` 을 사용해서 timeout이 나면 null, 나지 않았으면 코드 내의 값을 반환받는 코드를 만들 수 있다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

fun main() = runBlocking {
    val result = withTimeoutOrNull(3500L) {
        repeat(10) { i ->
            println(i)
            delay(1000L)
        }
        "Done"
    }
    println(result)	// will get cancelled before it produces this result
}
</div>
	
`withTimeout()`은 asynchronous하다. 그렇기 때문에 block 밖에서는 release되어야 할 리소스를 관리하는 데 주의해야 한다.
아래 코드에서는 `Resource`라는 클래스를 통해 리소스가 얼마나 반환되지 못하고 leak되는지 알아본다.
<div class="kotlin-code"  theme="darcula" >
var acquired = 0

class Resource {
    init { acquired++ } // Acquire the resource
    fun close() { acquired-- } // Release the resource
}

fun main() {
    runBlocking {
        repeat(100_000) { // Launch 100K coroutines
            launch { 
                val resource = withTimeout(60) { // Timeout of 60 ms
                    delay(50) // Delay for 50 ms
                    Resource() // Acquire a resource and return it from withTimeout block     
                }
                resource.close() // Release the resource
            }
        }
    }
    // Outside of runBlocking all coroutines have completed
    println(acquired) // Print the number of resources still acquired
}
</div>
`launch{}`가 10만번 불리기 때문에 각 launch의 코루틴들은 전부 concurrent하다. 그리고 이들은 전부 main 쓰레드에서 실행되기 때문에 thread-safe하다. 혼잡하지 않다면 60ms가 지날 일이 없지만, 너무 많은 코루틴이 한 쓰레드에서 돌아가기 때문에 일부는 60ms를 넘겨서 exception을 내게 된다. 이 경우 `resource.close()`를 하지 못한 채 끝나게 되므로 결과 값은 거의 항상 양수가 된다.<br /><br />
	
이를 방지하기 위해서는 resource의 reference를 받아서 `finally{}`에서 리소스를 회수하면 된다.
<div class="kotlin-code"  theme="darcula" >
import kotlinx.coroutines.*

var acquired = 0

class Resource {
    init { acquired++ } // Acquire the resource
    fun close() { acquired-- } // Release the resource
}

fun main() {
    runBlocking {
        repeat(100_000) { // Launch 100K coroutines
            launch { 
                var resource: Resource? = null // Not acquired yet
                try {
                    withTimeout(60) { // Timeout of 60 ms
                        delay(50) // Delay for 50 ms
                        resource = Resource() // Store a resource to the variable if acquired      
                    }
                    // We can do something else with the resource here
                } finally {  
                    resource?.close() // Release the resource if it was acquired
                }
            }
        }
    }
    // Outside of runBlocking all coroutines have completed
    println(acquired) // Print the number of resources still acquired
}
</div>
	
[runBlocking]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[coroutineScope]: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
