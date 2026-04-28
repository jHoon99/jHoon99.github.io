---
title: "Ch03-3. Dart 함수형 프로그래밍"
slug: "ch03-3-dart-functional"
date: 2026-04-27
draft: false
tags: ["Flutter", "Dart", "Lambda", "Iterable", "Functional Programming", "Generator"]
categories: ["Flutter"]
description: "Dart의 람다식, Iterable과 yield, 절차형 vs 함수형 프로그래밍을 Swift와 비교하며 정리한 기록"
---

Ch03 마지막은 함수형 프로그래밍이다. 람다, Iterable/yield, 절차형과 함수형의 차이를 정리한다.

## 람다식 (익명 함수)

Dart에서 함수는 **1급 객체**다. 변수에 담을 수 있고, 인자로 넘길 수 있고, 반환할 수도 있다. Swift의 클로저와 같은 개념이다.

```dart
// 일반 함수
int add(int a, int b) => a + b;

// 익명 함수를 변수에 담기
var multiply = (int a, int b) => a * b;

// 블록 바디 (여러 줄)
var greet = (String name) {
  var message = 'Hello, $name';
  return message;
};
```

Swift 클로저와 비교하면 문법이 좀 다르다:

```swift
// Swift
let add = { (a: Int, b: Int) -> Int in a + b }
let multiply: (Int, Int) -> Int = { $0 * $1 }
```

```dart
// Dart
var add = (int a, int b) => a + b;
// $0, $1 같은 축약은 없음
```

Swift에서는 `$0`, `$1`로 파라미터를 축약할 수 있는데, Dart는 그런 문법이 없다. 대신 화살표 `=>`가 있어서 한 줄짜리 함수를 간결하게 쓸 수 있다.

### 정렬에서의 활용

```dart
var list = [5, 2, 4, 1, 3];

// 오름차순
list.sort((a, b) => a.compareTo(b));

// 내림차순
list.sort((a, b) => b.compareTo(a));
```

Swift에서 `sorted(by: <)` 쓰는 것과 비슷한데, Dart는 `compareTo`를 쓰는 게 관례다. `sort`는 원본을 변경하는 것도 주의 (Swift의 `sort()`도 마찬가지).

### 함수를 인자로 넘기기

```dart
// 함수를 파라미터로 받는 함수
void repeat(int times, void Function(int) action) {
  for (var i = 0; i < times; i++) {
    action(i);
  }
}

repeat(3, (i) => print('$i번째 실행'));
// 0번째 실행
// 1번째 실행
// 2번째 실행
```

Dart에서 함수 타입은 `void Function(int)` 식으로 쓴다. Swift의 `(Int) -> Void`와 같은 의미인데 문법이 다르다.

### Tear-off

