---
title : Lifecycle-Aware Component
categories : 
  - Android
tags :
  - Android 
last_modified_at: 2023-10-14T22:27:00+09:00
---

<script src="https://unpkg.com/kotlin-playground@1" data-selector=".kotlin-code"></script>

# Lifecycle

안드로이드의 Activity, Fragment와 같은 구성 요소들은 framework(혹은 시스템)에서 관리되는 생명 주기를 갖는다.

이러한 구성 요소에 종속되어서, 해당 구성 요소의 생명 주기에 따라 동작하는 클래스를 만들어야 하는 경우가 있다.
예를 들어 다음과 같이 위치 정보를 수신하는 클래스가 있다.

```kotlin
class MyLocationListener(
        private val context: Context,
        private val callback: (Location) -> Unit
) {
    fun start() {
        // connect to system location service
    }

    fun stop() {
        // disconnect from system location service
    }
}
```

MyLocationListener 객체의 생명 주기를 Activity의 생명 주기에 종속시키기 위해 다음과 같은 코드를 추가한다.

```kotlin
class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this) { location ->
            // update UI using Location
        }
    }

    public override fun onStart() {
        super.onStart()
        myLocationListener.start()
        // manage other components that need to respond to the activity lifecycle
    }

    public override fun onStop() {
        super.onStop()
        myLocationListener.stop()
        // manage other components that need to respond to the activity lifecycle
    }
}
```

하지만 이런 컴포넌트가 많아진다면 액티비티의 생명 주기 함수들 (onStart, onStop 등)에서 해야 할 일이 그만큼 비대해진다. 액티비티의 생명 주기에 종속된 다양한 컴포넌트들을 모두 여기서 관리해 줄 책임이 생기므로, 유지보수하기 어려워진다.

또한 이로 인해 일종의 race condition이 발생할 수 있는데, 

```kotlin
    public override fun onStart() {
        super.onStart()
        Util.checkUserStatus { result ->
            // what if this callback is invoked AFTER activity is stopped?
            if (result) {
                myLocationListener.start()
            }
        }
    }

    public override fun onStop() {
        super.onStop()
        myLocationListener.stop()
    }
```
이렇게 되어 있다고 하면 `checkUserStatus`가 오래 걸리는 경우, 이미 액티비티가 onStop된 이후에 `myLocationListener.start()` 가 불릴 수 있다.

`Lifecycle`을 이용해 이 문제를 해결할 수 있다.

## Lifecycle, LifecycleOwner

Lifecycle에는 Event와 State를 나타내는 enum class가 정의되어 있다. 아래 이미지에서 event와 state의 개념을 시각적으로 알 수 있다.
<image src="https://developer.android.com/static/images/topic/libraries/architecture/lifecycle-states.svg"/>
<br /> Lifecycle을 implement하는 클래스는 위 개념대로 움직이게 될 것이다.

Lifecycle은 하나의 변수와 두 가지 함수를 갖는다.
```kotlin
public abstract val currentState: State
public abstract fun addObserver(observer: LifecycleObserver)
public abstract fun removeObserver(observer: LifecycleObserver)
```
자신(Lifecycle)을 observe하는 LifecycleObserver를 추가/제거하는 함수이다.

Activity나 Fragment가 직접 Lifecycle을 implement하지는 않고, LifecycleOwner라는 인터페이스를 implement한다.

```kotlin
public interface LifecycleOwner {
    public val lifecycle: Lifecycle
}
```

Activity 클래스부터 LifecycleOwner를 implement하진 않고, 한 단계 extend된 ComponentActivity에서부터 다음 방식으로 implement한다.

```kotlin
// ComponentActivity.java
private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

@Override
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
```

## LifecycleRegistry
주석에는 다음과 같이 소개되어 있다.
> An implementation of Lifecycle that can handle multiple observers.

생성자는 아래와 같다.
```kotlin
open class LifecycleRegistry private constructor(
    provider: LifecycleOwner,
    private val enforceMainThread: Boolean
) : Lifecycle() {
    // ...
    constructor(provider: LifecycleOwner) : this(provider, true)
}
```

