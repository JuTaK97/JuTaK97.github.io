---
title : Retrofit Call Adapter 내부 동작
categories : 
  - Android
tags :
  - Android 
  - Retrofit
last_modified_at: 2023-10-09T22:27:00+09:00
---

<script src="https://unpkg.com/kotlin-playground@1" data-selector=".kotlin-code"></script>

## Retrofit

Retrofit을 사용할 때는 각 함수에 annotation을 붙인 인터페이스를 만들고, `retrofit.create(MyRestAPI::class.java)` 으로 인스턴스를 만들어서 사용한다.

### case 1
```kotlin
interface MyRestAPI {
    @GET("/myapp/v1/path_of_api")
    fun getSomething(): Call<GetSomethingResult>
}
```

### case 2
```kotlin
interface MyRestAPI {
    @GET("/myapp/v1/path_of_api")
    suspend fun getSomething(): GetSomethingResult
}
```

<br />

간단하게 동기 함수로 쓰기 위해서는 case 1로 만들고 콜백을 이용해 결과 혹은 에러를 얻는다. 약간의 수고를 더해서 비동기 함수를 쓸 때는 case 2같이 사용할 수 있다.
내부에서 뭔가 엄청 잘 돼있는지 이렇게만 해 줘도 무척 잘 되지만, 에러 응답이 올 경우 이를 별도로 처리하기 애매하다.

먼저 내부에서 어떻게 돌아가는지 간단히 살펴본다.


### 1. 요청

우리가 `retrofit.create(MyRestAPI::class.java)`로 \<T\> 객체를 만들 때 Proxy 객체를 만드는데, 이때 InvocationHandler라는 걸 만든다. 
```kotlin
public Object invoke(Object proxy, Method method, Object[] args)
```
이거 하나 갖는 인터페이스이고, 우리가 `create`에 올바르게 service를 넣어 줬으면 

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fretrofit%2Fblob%2F8abb8af6ce730b137c2a9f95c3bb5164c93e955e%2Fretrofit%2Fsrc%2Fmain%2Fjava%2Fretrofit2%2FRetrofit.java%23L164-L167&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

위 코드에서 가장 마지막 loadServiceMethod이 불리고,

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fretrofit%2Fblob%2F8abb8af6ce730b137c2a9f95c3bb5164c93e955e%2Fretrofit%2Fsrc%2Fmain%2Fjava%2Fretrofit2%2FHttpServiceMethod.java%23L149-L153&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

위 함수로 이어진 다음

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fretrofit%2Fblob%2F8abb8af6ce730b137c2a9f95c3bb5164c93e955e%2Fretrofit%2Fsrc%2Fmain%2Fjava%2Fretrofit2%2FHttpServiceMethod.java%23L222-L254&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

위 함수로 이어진다.

우리는 아무 Call Adapter도 넣어주지 않았기 때문에, 224라인의 `callAdapter.adapt(call);` 는 DefaultCallAdapter의 아래 함수로 이어진다.
 
<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fretrofit%2Fblob%2F8abb8af6ce730b137c2a9f95c3bb5164c93e955e%2Fretrofit%2Fsrc%2Fmain%2Fjava%2Fretrofit2%2FDefaultCallAdapterFactory.java%23L58-L61&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

우리는 아무것도 넣어준게 없기 때문에 executor는 null이다. 그래서 그냥 call이 그대로 돌아온다.

다음으로 위 코드의 227라인에서는 args의 마지막 칸에 들어있는 continuation(우리가 부른 함수의 "다음 할 일" 일 것이다)을 꺼내고, 244라인의 await으로 이어진다.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fretrofit%2Fblob%2F8abb8af6ce730b137c2a9f95c3bb5164c93e955e%2Fretrofit%2Fsrc%2Fmain%2Fjava%2Fretrofit2%2FKotlinExtensions.kt%23L32-L63&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

