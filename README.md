# Kotlin Coroutines : Deep Dive

[1부 코틀린 코루틴 이해하기](#1부-코틀린-코루틴-이해하기)
- [1장 코틀린 코루틴을 배워야 하는 이유](#1장-코틀린-코루틴을-배워야-하는-이유)
- [2장 시퀀스 빌더](#2장-시퀀스-빌더)
- [3장 중단을 어떻게 작동할까?](#3장-중단을-어떻게-작동할까)
- [4장 코루틴의 실제 구현](#4장-코루틴의-실제-구현)
- [5장 코루틴: 언어 차원에서의 지원 vs 라이브러리](#5장-코루틴-언어-차원에서의-지원-vs-라이브러리)

[2부 코틀린 코루틴 라이브러리](#2부-코틀린-코루틴-라이브러리)
- [6장 코루틴 빌더](#6장-코루틴-빌더)
- [7장 코루틴 컨텍스트](#7장-코루틴-컨텍스트)
- [8장 잡과 자식 코루틴 기다리기](#8장-잡과-자식-코루틴-기다리기)
- [9장 취소](#9장-취소)
- [10장 예외 처리](#10장-예외-처리)
- [11장 코루틴 스코프 함수](#11장-코루틴-스코프-함수)
- [12장 디스패처](#12장-디스패처)
- [13장 코루틴 스코프 만들기](#13장-코루틴-스코프-만들기)
- [14장 공유 상태로 인한 문제](#14장-공유-상태로-인한-문제)
- [15장 코틀린 코루틴 테스트하기](#15장-코틀린-코루틴-테스트하기)

[3부 코틀린 코루틴 이해하기](#3부-코틀린-코루틴-이해하기)
- [16장 채널](#16장-채널)
- [17장 셀렉트](#17장-셀렉트)
- [18장 핫 데이터 소스와 콜드 데이터 소스](#18장-핫-데이터-소스와-콜드-데이터-소스)
- [19장 플로우란 무엇인가?](#19장-플로우란-무엇인가)
- [20장 플로우의 실제 구현](#20장-플로우의-실제-구현)
- [21장 플로우 만들기](#21장-플로우-만들기)
- [22장 플로우 생명주기 함수](#22장-플로우-생명주기-함수)
- [23장 플로우 처리](#23장-플로우-처리)
- [24장 공유플로우와 상태플로우](#24장-공유플로우와-상태플로우)
- [25장 플로우 테스트하기](#25장-플로우-테스트하기)

[4부 코틀린 코루틴 적용하기](#4부-코틀린-코루틴-적용하기)
- [26장 일반적인 사용 예제](#26장-일반적인-사용-예제)
- [27장 코루틴 활용 비법](#27장-코루틴-활용-비법)
- [28장 다른 언어에서의 코루틴 사용법](#28장-다른-언어에서의-코루틴-사용법)
- [29장 코루틴을 시작하는 것과 중단 함수 중 어떤 것이 나을까?](#29장-코루틴을-시작하는-것과-중단-함수-중-어떤-것이-나을까)
- [30장 모범 사례](#30장-모범-사례)



# 1부 코틀린 코루틴 이해하기

## 1장 코틀린 코루틴을 배워야 하는 이유
Android에서 api 콜 부터 view 까지 노출하는 과정을 가정해봅시다.

```kotlin
fun onCreate() {
	val news = getNewsFromApi()
	val sortedNews = news.sortedByDescending { it.publishedAt }
	view.showNews(sortedNews)
}
```

굉장히 직관적이고, 가독성도 좋습니다. 어떻게 보면 생각의 흐름대로 작성한 가장 이상적인 코드일지도..

그러나 사실 Android 에서는 위의 예제처럼 간단하게 구현하기 어렵습니다.

`onCreate()` 메소드는 `MainThread` 에서 실행되고, `getNewsFromApi()` 는 메인쓰레드를 블로킹하고, ANR 이 발생할 것 입니다.

`getNewsFromApi()` 가 다른 쓰레드로 호출되어도 `view.showNews()` 에서  `sortedNews` 에 대한 정보가 없으니 원하는 동작대로 작동하지 않을 것 입니다.

### 그럼 쓰레드 전환을 통해서 구현해봅시다

```kotlin
fun onCreate() {
	thread {
		val news = getNewsFromApi()
		val sortedNews = news.sortedByDescending { it.publishedAt }
		runOnUiThread {
			view.showNews(sortedNews)
		}
	}
}
```

`MainThread` 가 아닌 다른 블로킹이 가능한 쓰레드를 사용하고, 이후 결과만 `runOnUiThread` 를 통해 `MainThread` 로 전환했습니다.

위 방법으로 하면 어쩌면 직관적으로 보이지만 몇 가지 문제가 있습니다.

1. 쓰레드가 실행되면 멈출 수 있는 방법이 없습니다. → 메모리 릭이 발생할 수 있음
2. 쓰레드 생성만큼 비용이 많이 듭니다
3. 쓰레드 전환이 잦아지면 복잡도가 늘어나고 관리도 어렵습니다
4. 코드가 길어지고 이해하기 어렵습니다

### 콜백패턴이라면?

콜백의 기본적인 방법으로 함수를 논블로킹으로 만들고, 함수의 작업이 끝났을 때 호출 될 콜백 함수를 넘겨줍니다.

```kotlin
fun onCreate() {
	getNewsFromApi { news ->
		val sortedNews = news.sortedByDescending { it.publishedAt }
		view.showNews(sortedNews)
	}
}
```

콜백 패턴은 몇가지 문제가 발생할 수 있습니다.

1. 여전히 취소 처리가 어렵고, 취소하기 위해 모든 객체를 분리해서 관리해야합니다.
2. 병렬로 처리하기 어렵습니다.
3. 콜백헬이 만들어지고 가독성이 떨어집니다.

```kotlin
// 취소처리를 할 수 없다
// getConfigFromApi()와 getUserFromApi() 는 병렬로 처리할 수 있는 조건이나 구현하기 어렵다
// 반복적인 들여쓰기
fun showNews() {
	getConfigFromApi { config -> 
		getNewsFromApi(config) { news -> 
			getUserFromApi { user ->
				view.showNews(user, news)
			}
		}
	}
}
```

### RxJava 쓰면 되지ㅋ

우선 RxJava를 사용한 코드 예시를 봅시다.

```kotlin
fun onCreate() {
	disposables += getNewsFromApi()
		.subscribeOn(Schedulers.io())
		.observeOn(AndroidSchedulers.mainThread())
		.map {  news ->
			news.sortedByDescending { it.publishedAt }
		}
		.subscribe { sortedNews ->
			view.showNews(sortedNews)
		}
}
```

이전의 코드들을 보다가 보니 편안해지네요.

`disposables` 로 취소 처리도 가능해졌고, 쓰레드 전환도 적절하게 하면서 잘 구현한 코드로 보여집니다.

“***RxJava 에 익숙한 개발자들에게만요.”***

처음 RxJava 를 학습할 때를 떠올려봅시다…

`subscribeOn`, `observeOn`, `map`, `subscribe` , `zip`, `flatMap` 같은 함수들을 익히고, 필요에 따라 `Pair` 로 묶고 분리도 해야했습니다. `Observable`, `Single`, `Maybe` , `Flowable` 같은 클래스들을 공부하고, 함수들을 의도에 맞는 적절한 클래스로 감싸줘야했습니다. 아래처럼 말이죠.

```kotlin
fun getNewsFromApi(): Single<List<News>>
```

### 코루틴이라면 어떨까..?

책의 카피를 인용해보겠습니다.

> 코틀린 코루틴이 도입한 핵심 기능은 코루틴을 특정 지점에서 멈추고 이후에 재개할 수 있다는 것입니다.
> 

여기서 쓰레드 개념을 가져오면 혼동되기 쉽습니다. **“코루틴을 멈춘다”** 라는 의미가 thread 를 점유한다는 의미가 아닙니다.

제 방식대로 표현해본다면, “특정 지점에 마킹해두고, 해당 지점에서 재개할 수 있다”고 표현해 보겠습니다.

더 읽으면 알게 될 `Continuation` 객체를 suspend function의 인자로 넘기게 되며, 이 객체를 활용해서 특정 지점에서 멈추고 시작되는 동작이 가능하게 됩니다. (그래서 비용이 쓰레드를 직접적으로 사용하는 것보다 훨씬 저렴합니다. “light-weight”)

앞의 예시들을 코루틴으로 구현해보겠습니다.

`viewModelScope` 는 안드로이드에서 사용하는 코루틴 스코프 입니다. 코루틴은 코루틴 스코프가 가지고 있는 `Job` 으로 취소처리를 할 수 있습니다.

```kotlin
fun showNews() {
	viewModelScope.launch {
		val config = getConfigFromApi()
		val news = getNewsFromApi(config)
		val user = getUserFromApi()
		view.showNews(user, news)
	}
}
```

위의 코드를 병렬처리를 하기 위해 아래처럼 작성할 수 있습니다.

```kotlin
fun showNews() {
	viewModelScope.launch {
		val config = async { getConfigFromApi() }
		val news = async { getNewsFromApi(config.await()) }
		val user = async { getUserFromApi() }
		view.showNews(user.await(), news.await())
	}
}
```

## 2장 시퀀스 빌더

## 3장 중단을 어떻게 작동할까?

## 4장 코루틴의 실제 구현

## 5장 코루틴: 언어 차원에서의 지원 vs 라이브러리

# 2부 코틀린 코루틴 라이브러리

## 6장 코루틴 빌더
일반 함수가 중단 함수를 호출할 수 없는 이유가 무엇일까요?

앞에서 중단 함수가 어떻게 디컴파일 되는지 공부했습니다. 중단 함수는 마지막 인자로 Continuation 객체를 다른 중단 함수로 전달해야 합니다.

즉, 모든 중단 함수는 또 다른 중단 함수에 의해 호출돼야 합니다.

그렇기 때문에 중단 함수를 시작하는 지점이 반드시 있고, **코루틴 빌더**(`coroutine builder`)가 그 역할을 합니다. 코루틴 빌더 3가지에 대해서 설명해보겠습니다. (launch, runBlocking, async)

### 1. launch

launch 는 `CoroutineScope` 인터페이스의 확장 함수입니다. 

💡 `CoroutineScope` 인터페이스는 부모 코루틴과 자식 코루틴 관계를 정립하기 위한 목적으로 사용되는 **구조화된 동시성(structured concurrency)** 의 핵심입니다.

launch 동작의 결과는 새로운 `thread` 를 시작하는 것과 비슷합니다. 아래의 코드를 봅시다.

```kotlin
fun main() {

// 아래의 java 코드를 보기 전에 결과를 예상해보세요

	GlobalScope.launch {
		delay(1000L)
		println("World")
	}
	GlobalScope.launch {
		delay(1000L)
		println("World")
	}
	GlobalScope.launch {
		delay(1000L)
		println("World")
	}
	println("Hello, ")
	Thread.sleep(2000L)
}
```

```java
fun main() {
// java 로 바꿔보면 이런식으로 짤 수 있습니다
// ^ 위 코틀린 코드부터 보세요 ^

	thread(isDeamon = true) {
		Thread.sleep(1000L)
		println("World")
	}
	thread(isDeamon = true) {
		Thread.sleep(1000L)
		println("World")
	}
	thread(isDeamon = true) {
		Thread.sleep(1000L)
		println("World")
	}
	println("Hello, ")
	Thread.sleep(2000L)
}
```

- **정답 확인하기**
    
    ```kotlin
    // Hello, 
    // (1초 후)
    // World!
    // World!
    // World!
    
    ```
    

**코루틴**을 멈추는 것과 **쓰레드**를 멈추는 것은 엄청난 차이가 있습니다. 결과는 동일하지만, 비용은 코루틴이 훨씬 저렴합니다.

### 2. runBlocking

`runBlocking` 은 코루틴 빌더이지만 쓰레드를 블로킹 합니다. 아래 예제를 봅시다.

```kotlin
fun main() {

// 결과를 예상해보세요

	runBlocking {
		delay(1000L)
		println("World")
	}
	runBlocking {
		delay(1000L)
		println("World")
	}
	runBlocking {
		delay(1000L)
		println("World")
	}
	println("Hello, ")
}
```

- **정답 확인하기**
    
    ```kotlin
    // (1초 후)
    // World!
    // (1초 후)
    // World!
    // (1초 후)
    // World!
    // Hello, 
    ```
    

일반적으로 사용되진 않지만, 필요한 경우는 실제 2가지가 있습니다.

1. 프로그램이 끝나는 것을 방지하지 위해 

2. 비슷한 이유로 테스트를 위한 경우입니다.

### 3. async

`async` 의 return type은 `Deferred<T>` 입니다. 

💡다른 빌더들은?
- `launch` 의 return type 은???  ——————> `Job` 
- `runBlocking` 은 ???  ————————————————————————> `T`


`Deferred` 에는 작업이 끝나면 값을 반환하는 **중단 함수**인 `await()` 가 있습니다.

`launch` 빌더와 비슷하게 `async` 빌더도 호출되자마자 **코루틴을 즉시 시작**합니다.

따라서 몇 개의 작업을 **한번에 시작하고, 모든 결과를 한꺼번에 기다릴 때** 사용할 수 있습니다.

`Deferred` 에 값을 저장하기 때문에 `await()` 에서 값이 반환되는 즉시 값을 사용할 수 있습니다. 하지만 값이 생성되기 전에 `await()` 를 호출하면 값이 나올 때 까지 기다립니다.

예제 코드를 보겠습니다.

```kotlin
fun main() = runBlocking {
	val res1 = GlobalScope.async {
		delay(1000L)
		"Text 1"
	}
	val res2 = GlobalScope.async {
		delay(3000L)
		"Text 2"
	}
	val res3 = GlobalScope.async {
		delay(2000L)
		"Text 3"
	}

	println(res1.await())
	println(res2.await())
	println(res3.await())
}
```

- **펼쳐서 정답 확인하기**
    
    ```kotlin
    // (1초 후)
    // Text 1
    // (2초 후) 
    // Text 2
    // Text 3
    ```
    

### 구조화된 동시성(Structured Concurrency)

 앞의 예제들에서 `launch` 와 `async` 를 사용할 때 `GlobalScope` 가 필요했던 이유는 뭘까요?

이 둘은 `CoroutineScope` 의 확장 함수이기 때문입니다.

함수의 `block` 인자를 보면 `CoroutineScope` 함수형 타입이라는 것을 확인할 수 있습니다.

```kotlin
fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job

fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T>

fun <T> runBlocking(
		context: CoroutineContext, 
		block: suspend CoroutineScope.() -> T
): T
```

부모는 자식들을 위한 스코프를 제공하고, 자식들을 해당 스코프 내에서 호출합니다.

이를 통해 **구조화된 동시성**이라는 관계가 성립됩니다.

부모-자식 관례 특징은 다음과 같습니다.

1. 자식은 부모로부터 컨텍스트를 상속받습니다.
2. 부모는 모든 자식들의 작업이 마칠 때까지 기다립니다.
3. 부모 코루틴이 취소되면 자식 코루틴도 취소됩니다.
4. 자식 코루틴에서 에러가 발생하면, 부모 코루틴도 에러로 소멸합니다.

*스포주의!!*

 *- **취소(CancellationException)는** 부모 코루틴에게 **전파되는 에러**가 아닙니다.*

## 7장 코루틴 컨텍스트

## 8장 잡과 자식 코루틴 기다리기

## 9장 취소

## 10장 예외 처리

## 11장 코루틴 스코프 함수

## 12장 디스패처

## 13장 코루틴 스코프 만들기

## 14장 공유 상태로 인한 문제

## 15장 코틀린 코루틴 테스트하기

# 3부 코틀린 코루틴 이해하기

## 16장 채널

## 17장 셀렉트

## 18장 핫 데이터 소스와 콜드 데이터 소스

## 19장 플로우란 무엇인가?

## 20장 플로우의 실제 구현

## 21장 플로우 만들기

## 22장 플로우 생명주기 함수

## 23장 플로우 처리

## 24장 공유플로우와 상태플로우

## 25장 플로우 테스트하기

# 4부 코틀린 코루틴 적용하기

## 26장 일반적인 사용 예제

## 27장 코루틴 활용 비법

## 28장 다른 언어에서의 코루틴 사용법

## 29장 코루틴을 시작하는 것과 중단 함수 중 어떤 것이 나을까?

## 30장 모범 사례
