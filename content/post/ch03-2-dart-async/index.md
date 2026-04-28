---
title: "Ch03-2. Dart 비동기 - Future, Stream"
slug: "ch03-2-dart-async"
date: 2026-04-26
weight: 5
draft: false
tags: ["Flutter", "Dart", "Future", "async", "await", "Stream", "BroadcastStream"]
categories: ["Flutter"]
description: "Dart의 Future, async/await, Stream, BroadcastStream을 Swift의 비동기 패턴과 비교하며 정리한 기록"
---

Ch03 두 번째는 비동기 프로그래밍이다. Future와 Stream이 핵심인데, Swift의 async/await이랑 비교하면 이해가 빠르다.

## Future - 미래의 값

`Future<T>`는 "아직 안 끝난 비동기 작업의 결과"를 나타낸다. [Dart 공식 문서](https://dart.dev/codelabs/async-await)에 따르면 두 가지 상태가 있다:

- **Uncompleted**: 작업 진행 중
- **Completed**: 값 또는 에러로 완료

Swift에서는 `async` 함수가 값을 직접 반환하고 런타임이 내부적으로 관리하는데, Dart는 `Future<T>`라는 래퍼 객체를 명시적으로 반환한다. 타입 시그니처에 비동기 여부가 드러나는 셈이다.

```dart
// Dart: 반환 타입이 Future<String>
Future<String> fetchUserName() async {
  await Future.delayed(Duration(seconds: 1));
  return 'jHoon';
}
```

```swift
// Swift: 반환 타입이 그냥 String, async가 키워드로 붙음
func fetchUserName() async -> String {
    try await Task.sleep(nanoseconds: 1_000_000_000)
    return "jHoon"
}
```

### async/await

사용법은 Swift랑 거의 똑같다. `async` 붙이고 `await`으로 기다리면 된다.

```dart
void main() async {
  print('주문 시작');
  var order = await fetchOrder();
  print('주문 완료: $order');
}

Future<String> fetchOrder() async {
  await Future.delayed(Duration(seconds: 2));
  return '아이스 아메리카노';
}
// 주문 시작
// (2초 후)
// 주문 완료: 아이스 아메리카노
```

한 가지 차이: Dart에서 `async` 함수는 **첫 번째 `await`을 만나기 전까지 동기적으로 실행**된다. `await` 이전 코드는 바로 실행되고, `await`에서 비로소 비동기로 전환된다.

### 에러 처리

```dart
// try-catch (권장)
try {
  var data = await fetchData();
  print(data);
} catch (e) {
  print('에러: $e');
}

// then + catchError (콜백 스타일)
fetchData()
    .then((data) => print(data))
    .catchError((e) => print('에러: $e'))
    .whenComplete(() => print('완료'));
```

Swift의 `do-catch`/`try await`과 구조적으로 거의 같다. `then`/`catchError` 체이닝은 Swift의 옛날 completion handler 패턴이랑 비슷한 느낌인데, `async/await`이 훨씬 읽기 좋으니까 `try-catch` 쓰는 게 낫다.

### 타임아웃

```dart
try {
  var result = await fetchData().timeout(Duration(seconds: 3));
  print(result);
} on TimeoutException {
  print('3초 안에 응답이 없음');
}
```

### 순차 vs 병렬 실행

```dart
// 순차: 총 3초 (1+1+1)
var a = await fetchA();  // 1초
var b = await fetchB();  // 1초
var c = await fetchC();  // 1초

// 병렬: 총 1초 (동시 실행, 가장 긴 것 기준)
var results = await Future.wait([fetchA(), fetchB(), fetchC()]);
```

Swift에서는 `async let`으로 병렬 실행하고 tuple로 받는 반면, Dart는 `Future.wait`으로 `List`에 담아서 받는다.

```swift
// Swift 병렬 실행
async let a = fetchA()
async let b = fetchB()
let results = try await (a, b)  // tuple
```

서로 의존성이 없는 API 호출은 `Future.wait`으로 묶으면 체감 속도가 확 빨라진다.

## Stream - 연속된 비동기 데이터

`Future`가 한 번의 결과라면, `Stream`은 **여러 번 값을 받을 수 있는** 비동기 시퀀스다. [Dart 공식 가이드](https://dart.dev/libraries/async/creating-streams)에서 자세히 다루고 있다.

채팅 메시지, 센서 데이터, 실시간 주가처럼 시간에 따라 계속 들어오는 데이터를 처리할 때 쓴다.

### 만드는 방법 1: async* + yield

```dart
Stream<int> countDown(int from) async* {
  for (int i = from; i >= 0; i--) {
    await Future.delayed(Duration(seconds: 1));
    yield i;  // 값을 하나씩 내보냄
  }
}

// 사용
void main() async {
  await for (var count in countDown(5)) {
    print(count);  // 5, 4, 3, 2, 1, 0 (1초 간격)
  }
}
```

`async*`는 Stream을 생성하는 제너레이터 함수다. `yield`로 값을 하나씩 내보낸다. Swift의 `AsyncStream` + continuation 방식보다 훨씬 간결하다.

```swift
// Swift: continuation 기반이라 좀 장황함
let stream = AsyncStream<Int> { continuation in
    for i in stride(from: 5, through: 0, by: -1) {
        try? await Task.sleep(nanoseconds: 1_000_000_000)
        continuation.yield(i)
    }
    continuation.finish()
}

for await count in stream {
    print(count)
}
```

### 만드는 방법 2: StreamController

직접 이벤트를 push하고 싶을 때는 `StreamController`를 쓴다.

```dart
var controller = StreamController<String>();

// 구독
controller.stream.listen(
  (data) => print('받음: $data'),
  onError: (e) => print('에러: $e'),
  onDone: () => print('스트림 종료'),
);

// 이벤트 push
controller.add('첫 번째');
controller.add('두 번째');
controller.addError(Exception('문제 발생'));
controller.add('세 번째');
controller.close();
```

`StreamController`는 Flutter에서 위젯 간 데이터 전달이나 상태 변경 알림에도 많이 쓴다. `StreamBuilder` 위젯이랑 조합하면 실시간 UI 업데이트가 가능하다.

### Stream 변환

Stream도 `map`, `where` 같은 변환 메서드를 지원한다:

```dart
countDown(10)
    .where((n) => n.isEven)         // 짝수만
    .map((n) => '$n초 남았습니다')   // 문자열로 변환
    .listen((msg) => print(msg));
// 10초 남았습니다, 8초 남았습니다, ...
```

### BroadcastStream - 여러 리스너

기본 Stream은 **단일 구독**만 가능하다. 두 번 `listen`하면 에러가 난다. 여러 곳에서 동시에 구독하려면 **BroadcastStream**을 써야 한다.

```dart
// 방법 1: StreamController.broadcast()
var controller = StreamController<int>.broadcast();

controller.stream.listen((data) => print('A: $data'));
controller.stream.listen((data) => print('B: $data'));

controller.add(1);
// A: 1
// B: 1

// 방법 2: 기존 스트림을 변환
var broadcast = countDown(3).asBroadcastStream();
broadcast.listen((n) => print('위젯1: $n'));
broadcast.listen((n) => print('위젯2: $n'));
```

주의할 점: broadcast 스트림은 리스너가 없을 때 발생한 이벤트는 그냥 버린다. 늦게 구독하면 이전 데이터를 못 받는다.

Swift Combine으로 비교하면:

| Dart | Swift Combine | 설명 |
|------|-------------|------|
| `StreamController()` | - | 단일 구독 스트림 |
| `StreamController.broadcast()` | `PassthroughSubject` | 다중 구독, 이전 값 안 줌 |
| 직접 구현 필요 | `CurrentValueSubject` | 최신 값 유지 + 다중 구독 |

### yield* - 다른 스트림/이터러블 위임

`yield*`는 다른 스트림이나 이터러블의 모든 값을 그대로 내보낸다. 재귀적 스트림을 만들 때 유용하다.

```dart
Stream<String> countStream(int max) async* {
  for (int i = 1; i <= max; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield '$i';
  }
  yield '완료!';
  yield* countStream(max);  // 재귀: 다시 처음부터
}
```

`yield`가 값 하나를 내보내는 거라면, `yield*`는 통째로 위임하는 거다. `for` 루프로 하나씩 `yield`하는 것보다 효율적이다.

## Future vs Stream 정리

| | Future | Stream |
|--|--------|--------|
| 데이터 | 한 번 | 여러 번 |
| 사용 시점 | API 호출, 파일 읽기 | 실시간 데이터, 이벤트 |
| 소비 | `await` | `await for` 또는 `listen` |
| Swift 대응 | `async` 함수 반환값 | `AsyncStream` / Combine |
| 생성 | `async` 함수 | `async*` + `yield` 또는 `StreamController` |

Swift에서 넘어오면서 느낀 건, Dart의 `async*` + `yield`가 Swift의 `AsyncStream` continuation 방식보다 확실히 간결하다는 거다. 스트림을 만드는 코드 자체가 직관적이라 좋았다. 반면 `Future.wait` vs `async let` 같은 병렬 처리는 Swift 쪽이 tuple로 받으니까 타입이 더 명확한 느낌이다.