생성자로 받은 provider는 다음과 같이 weak reference로 저장된다.
```kotlin
private val lifecycleOwner: WeakReference<LifecycleOwner>
init {
    lifecycleOwner = WeakReference(provider)
}
```
이렇게 함으로써 사용자가 Activity나 Fragment의 lifecycle을 leak시킨다 하더라도 Activity나 Fragment 자체가 leak되진 않는다.

LifecycleRegistry는 여러 observer들을 관리하기 위해 observer들을 저장해 놓을 필요가 있는데, FastSafeIterableMap를 사용한다.
```kotlin
/**
    * Custom list that keeps observers and can handle removals / additions during traversal.
    *
    * Invariant: at any moment of time for observer1 & observer2:
    * if addition_order(observer1) < addition_order(observer2), then
    * state(observer1) >= state(observer2),
    */
private var observerMap = FastSafeIterableMap<LifecycleObserver, ObserverWithState>()
```
재밌어 보이는 invariant가 존재한다. 

LifecycleRegistry를 동작시키는 핵심적인 함수는 아래 두 함수이다.
```kotlin
open fun handleLifecycleEvent(event: Event) {
    enforceMainThreadIfNeeded("handleLifecycleEvent")
    moveToState(event.targetState)
}

private fun moveToState(next: State) {
    if (state == next) {
        return
    }
    check(!(state == State.INITIALIZED && next == State.DESTROYED)) {
        "no event down from $state in component ${lifecycleOwner.get()}"
    }
    state = next
    if (handlingEvent || addingObserverCounter != 0) {
        newEventOccurred = true
        // we will figure out what to do on upper level.
        return
    }
    handlingEvent = true
    sync()
    handlingEvent = false
    if (state == State.DESTROYED) {
        observerMap = FastSafeIterableMap()
    }
}
```
외부에서 특정 Event를 전달하면 해당 Event의 targetState로 이동하기 위한 작업을 수행한다. targetState는 해당 Event를 완료하면 도달할 State이다.
handlingEvent, newEventOccurred, addingObserverCounter 같은 변수들이 있는데 reentrance를 보장하기 위한 로직으로 보이므로 패스하고 핵심 로직만 살펴보면, `state = next`와 `sync()` 두 가지 작업을 한다.

LifecycleRegistry의 state를 바꾼 후 sync에서는 observer들에게 해당 이벤트를 전파해 준다. 
```kotlin
private fun sync() {
    val lifecycleOwner = lifecycleOwner.get() ?: // 생략
    while (!isSynced) {
        if (state < observerMap.eldest()!!.value.state) {
            backwardPass(lifecycleOwner)
        }
        val newest = observerMap.newest()
        if (!newEventOccurred && newest != null && state > newest.value.state) {
            forwardPass(lifecycleOwner)
        }
    }
}

private val isSynced: Boolean
    get() {
        if (observerMap.size() == 0) {
            return true
        }
        val eldestObserverState = observerMap.eldest()!!.value.state
        val newestObserverState = observerMap.newest()!!.value.state
        return eldestObserverState == newestObserverState && state == newestObserverState
    }
```
isSynced 조건이 될 때까지 while문을 반복하는데, isSynced는 모든 observer의 상태가 state와 동일한 것을 의미한다.
while문애서는 두 가지 동작을 하는데,
1. eldest observer의 state가 Lifecycle의 state보다 크면 backwardPass 수행
2. newest observer의 state가 Lifecycle의 state보다 작으면 forwardPass 수행

(state의 대소관계는 Destroyed \< INITIALIZED \< CREATED \< STARTED \< RESUMED 순서이다.)

Lifecycle이 onStart 이벤트를 받아서 CREATED -> STARTED로 state change가 있었다고 하자.
Invariant에 의해 eldest observer는 가장 큰 state를 갖는다. 따라서 현재 lifecycle의 state보다 state가 큰 observer가 있는지를 알아보기 위해 eldest observer의 state와 비교한 것이다. 반대로 lifecycle의 state보다 작은 state를 갖는 observer가 있는지를 보려면 oldest observer의 state가 lifecycle의 state보다 작은지 검사하면 된다.
결국 모든 observer들의 state를 Lifecycle의 state로 sync시키는 작업을 여기서 하게 된다.

