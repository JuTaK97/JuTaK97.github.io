---
title : Jetpack Compose를 사용하며 겪은 일 1
categories : 
  - Android
tags :
  - Android 
  - Jetpack Compose
last_modified_at: 2023-02-20T15:53:00+09:00
---

<script src="https://unpkg.com/kotlin-playground@1" data-selector=".kotlin-code"></script>


## 이론적 배경

[공식 문서]에서 알 수 있듯, 우리는 `rememberCoroutineScope()`를 사용해서 Composable 함수 내에서 쉽게 suspend function들을 사용할 수 있다.
들어가 보면 긴 주석이 달려 있다.
<div class="kotlin-code"  theme="darcula" >
/**
 * Return a [CoroutineScope] bound to this point in the composition using the optional
 * [CoroutineContext] provided by [getContext]. [getContext] will only be called once and the same
 * [CoroutineScope] instance will be returned across recompositions.
 *
 * This scope will be [cancelled][CoroutineScope.cancel] when this call leaves the composition.
 * The [CoroutineContext] returned by [getContext] may not contain a [Job] as this scope is
 * considered to be a child of the composition.
 *
 * The default dispatcher of this scope if one is not provided by the context returned by
 * [getContext] will be the applying dispatcher of the composition's [Recomposer].
 *
 * Use this scope to launch jobs in response to callback events such as clicks or other user
 * interaction where the response to that event needs to unfold over time and be cancelled if the
 * composable managing that process leaves the composition. Jobs should never be launched into
 * **any** coroutine scope as a side effect of composition itself. For scoped ongoing jobs
 * initiated by composition, see [LaunchedEffect].
 *
 * This function will not throw if preconditions are not met, as composable functions do not yet
 * fully support exceptions. Instead the returned scope's [CoroutineScope.coroutineContext] will
 * contain a failed [Job] with the associated exception and will not be capable of launching
 * child jobs.
 */
 </div>
조금 어렵지만.. 대략 이해하자면 현 시점의 컴포지션에 바인딩 된 어떤 코루틴 컨텍스트를 반환해 주는 것 같다. 컴포지션을 떠나면 이 스코프는 `cancel`될 것이며, 터치 이벤트 같은 콜백에 사용하도록 만들어 졌고 컴포지션 자체의 사이드 이펙트로서 `launch`되면 절대 안된다고 설명되어 있다.

그동안 SNUTT에서는 별 생각 없이 `scope.launch{}`를 마구 써 왔다. 대부분의 launch가 API를 부르는 데 사용되었기 때문에 큰 문제는 없는데
이번에 관심강좌 마이너 픽스를 하면서 그동안 겪지 못했던 coroutineScope 관련 조금 다른 경험을 하게 되었다.

## 상황

Jetpack Navigation에서 하나의 destination으로 사용되는 컴포저블 함수 `BookmarkPage()`가 있다.
<div class="kotlin-code"  theme="darcula" >
    NavHost(
        // ...
    ) {
        // ...

        composable2(NavigationDestination.Bookmark) {
            val parentEntry = remember(it) {
                navController.getBackStackEntry(NavigationDestination.Home)
            }
            val searchViewModel = hiltViewModel&#60;SearchViewModel&#62;(parentEntry)
            BookmarkPage(searchViewModel)
        }
    }
</div>

`BookmarkPage()` 컴포저블에서는 `SearchViewModel`이 들고 있는 `StateFlow<List<LectureDto>>`를 `.collectAsState()`를 이용해 구독해서, `LazyColumn`을 이용해 아래와 같은 화면을 구성하게 된다.

여기서 자세히 버튼을 누르면 `BookmarkPage()`의 메인 레이아웃인 `ModalBottomSheetLayout`의 `sheetState`에서 제공하는 `.show()` api를 이용해서 강의 상세 바텀시트를 열게 되고, 이 강의 상세 페이지에서도 관심강좌 지정/해제가 가능하도록 하는 것이 이번 마이너 패치의 핵심 내용이었다.

강의 상세 페이지에 새로 추가된 `관심강좌 제외` 혹은 `관심강좌 추가` 버튼이 이 역할을 하게 되고, 코드는 다음과 같다.

<div class="kotlin-code"  theme="darcula" >
    scope.launch {
        launchSuspendApi(apiOnProgress, apiOnError) {
            if(isBookmarked) {
                searchViewModel.deleteBookmark(editingLectureDetail)
                context.toast("관심장좌 목록에서 제외하였습니다.")
            } else {
                searchViewModel.addBookmark(editingLectureDetail)
                context.toast("관심장좌 목록에 추가하였습니다.")
            }
        }
    }
</div>

