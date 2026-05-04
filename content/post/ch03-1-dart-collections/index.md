---
title: "Ch03-1. Dart 컬렉션과 제네릭"
slug: "ch03-1-dart-collections"
date: 2026-04-25
draft: false
tags: ["Flutter", "Dart", "List", "Map", "Set", "Generics", "Abstract Class"]
categories: ["Flutter"]
description: "Dart의 List, Map, Set 컬렉션과 제네릭, abstract class를 Swift와 비교하며 정리한 기록"
---

Ch03부터는 Dart 문법을 좀 더 깊게 판다. 첫 번째는 컬렉션과 제네릭이다.

## List - Swift의 Array

Dart의 `List`는 Swift의 `Array`와 거의 같다. 선언 방식도 비슷한데 몇 가지 차이가 있다.

```dart
var scores = [95, 87, 72, 64, 91];           // List<int> 추론
var names = <String>['Alice', 'Bob'];          // 타입 명시
var empty = <int>[];                           // 빈 리스트
var generated = List.generate(5, (i) => i * 2); // [0, 2, 4, 6, 8]
```

Swift에서는 `[Int]()`나 `Array(repeating:count:)` 같은 걸 쓰는데, Dart는 `List.generate`로 초기값을 만들 수 있어서 좀 더 유연하다.

### 주요 메서드

Swift에서 자주 쓰던 메서드들이 Dart에서는 이름이 다른 경우가 있다:

```dart
var items = [1, 2, 3, 4, 5, 6, 7, 8];

// where == Swift의 filter
var evens = items.where((n) => n.isEven).toList();  // [2, 4, 6, 8]

// map은 동일
var doubled = items.map((n) => n * 2).toList();  // [2, 4, 6, ...]

// expand == Swift의 flatMap
var nested = [[1, 2], [3, 4], [5]];
var flat = nested.expand((list) => list).toList();  // [1, 2, 3, 4, 5]

// any == Swift의 contains(where:)
var hasEven = items.any((n) => n.isEven);  // true

// every == Swift의 allSatisfy
var allPositive = items.every((n) => n > 0);  // true
```

여기서 중요한 게, `map`이나 `where`가 반환하는 건 `List`가 아니라 **`Iterable`**이다. 지연 평가(lazy)라서 `.toList()`를 호출해야 실제로 계산된다. Swift는 `map`/`filter`가 바로 `Array`를 반환하는 것과 다르다.

| Dart | Swift | 설명 |
|------|-------|------|
| `where()` | `filter()` | 조건 필터링 |
| `expand()` | `flatMap()` | 중첩 평탄화 |
| `any()` | `contains(where:)` | 하나라도 만족? |
| `every()` | `allSatisfy()` | 전부 만족? |
| `fold()` | `reduce(into:)` | 초기값 있는 축약 |
| `take(n)` | `prefix(n)` | 앞에서 n개 |
| `skip(n)` | `dropFirst(n)` | 앞에서 n개 건너뛰기 |

### reduce vs fold

`reduce`는 첫 번째 원소를 초기값으로 쓰고, `fold`는 초기값을 직접 지정한다. 빈 리스트에서 `reduce`를 쓰면 에러가 나니까 `fold`가 더 안전하다.

```dart
var prices = [1200, 3500, 890, 4200];

// reduce: 첫 원소가 초기값 → 빈 리스트면 에러
var total1 = prices.reduce((sum, p) => sum + p);  // 9790

// fold: 초기값 지정 → 빈 리스트도 안전
var total2 = prices.fold<int>(0, (sum, p) => sum + p);  // 9790
```

### 삽입, 교환, 정렬

```dart
var list = [1, 2, 3, 4, 5];

list.insert(1, 99);          // [1, 99, 2, 3, 4, 5]
list.removeAt(3);             // 인덱스 3 제거

// 교환은 직접 구현해야 함
var temp = list[0];
list[0] = list[2];
list[2] = temp;

// extension으로 만들면 편하다
extension ListSwap<T> on List<T> {
  void swap(int i, int j) {
    final temp = this[i];
    this[i] = this[j];
    this[j] = temp;
  }
}

list.swap(0, 2);  // 깔끔
```

Swift에서는 `swapAt()`이 기본 제공되는데, Dart는 없어서 extension으로 만들어야 한다.

### Spread 연산자와 Collection if/for

Dart만의 기능인데, 꽤 편하다.

```dart
var defaults = ['en', 'ko'];
var userLangs = ['ja'];
var allLangs = [...defaults, ...userLangs, 'zh'];
// ['en', 'ko', 'ja', 'zh']

// null-aware spread
List<String>? extras;
var combined = [...defaults, ...?extras];  // null이면 무시

// collection if: 조건부 원소 추가
var isAdmin = true;
var menu = [
  'Home',
  'Profile',
  if (isAdmin) 'Admin Panel',
];

// collection for: 반복으로 원소 생성
var numbers = [1, 2, 3, 4, 5];
var labels = [
  for (var n in numbers)
    if (n.isEven) '$n은 짝수',
];
// ['2은 짝수', '4은 짝수']
```

Swift에서는 이런 걸 하려면 `filter` + `map`을 체이닝하거나 따로 로직을 짜야 하는데, Dart는 리터럴 안에서 바로 된다. 처음 봤을 때 꽤 신기했음.

## Map - Swift의 Dictionary

```dart
var headers = {
  'Content-Type': 'application/json',
  'Authorization': 'Bearer abc123',
};  // Map<String, String>

// 접근 (nullable 반환 — Swift와 동일)
var type = headers['Content-Type'];  // String?
headers['Cache-Control'] = 'no-cache';  // 추가/수정
```