함수를 괄호 없이 이름만 쓰면 자동으로 클로저가 된다. [Effective Dart](https://dart.dev/effective-dart/usage)에서도 람다로 감싸지 말고 tear-off를 쓰라고 권장한다.

```dart
// 이것보다
items.forEach((item) => print(item));

// 이렇게 (tear-off)
items.forEach(print);
```

### typedef로 함수 타입에 이름 붙이기

함수 타입이 길어지면 `typedef`로 별칭을 만들 수 있다. Swift의 `typealias`와 같다.

```dart
typedef Predicate<T> = bool Function(T item);
typedef Mapper<T, R> = R Function(T item);

// 사용
Predicate<int> isEven = (n) => n % 2 == 0;
Mapper<String, int> getLength = (s) => s.length;

var evenNumbers = [1, 2, 3, 4, 5].where(isEven).toList();  // [2, 4]
```

### 함수를 반환하는 함수

```dart
double Function(double) makeDiscounter(double percent) {
  return (price) => price * (1 - percent / 100);
}

var tenOff = makeDiscounter(10);
var twentyOff = makeDiscounter(20);

print(tenOff(10000));   // 9000.0
print(twentyOff(10000)); // 8000.0
```

클로저가 `percent`를 캡처해서 기억하는 거다. Swift에서도 완전히 같은 패턴이 가능하다.

## Iterable과 yield

### Iterable이란

`Iterable`은 순차적으로 접근할 수 있는 컬렉션의 추상 타입이다. `List`와 `Set`이 `Iterable`을 구현하고 있다. [Dart 공식 codelab](https://dart.dev/codelabs/iterables)에서 자세히 설명하고 있다.

중요한 건 **지연 평가(lazy evaluation)**다. `map()`이나 `where()` 같은 메서드가 반환하는 `Iterable`은 바로 계산되지 않고, 실제로 값을 꺼낼 때 계산된다.

```dart
var mapped = [1, 2, 3].map((n) {
  print('변환 중: $n');
  return n * 2;
});
// 여기까지는 아무것도 출력 안 됨!

print(mapped.toList());  // 이때 비로소 "변환 중" 출력
```

Swift에서는 `map`/`filter`가 바로 `Array`를 반환한다(eager). Dart처럼 lazy로 쓰려면 `.lazy.map { ... }`으로 명시해야 한다. Dart는 기본이 lazy인 셈.

### sync* 제너레이터와 yield

`sync*`로 `Iterable`을 직접 만들 수 있다. 값을 하나씩 `yield`로 내보내는 제너레이터 함수다.

```dart
Iterable<int> range(int start, int end) sync* {
  for (int i = start; i <= end; i++) {
    yield i;
  }
}

for (var n in range(1, 5)) {
  print(n);  // 1, 2, 3, 4, 5
}
```

Swift에서 같은 걸 하려면 `Sequence`/`IteratorProtocol`을 직접 구현해야 하는데, Dart는 `sync*` + `yield` 두 키워드로 끝난다. 확실히 간결하다.

```swift
// Swift: 같은 걸 하려면 이만큼 써야 함
struct Range: Sequence, IteratorProtocol {
    var current: Int
    let end: Int
    mutating func next() -> Int? {
        guard current <= end else { return nil }
        defer { current += 1 }
        return current
    }
}
```

### yield* - 위임

`yield*`는 다른 Iterable이나 Stream의 모든 원소를 그대로 내보낸다.

```dart
Iterable<int> countdown(int from) sync* {
  if (from >= 0) {
    yield from;
    yield* countdown(from - 1);  // 재귀 위임
  }
}

print(countdown(5).toList());  // [5, 4, 3, 2, 1, 0]
```

`yield*` 없이 하려면 `for (var v in countdown(from - 1)) yield v;` 이렇게 루프를 돌려야 하는데, `yield*`가 이걸 한 줄로 해준다. 재귀적 구조를 다룰 때 편하다.

### sync* vs async*

| | sync* | async* |
|--|-------|--------|
| 반환 타입 | `Iterable<T>` | `Stream<T>` |
| 실행 | 동기 | 비동기 |
| `await` 사용 | 불가 | 가능 |
| 소비 | `for-in` | `await for` 또는 `listen` |

`sync*`는 동기 제너레이터, `async*`는 비동기 제너레이터다. 둘 다 `yield`로 값을 내보내는 건 같은데, `async*`는 비동기 작업(네트워크, 타이머 등)을 사이에 끼울 수 있다.

## 절차형 vs 함수형

같은 문제를 두 가지 방식으로 풀 수 있다.

**문제**: 상품 목록에서 재고가 있는 것만 골라서 10% 할인 적용한 총액을 구하기.

```dart
class Product {
  final String name;
  final int price;
  final bool inStock;
  const Product(this.name, this.price, this.inStock);
}

var products = [
  Product('노트북', 1500000, true),
  Product('마우스', 35000, true),
  Product('키보드', 89000, false),
  Product('모니터', 450000, true),
];
```

**절차형 (명령형)**: 어떻게 할지를 단계별로 지시

```dart
var total = 0;
for (var p in products) {
  if (p.inStock) {
    total += (p.price * 0.9).round();
  }
}
print(total);  // 1786500
```

**함수형 (선언형)**: 무엇을 할지를 선언

```dart
var total = products
    .where((p) => p.inStock)
    .map((p) => (p.price * 0.9).round())
    .reduce((sum, price) => sum + price);
print(total);  // 1786500
```

같은 결과지만 접근 방식이 다르다:

| | 절차형 | 함수형 |
|--|--------|--------|
| 상태 | 변수를 만들고 값을 바꿈 | 데이터를 변환해서 새로 만듦 |
| 흐름 | for, if로 직접 제어 | where, map, reduce로 선언 |
| 부수 효과 | 있을 수 있음 | 최소화 |
| 가독성 | 한 줄씩 따라가야 함 | 의도가 한눈에 보임 |

Dart는 멀티 패러다임 언어라 둘 다 자유롭게 쓸 수 있다. [Effective Dart](https://dart.dev/effective-dart/usage)에서는 **변환 작업에는 함수형**(map/where/reduce 체이닝), **부수 효과가 있는 작업에는 for-in 루프**를 권장한다.

```dart
// 변환 → 함수형
var names = users.map((u) => u.name).toList();

// 부수 효과(출력, 상태 변경) → for-in
for (var user in users) {
  print(user.name);
  sendNotification(user);
}
```

### 메서드 체이닝

함수형 스타일의 장점은 체이닝으로 드러난다:

```dart
// CSV 데이터 처리
var activeUsers = rawLines
    .skip(1)                                // 헤더 건너뛰기
    .map((line) => line.split(','))         // 필드 분리
    .where((fields) => fields.length >= 3)  // 유효성 검사
    .where((fields) => fields[2] == 'active') // 활성 유저만
    .map((fields) => fields[0])             // 이름만 추출
    .toList();
```

Dart에서 체이닝이 편한 이유가 `where`/`map`이 기본 lazy라서 중간에 불필요한 리스트를 만들지 않기 때문이다. Swift에서는 `.lazy`를 안 붙이면 각 단계마다 새 `Array`가 생긴다.

### 캐스케이드 연산자 (..)

Dart만의 문법인데, 같은 객체에 여러 메서드를 연속 호출할 때 쓴다. 메서드 체이닝이랑은 다른 개념이다:

```dart
// 캐스케이드 없이
var paint = Paint();
paint.color = Colors.black;
paint.strokeWidth = 5.0;
paint.strokeCap = StrokeCap.round;

// 캐스케이드로 축약
var paint = Paint()
  ..color = Colors.black
  ..strokeWidth = 5.0
  ..strokeCap = StrokeCap.round;
```

`..`은 메서드의 반환값을 무시하고 **원래 객체를 반환**한다. 메서드 체이닝은 각 메서드가 새 값을 반환해야 하지만, 캐스케이드는 void 반환 메서드에도 쓸 수 있다. Swift에는 이런 문법이 없다.

## 정리

| Dart | Swift | 설명 |
|------|-------|------|
| `(x) => x * 2` | `{ $0 * 2 }` | 익명 함수 |
| `void Function(int)` | `(Int) -> Void` | 함수 타입 |
| `typedef` | `typealias` | 함수 타입 별칭 |
| `where()` | `filter()` | 필터링 |
| `expand()` | `flatMap()` | 평탄화 |
| `fold()` | `reduce(into:)` | 초기값 있는 축약 |
| `sync*` + `yield` | `Sequence` 프로토콜 구현 | 제너레이터 |
| `..` (cascade) | 없음 | 같은 객체에 연속 호출 |
| lazy 기본 | `.lazy` 명시 필요 | 지연 평가 |

Swift에서 넘어오면서 제일 인상적이었던 건 `sync*`/`async*` 제너레이터 문법이다. Swift에서 `Sequence` 만들려면 struct에 `IteratorProtocol` 구현하고 `next()` 메서드 만들고... 하는 게 Dart에서는 `yield` 한 줄이면 끝난다. 그리고 Iterable이 기본 lazy라는 것도 Swift와의 큰 차이점인데, 대량 데이터 처리할 때는 이게 유리하다.