forwardPass와 backwardPass를 자세히 보기 전에, addObserver를 봐야 한다.
```kotlin
override fun addObserver(observer: LifecycleObserver) {
    val initialState = if (state == State.DESTROYED) State.DESTROYED else State.INITIALIZED
    val statefulObserver = ObserverWithState(observer, initialState)
    observerMap.putIfAbsent(observer, statefulObserver)
    val lifecycleOwner = lifecycleOwner.get()?: // 생략
    var targetState = calculateTargetState(observer)
    while (statefulObserver.state < targetState && observerMap.contains(observer)) {
        val event = Event.upFrom(statefulObserver.state) ?: // 생략
        statefulObserver.dispatchEvent(lifecycleOwner, event)
        targetState = calculateTargetState(observer)
    }
    if (!isReentrance) {
        sync()
    }
}
```
addObserver에서 하는 주요한 동작은 다음 2가지이다.
1. observer를 initial state와 함께 ObserverWithState로 래핑 후 observerMap에 추가
2. statefulObserver의 state를 targetState로 조정 (이 과정에서 각 단계별 Event 전달)

주석을 보면 더 이해하기 쉽다.
> Adds a LifecycleObserver that will be notified when the LifecycleOwner changes state.

> The given observer will be brought to the current state of the LifecycleOwner. For example, if the LifecycleOwner is in Lifecycle.State.STARTED state, the given observer will receive Lifecycle.Event.ON_CREATE, Lifecycle.Event.ON_START events.

Initial state는 INITIALIZED로 주고, LifeCycle의 state까지 차례차례 Event를 줘서 승급시키는 느낌이다.
ObserverWithState로 래핑하면서 dispatchEvent라는 함수가 생겼는데, 아래 코드를 보면 금방 이해할 수 있다.
```kotlin
internal class ObserverWithState(observer: LifecycleObserver?, initialState: State) {
    var state: State
    var lifecycleObserver: LifecycleEventObserver

    init {
        lifecycleObserver = Lifecycling.lifecycleEventObserver(observer!!)
        state = initialState
    }

    fun dispatchEvent(owner: LifecycleOwner?, event: Event) {
        val newState = event.targetState
        state = min(state, newState)
        lifecycleObserver.onStateChanged(owner!!, event)
        state = newState
    }
}
```
별 거 없고, Observer마다 State를 부여하고 onStateChanged를 불러주는 게 끝이다. 

조금 재밌는 부분은 Lifecycling.lifecycleEventObserver인데, 아무 function도 없는 LifecycleObserver 인터페이스의 익명 객체를 LifecycleEventObserver로 변환시키는 함수이다.

내부를 보면 조금 정신없지만, 각 케이스별로 Adapter를 만들어놔서 어떤 게 들어오든 LifecycleEventObserver로 바꾸는 것을 볼 수 있다. 복잡한 기타 케이스는 볼 필요 없어 보이고 DefaultLifecycleObserver인 경우만 보면, DefaultLifecycleObserverAdapter에서 각 Event마다 onCreate, onStart 등 DefaultLifecycleObserver의 함수로 매칭시켜 놓은 것을 확인할 수 있다.

다시 돌아와서 forwardPass와 backwardPass를 보면
- forwardPass에서는 oldest observer부터 순회하며, observer의 state에서 한 칸 up하는 Event를 observer에게 전달한다.
- backwardPass에서는 newest observer부터 순회하며, observer의 state에서 한 칸 down하는 Event를 observer에게 전달한다.

이를 통해 Invariant를 유지하면서 observer들의 state를 바꾼다.

<br />
정리하면, Activity나 Fragment는 LifecycleRegistry를 이용해 LifecycleOwner를 implement하고, 자기 자신의 생명 주기 함수(onCreate 등)에서 적절히 LifecycleRegistry.handleLifecycleEvent를 불러서 자신의 Lifecycle state를 관리한다.