suspendCancellableCoroutine의 내부 블럭에서는 Call.enqueue 함수가 불린다. 이 enqueue는 어떤 call의 enqueue냐 하면, 아까 저 위에서 `new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);`로 만든 call이기 때문에 아래의 enqueue가 불린다.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fretrofit%2Fblob%2F8abb8af6ce730b137c2a9f95c3bb5164c93e955e%2Fretrofit%2Fsrc%2Fmain%2Fjava%2Fretrofit2%2FOkHttpCall.java%23L115-L116&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

위 함수의 바디에서는 아래의 call.enqueue가 불린다.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fretrofit%2Fblob%2F8abb8af6ce730b137c2a9f95c3bb5164c93e955e%2Fretrofit%2Fsrc%2Fmain%2Fjava%2Fretrofit2%2FOkHttpCall.java%23L147-L166&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>

이 call은 RealCall이라는 OkHttp의 클래스 인스턴스라서 아래 코드로 이어진다.

<script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fokhttp%2Fblob%2Fbffc4cb555b1b03fd64874dca8a5f48b54fccdfa%2Fokhttp%2Fsrc%2FjvmMain%2Fkotlin%2Fokhttp3%2Finternal%2Fconnection%2FRealCall.kt%23L164-L169&style=default&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></script>


비동기 함수를 파고들다보면 어디서 멈춰야 할지 잘 모르겠는데, 보통 그냥 dispatcher 가 등장하면 여기까지만 봐야겠다~ 생각하는 편이다. 이번에도 여기에서, dispatcher에게 잘 넘겼겠거니 하고 멈춘다.

### 2. 응답

dispatcher에게 callback과 함께 넘긴 우리의 Call에 대한 응답이 오면 callback이 불린다. 이 콜백은 okhttp3.CallBack인데, 얘를 어디에 넣어줬냐 하면 위위 코드의 `call.enqueue` 안에 구현해서 넣어준 콜백이 있었다.
161라인의 `callback.onResponse(OkHttpCall.this, response);` 가 불리는데, 이 `callback`은 어디에서 넣어준 콜백인가 하면 저 위에 `suspend fun <T : Any> Call<T>.await(): T`에서 구현해서 넣어줬던 Retrofit.CallBack 이다.

위에서 onResponse 함수의 바디만 갖고 와서 보면,

```kotlin
if (response.isSuccessful) {
  val body = response.body()
  if (body == null) {
    val invocation = call.request().tag(Invocation::class.java)!!
    val method = invocation.method()
    val e = KotlinNullPointerException("Response from " +
        method.declaringClass.name +
        '.' +
        method.name +
        " was null but response body type was declared as non-null")
    continuation.resumeWithException(e)
  } else {
    continuation.resume(body)
  }
} else {
  continuation.resumeWithException(HttpException(response))
}
```
40X나 50X의 경우에는 맨 아래 `continuation.resumeWithException(HttpException(response))`가 불린다.
우리의 서버가 보내준 에러 메시지는 `response.errorBody`에 들어있기 때문에, 최상단 try-catch에서 catch하는 HttpException에서 `e.response()?.errorBody()?.string()`로 에러 메시지를 가져올 수 있다.

API 함수를 부르는 곳에서 try-catch를 해서 catch된 exception의 errorBody를 파싱해서 써도 되지만, 기왕이면 더 하단에서 모두 파싱을 끝낸 후에 우리의 커스텀 Exception 객체로 exception을 발생시키고 싶다.
이유가 꼭 이것뿐만은 아니고, 에러 원인이 40X나 50X가 아니라 Http Fail이나 와야 하는 body가 오지 않은 경우에는 HttpException이 아니라 KotlinNullPointerException이나 UnknownHostException가 오는데 이를 상단 호출부에서 분기를 치는 게 무척 어색하기 때문이기도 하다.


