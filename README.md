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
- [8장 Job과 자식 코루틴 기다리기](#8장-잡과-자식-코루틴-기다리기)
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



# 1부-코틀린-코루틴-이해하기

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