이렇게 제공되는 Lifecycle을 이용하는 것은 LifecycleObserver이다.
## LifecycleObserver
위에서 계속 살펴봤듯, LifecycleRegistry 안의 map에 LifecycleObserver들이 저장된다. LifecycleObserver는 자기 자신을 lifecycle owner에게 등록하고, owner가 보내 주는 dispatchEvent를 받게 된다.

이와 같이 Lifecycle에 종속되어 살아가는 lifecycle-aware components들이 존재하는데, 대표적으로 LiveData가 있다.

LiveData는 주로 다음과 같이 observe한다.
```kotlin
// Activity 내에서..
viewModel.data.observe(this) { newValue ->
    // UI update or do something
}
```

observe 함수 내부를 보면, 람다로 받은 `Observer` 인터페이스 익명 객체를 `LifecycleBoundObserver`라는 자체 클래스로 래핑한다. 이 클래스는 `LifecycleEventObserver`를 implement하고, LiveData 자체의 로직과 관련된 `ObserverWrapper`를 extend한다.

우리가 지금 보고 싶은 건 LiveData의 observer가 this(=Activity)의 Lifecycle에 어떻게 종속되어 있는지 이다. LifecycleEventObserver를 어떻게 implement했는지 보면 된다.

```kotlin
@Override
public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
    Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
    if (currentState == DESTROYED) {
        removeObserver(mObserver);
        return;
    }
    Lifecycle.State prevState = null;
    while (prevState != currentState) {
        prevState = currentState;
        activeStateChanged(shouldBeActive());
        currentState = mOwner.getLifecycle().getCurrentState();
    }
}
```
owner의 lifecycle이 DESTROYED라면 removeObserver를 진행한다. 이 함수는 Lifecycle이랑은 관계없는 LiveData의 함수인데, LiveData가 자체적으로 가지고 있는 observer 리스트에서 해당 observer를 제거한다.

owner의 lifecycle이 STARTED 이상이라면(shouldBeActive() == true) `activeStateChanged(true);`를 수행한다.
```kotlin
void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        return;
    }
    // immediately set active state, so we'd never dispatch anything to inactive owner
    mActive = newActive;
    changeActiveCounter(mActive ? 1 : -1);
    if (mActive) {
        dispatchingValue(this);
    }
}
```
mActive는 각 observer마다 갖는 값인데 이게 false였다면, 즉 observe 후 아직 활성화되지 않은 상태라면 true로 바꿔 준 후 dispatchingValue를 불러서 LiveData들의 모든 observer의 onChanged를 트리거시킨다. 결국 그냥 최초 observe 때 owner가 STARTED 이상이라면 값을 emit하도록 하는 부분이다.

이런 식으로 owner의 Lifecycle 변화 이벤트를 owner가 아닌 LiveData 내부에서 수신하고, LiveData가 자신의 생명 주기를 조절할 수 있게 된다.

## 정리
맨 처음 LocationListener의 예시에서 두 가지 문제점이 있었다. 하지만 이제 LocationListener가 LifecycleObserver  (LifecycleEventObserver 혹은 DefaultLifecycleObserver)를 implement하도록 하면 이 문제점들이 해결된다.

Activity나 Fragment의 생명 주기에 종속된 컴포넌트가 많아 질수록 Activity와 Fragment 자체의 생명 주기 함수들에서 할 일이 너무 많아진다는 문제가 있었다. 하지만 이제는 해당 로직이 Observer 내부에 모두 존재하고, owner가 해 줘야 할 일은 observer를 등록해 주기만 하면 끝이므로 해결됐다.

또한 Activity의 onStart가 너무 비대해져서 race condition이 생기는 문제가 있었는데, 이 또한 observer가 owner의 lifecycle의 current state를 상시 확인할 수 있게 됐으므로 해결됐다.


## Further

- Lifecycle-aware한 Coroutine 
- ViewModel, Fragment