## Custom Call Adapter
내부 동작을 간단히 알아봤는데, 이제 커스텀 CallAdapter를 만들어야 할 필요성을 알게 되었다. Retrofit의 빌더에 커스텀 CallAdapter를 넣어 준다.

```kotlin
return Retrofit.Builder()
    .baseUrl("http://~~~~~~.com")
    .client(okHttpClient)
    .addConverterFactory(MoshiConverterFactory.create(moshi)) 
    .addCallAdapterFactory(MyCallAdapterFactory()) // 여기
```

<br />
MyCallAdapterFactory와 MyCallAdapter는 아래와 같이 만들 수 있다.

```kotlin
class MyCallAdapterFactory: CallAdapter.Factory() {
    override fun get(
        returnType: Type,
        annotations: Array<out Annotation>,
        retrofit: Retrofit
    ): CallAdapter<*, *> {
        return MyCallAdapter<Any>(returnType)
    }
}
```

```kotlin
class MyCallAdapter<R>(
    private val bodyType: Type,
): CallAdapter<R, Any> {

    override fun responseType(): Type {
        return (bodyType as? ParameterizedType)?.let {
            it.actualTypeArguments[0]
        } ?: Unit::class.java
    }

    override fun adapt(call: Call<R>): Any {
        return call.let {
            object: Call<R> by it {
                override fun enqueue(callback: Callback<R>) {
                    it.enqueue(object: Callback<R> {
                        override fun onResponse(call: Call<R>, response: Response<R>) {
                            if (response.isSuccessful) {
                                callback.onResponse(call, response)
                            } else {
                                callback.onFailure(call, parseErrorBody(response))
                            }
                        }

                        override fun onFailure(call: Call<R>, t: Throwable) {
                            callback.onFailure(call, t)
                        }
                    })
                }
            }
        }
    }
}
```

뭔가 굉장히 복잡한데, 뭘 해야되는지만 이해하면 된다.
Retrofit 라이브러리 코드 내에서 Call 인스턴스를 만드는데, 우리는 이 Call의 함수 중 enqueue를 수정할 것이다. 정확히는 enqueue 안에 들어가는 CallBack에 손을 댈 건데, 사이에 한 계층을 추가한다.

커스텀 Call Adapter를 넣지 않았다면, 에러일 때 아래 주석으로 표시한 라인이 불리게 된다.

```kotlin
override fun onResponse(call: Call<T>, response: Response<T>) {
  if (response.isSuccessful) {
    // ...
  } else {
    continuation.resumeWithException(HttpException(response)) // 여기가 불리는 것이 문제
  }
}

override fun onFailure(call: Call<T>, t: Throwable) {
  continuation.resumeWithException(t) // 커스텀 Error 객체를 여기로 쏴 주는 것이 목표
}
```

거기로 가지 않고 `resumeWithException(t)`으로 가도록 아래와 같이 가로챈다.

```kotlin
it.enqueue(object: Callback<R> {
    override fun onResponse(call: Call<R>, response: Response<R>) {
        if (response.isSuccessful) {
            callback.onResponse(call, response)
        } else {
            callback.onFailure(call, parseErrorBody(response)) // 여기
        }
    }

    override fun onFailure(call: Call<R>, t: Throwable) {
        callback.onFailure(call, t)
    }
})
```

위 코드의 else문(주석 `//여기`)에서 parse한 후 callback.onFailure를 불러서 우리의 커스텀 Exception으로 resumeWithException할 수 있도록 유도한다.

마지막으 parseErrorBody는 각자 앱마다의 구현을 해 주면 된다.
스누티티 안드로이드의 경우 serializer를 이용해 `response.errorBody()?.string()`를 ErrorDTO의 형태로 deserialize한 후 response와 함께 `ErrorParsedHttpException`라는 클래스의 형태로 throw한다. [(코드)](https://github.com/wafflestudio/snutt-android/blob/develop/app/src/main/java/com/wafflestudio/snutt2/lib/network/call_adapter/ErrorParsingCallAdapter.kt)