Swift의 `Dictionary`와 거의 같은데, 유용한 메서드 몇 가지:

```dart
var wordCount = <String, int>{};
var words = ['dart', 'flutter', 'dart', 'widget', 'dart'];

// update: 있으면 업데이트, 없으면 생성
for (var word in words) {
  wordCount.update(word, (count) => count + 1, ifAbsent: () => 1);
}
// {'dart': 3, 'flutter': 1, 'widget': 1}

// putIfAbsent: 없을 때만 생성
var cache = <String, List<String>>{};
cache.putIfAbsent('users', () => []).add('Alice');
cache.putIfAbsent('users', () => []).add('Bob');
// {'users': ['Alice', 'Bob']}

// containsKey
if (wordCount.containsKey('dart')) {
  print('dart가 ${wordCount['dart']}번 나옴');
}
```

Swift에서는 `cache["users", default: []].append("Alice")` 이런 식으로 default subscript를 쓰는데, Dart는 `putIfAbsent`로 처리한다.

## Set - 중복 없는 컬렉션

```dart
var fruits = {'apple', 'banana', 'apple', 'grape'};
// {'apple', 'banana', 'grape'} — 중복 제거

fruits.add('mango');
fruits.contains('apple');  // true

// List에서 중복 제거할 때 유용
var list = [1, 2, 2, 3, 3, 3];
var unique = list.toSet().toList();  // [1, 2, 3]
```

Swift의 `Set`과 동일하다. 교집합, 합집합도 `intersection()`, `union()`으로 가능하다.

## 제네릭

Swift의 제네릭과 문법이 거의 같다. `<T>` 대신 제약을 걸 때 Swift는 `<T: Protocol>`이고 Dart는 `<T extends Class>` 이 차이 정도.

```dart
// Dart 3의 sealed class로 제네릭 Result 패턴
sealed class Result<T> {
  const Result();
}

class Success<T> extends Result<T> {
  final T data;
  const Success(this.data);
}

class Failure<T> extends Result<T> {
  final String message;
  const Failure(this.message);
}

// 사용
Result<int> fetchScore() {
  var ok = true;
  return ok ? Success(95) : Failure('서버 에러');
}

// switch로 분기 — sealed라서 빠뜨리면 컴파일 경고
var result = fetchScore();
switch (result) {
  case Success(:final data):
    print('점수: $data');
  case Failure(:final message):
    print('실패: $message');
}
```

Swift의 `enum`+`associated value` 패턴이랑 비슷한데, Dart는 `sealed class` + 서브클래스로 구현한다. Dart 3 이전에는 `data = null as T` 같은 꼼수를 썼는데, null safety에서 터지니까 이제는 sealed가 정석이다.

### 타입 제약

```dart
// T는 Comparable을 구현해야 함
class SortedList<T extends Comparable<T>> {
  final _items = <T>[];

  void add(T item) {
    _items.add(item);
    _items.sort();
  }

  T get first => _items.first;
  T get last => _items.last;
}

var sorted = SortedList<int>();
sorted.add(42);
sorted.add(7);
sorted.add(23);
print(sorted.first);  // 7
```

Swift에서 `<T: Comparable>` 쓰는 것과 같은 패턴이다. 키워드만 `extends`로 다르다.

## Abstract Class - Swift의 Protocol

Dart에는 `protocol` 키워드가 없다. 대신 `abstract class`가 그 역할을 한다. 그리고 Dart의 모든 클래스는 **암묵적 인터페이스**를 정의한다.

```dart
// abstract class = Swift의 protocol + default implementation
abstract class Animal {
  String get name;        // 추상 getter (구현 강제)
  void eat();             // 추상 메서드 (구현 강제)

  void breathe() {        // 기본 구현 (Swift의 protocol extension과 비슷)
    print('$name이(가) 숨을 쉰다');
  }
}

class Dog extends Animal {
  @override
  String get name => '강아지';

  @override
  void eat() => print('$name이(가) 사료를 먹는다');
}

class Cat extends Animal {
  @override
  String get name => '고양이';

  @override
  void eat() => print('$name이(가) 참치를 먹는다');
}
```

### extends vs implements

```dart
// extends: 구현을 상속받음 (단일 상속)
class Dog extends Animal { ... }

// implements: 인터페이스만 가져옴 (모든 걸 직접 구현해야 함, 다중 가능)
class Robot implements Animal {
  @override
  String get name => '로봇';

  @override
  void eat() {}          // 전부 직접 구현

  @override
  void breathe() {}      // 기본 구현도 안 물려받음
}
```

Swift에서는 `protocol` 채택하면 required만 구현하면 되는데, Dart의 `implements`는 **모든 멤버를 직접 구현**해야 한다. 기본 구현을 물려받고 싶으면 `extends`를 써야 한다.

| Dart | Swift | 설명 |
|------|-------|------|
| `abstract class` | `protocol` | 추상 타입 정의 |
| `extends` | `:` (class 상속) | 구현 상속, 단일만 가능 |
| `implements` | `:` (protocol 채택) | 인터페이스만, 다중 가능 |
| `mixin` + `with` | protocol extension | 수평적 코드 재사용 |

## 정리

Swift에서 넘어오면서 제일 헷갈렸던 건 `where`/`expand` 같은 메서드 이름 차이랑, `map`/`where`가 lazy `Iterable`을 반환한다는 점이다. `.toList()` 안 붙여서 삽질한 적이 있다. 반면 spread 연산자랑 collection if/for는 Swift에 없는 기능인데 익숙해지면 꽤 편하다.