이때의 `scope`는 `LectureDetailPage()`에서 `rememberCoroutineScope()`로 가져온 코루틴 스코프이다. 

검색한 강의의 상세 정보를 바텀시트를 통해 볼 때는 뒤로가기를 누르면 `onCloseViewMode` 파라미터로 받아온 람다를 실행하게 되는데, 이는 바텀시트를 닫기 위함이다. 바텀시트를 닫기 위해 필요한 `sheetState`는 `LectureDetailPage`가 아니라 이를 바텀시트로 띄운 상위 컴포저블이 가지고 있기 때문에 람다로 전달해서 사용하고 있다.

## 문제 상황

문제가 된 코드는 이곳이다.
<div class="kotlin-code"  theme="darcula" >
    LectureDetailPage(onCloseViewMode = { 
        scope.launch {
            sheetState.hide()
        }
    }, vm = lectureDetailViewModel, searchViewModel = searchViewModel)
</div>

도대체 왜인지 모르겠지만 `scope.launch{}` 내의 코드가 아예 동작하기 않아서 뒤로가기를 눌러도 바텀시트가 닫히지를 않았다. 로그를 찍어 봐도, 아예 블록이 실행 자체가 되지 않았다. 그것도, 꼭 바텀시트 내 강의 상세 페이지에서 관심강좌 제외 후 다시 추가를 했을 때만 이런 현상이 발생했다.


## 원인 파악 시도

새로 추가한 피처(관심강좌 제외&추가)를 하지 않으면 뒤로가기가 멀쩡히 잘 동작했으니, 당연히 이 부분이 문제일 것이라고 최초에 판단을 했다.
그런데 API를 잘못 쏜 것도 아니고, 400이 온 것도 아니다. 너무나 멀쩡하게 돌아가고 있었고, `CancellationException`이 발생하지도 않았다.

온갖 것의 로그를 찍어 보다가 `scope.launch{}`가 반환한 `Job`의 로그를 보다가 단서를 찾았다. 

<div class="kotlin-code"  theme="darcula" >
    LectureDetailPage(onCloseViewMode = { 
        val job = scope.launch {
            sheetState.hide()
        }
        Log.d("aaaa", job.isCancelled.toString())   // 로그의 출력은 true
    }, vm = lectureDetailViewModel, searchViewModel = searchViewModel)
</div>

Job의 상태가 cancelled이었다. 무엇 때문에 cancel되었을까 찾다가, 이번에는 `scope.toString()`으로 로그를 찍어 보았다. 

<div class="kotlin-code"  theme="darcula" >
    LectureDetailPage(onCloseViewMode = { 
        Log.d("aaaa", scope.coroutineContext.toString())    // 로그
        scope.launch {
            sheetState.hide()
        }
    }, vm = lectureDetailViewModel, searchViewModel = searchViewModel)
</div>

출력 결과는 
```
[androidx.compose.ui.platform.MotionDurationScaleImpl@e4a79f7, androidx.compose.runtime.BroadcastFrameClock@2e96064, JobImpl{Cancelled}@74fd0bf, AndroidUiDispatcher@d8c782]
```

모종의 이유로, 이 scope의 Job들은 모두 cancel당한다. 원인을 찾기 위해 조금 더 삽질을 하다가, scope를 다른 걸 써보자 생각이 들어서 `onCloseViewMode` 람다에 scope를 전달해주도록 바꾸어 보았다. 즉, `onCloseViewMode`를 부르는 쪽인 `LectureDetailPage()`의 scope를 가져와 사용해 보았다.

<div class="kotlin-code"  theme="darcula" >
    LectureDetailPage(onCloseViewMode = { newScope ->
        newScope.launch {
            sheetState.hide()
        }
    }, vm = lectureDetailViewModel, searchViewModel = searchViewModel)
</div>

그랬더니 짜잔! 정상적으로 동작했다.

## 해결 및 결론

원래 사용하던 scope는 자세히 버튼을 누른 `SearchListItem()` 컴포넌트 컴포저블의 것이다. 하지만 관심강좌 제외를 하면 `BookmarkPage()`에서 구독하고 있는 관심강좌 리스트에서 해당 강의가 사라지고, 이로 인해 scope가 묶여 있던 컴포저블이 사라지게 되어 이 scope가 cancel되었던 것이다.
뒤에 있던 목록이 바텀시트에 가려서 안 보여서 원인을 찾는 데 시간이 좀 걸렸다. 한 2시간 삽질했다..

그래도 공식 문서에서 글로만 읽었던 `This scope will be [cancelled][CoroutineScope.cancel] when this call leaves the composition.`라는 문장을 직접 겪어 보니 좋은 경험이었다.


[공식 문서]: https://developer.android.com/jetpack/compose/kotlin#coroutines