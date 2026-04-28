---
title: "Ch03-4. SOLID, DRY, KISS - 설계 원칙 정리"
slug: "ch03-4-solid-dry-kiss"
date: 2026-04-28
weight: 2
draft: false
tags: ["Flutter", "Dart", "SOLID", "DRY", "KISS", "Design Principles"]
categories: ["Flutter"]
description: "SOLID 원칙, DRY, KISS를 Dart/Flutter 코드 예제로 정리한 기록"
---

강의에서 SOLID, DRY, KISS 원칙이 나왔다. 코드를 따로 치진 않았는데 개념 정리는 해둬야 할 것 같아서 Dart/Flutter 기준으로 정리한다.

## SOLID 원칙

Robert C. Martin(Uncle Bob)이 정리한 객체지향 설계 5원칙이다. 원래 Java 세계에서 나온 건데 Dart/Flutter에서도 똑같이 적용된다.

### S — 단일 책임 원칙 (Single Responsibility)

> 클래스는 하나의 책임만 가져야 한다.

하나의 클래스가 여러 일을 하면 한쪽을 고칠 때 다른 쪽이 깨질 수 있다.

```dart
// 나쁜 예: 한 클래스가 데이터 로딩 + UI 포맷팅 + 저장까지 다 함
class UserManager {
  Future<User> fetchUser(int id) async { ... }
  String formatUserName(User user) => '${user.lastName} ${user.firstName}';
  Future<void> saveToLocal(User user) async { ... }
}

// 좋은 예: 책임을 나눔
class UserRepository {
  Future<User> fetch(int id) async { ... }
  Future<void> save(User user) async { ... }
}

class UserFormatter {
  String formatName(User user) => '${user.lastName} ${user.firstName}';
}
```

Flutter에서 이게 특히 중요한 게, Widget 안에 비즈니스 로직을 때려박으면 나중에 유지보수가 지옥이다. 그래서 GetX든 BLoC든 상태관리 패턴이 전부 UI와 로직을 분리하는 구조다.

### O — 개방-폐쇄 원칙 (Open-Closed)

> 확장에는 열려 있고, 수정에는 닫혀 있어야 한다.

기존 코드를 건드리지 않고 새 기능을 추가할 수 있어야 한다는 뜻이다.

```dart
// 나쁜 예: 새 할인 타입 추가하면 이 함수를 계속 수정해야 함
double calcDiscount(String type, double price) {
  if (type == 'percent') return price * 0.1;
  if (type == 'fixed') return 1000;
  // 새 타입 추가할 때마다 여기를 고침...
  return 0;
}

// 좋은 예: 새 할인은 클래스만 추가하면 됨
abstract class Discount {
  double calculate(double price);
}

class PercentDiscount extends Discount {
  final double rate;
  PercentDiscount(this.rate);

  @override
  double calculate(double price) => price * rate;
}

class FixedDiscount extends Discount {
  final double amount;
  FixedDiscount(this.amount);

  @override
  double calculate(double price) => amount;
}

// 새로 추가해도 기존 코드 안 건드림
class BuyOneGetOneFree extends Discount {
  @override
  double calculate(double price) => price;
}
```

Swift에서 `protocol` + 구현체 패턴으로 하는 것과 완전히 같다. Dart는 `abstract class`가 그 역할을 한다.

### L — 리스코프 치환 원칙 (Liskov Substitution)

> 부모 타입 자리에 자식 타입을 넣어도 정상 동작해야 한다.

이름이 어려운데 실제로는 단순하다. 상속받은 클래스가 부모의 계약을 깨면 안 된다는 거다.

```dart
class Rectangle {
  double width;
  double height;
  Rectangle(this.width, this.height);

  double get area => width * height;
}

// 나쁜 예: 정사각형이 직사각형을 상속
class Square extends Rectangle {
  Square(double size) : super(size, size);

  // width를 바꾸면 height도 바뀌어야 하는데...
  // Rectangle을 기대하는 코드에서 예상과 다르게 동작함
  @override
  set width(double value) {
    super.width = value;
    super.height = value;  // 부모의 계약 위반
  }
}

void printArea(Rectangle r) {
  r.width = 5;
  r.height = 3;
  print(r.area);  // Rectangle이면 15인데, Square면 9가 나옴
}
```

유명한 "정사각형-직사각형 문제"다. 수학적으로는 정사각형이 직사각형의 하위지만, 코드에서는 상속하면 안 되는 케이스다. 이럴 때는 상속 대신 공통 인터페이스를 만드는 게 맞다.

### I — 인터페이스 분리 원칙 (Interface Segregation)

> 클라이언트가 쓰지 않는 메서드에 의존하지 않아야 한다.

거대한 인터페이스 하나보다 작은 인터페이스 여러 개가 낫다.

```dart
// 나쁜 예: 모든 걸 하나에 때려넣음
abstract class Worker {
  void code();
  void design();
  void managePeople();
  void writeTests();
}

// Developer가 design()이나 managePeople()을 왜 구현해야 하지?

// 좋은 예: 역할별로 분리
abstract class Coder {
  void code();
  void writeTests();
}

abstract class Designer {
  void design();
}

abstract class Manager {
  void managePeople();
}

class Developer implements Coder {
  @override
  void code() => print('코딩 중');
  @override
  void writeTests() => print('테스트 작성 중');
}
```

Dart에서는 `implements`로 여러 인터페이스를 동시에 구현할 수 있으니까 분리해도 조합이 자유롭다. Swift의 protocol composition(`Coder & Manager`)이랑 같은 느낌이다.

### D — 의존성 역전 원칙 (Dependency Inversion)

> 구체 클래스가 아니라 추상에 의존해야 한다.

실전에서 제일 체감 큰 원칙이다. 테스트할 때 mock으로 교체하려면 이게 필수다.

```dart
// 나쁜 예: 구체 클래스에 직접 의존
class UserViewModel {
  final ApiService _api = ApiService();  // 이러면 테스트 때 교체 불가

  Future<void> loadUser() async {
    var user = await _api.fetchUser();
  }
}

// 좋은 예: 추상에 의존 + 생성자 주입
abstract class UserRepository {
  Future<User> fetchUser();
}

class ApiUserRepository implements UserRepository {
  @override
  Future<User> fetchUser() async {
    // 실제 API 호출
  }
}

class UserViewModel {
  final UserRepository _repo;  // 추상 타입에 의존
  UserViewModel(this._repo);   // 외부에서 주입

  Future<void> loadUser() async {
    var user = await _repo.fetchUser();
  }
}

// 실제 사용
var vm = UserViewModel(ApiUserRepository());

// 테스트
var vm = UserViewModel(MockUserRepository());
```

Swift에서 protocol로 DI(의존성 주입) 하는 것과 완전히 같은 패턴이다. Flutter에서 GetX의 `Get.put()`, Provider의 `ChangeNotifierProvider` 같은 것들이 전부 이 원칙 위에서 돌아간다.

## DRY — Don't Repeat Yourself

> 같은 로직을 두 번 이상 쓰지 마라.

복붙이 3번 이상 되면 함수나 클래스로 빼라는 거다.

```dart
// 나쁜 예: 같은 검증 로직이 여기저기 흩어져 있음
void createUser(String email) {
  if (!email.contains('@') || email.length < 5) throw '이메일 형식 오류';
  // ...
}

void updateEmail(String email) {
  if (!email.contains('@') || email.length < 5) throw '이메일 형식 오류';
  // ...
}

// 좋은 예: 한 곳에서 관리
class EmailValidator {
  static void validate(String email) {
    if (!email.contains('@') || email.length < 5) throw '이메일 형식 오류';
  }
}

void createUser(String email) {
  EmailValidator.validate(email);
}
```

다만 주의할 점이 있다. "코드가 비슷하게 생겼다"고 무조건 합치면 안 된다. 지금은 같아 보여도 나중에 다르게 변할 수 있는 로직이면 분리해두는 게 맞다. 억지로 합치면 오히려 조건 분기가 늘어나서 더 복잡해진다.

## KISS — Keep It Simple, Stupid

> 단순하게 해라.

과도한 추상화, 쓸데없는 패턴 적용을 경계하라는 거다.

```dart
// KISS 위반: 간단한 걸 과하게 추상화
abstract class StringProcessor {
  String process(String input);
}

class UpperCaseProcessor implements StringProcessor {
  @override
  String process(String input) => input.toUpperCase();
}

class ProcessorFactory {
  static StringProcessor create(String type) {
    switch (type) {
      case 'upper': return UpperCaseProcessor();
      default: throw 'Unknown type';
    }
  }
}

var result = ProcessorFactory.create('upper').process('hello');

// KISS: 그냥 이렇게 하면 되는데?
var result = 'hello'.toUpperCase();
```

SOLID을 배우면 뭐든 인터페이스로 빼고 패턴을 적용하고 싶어지는데, 간단한 문제에 복잡한 구조를 씌우면 오히려 코드가 읽기 어려워진다. "지금 이 추상화가 정말 필요한가?"를 항상 생각해야 한다.

## 정리

| 원칙 | 한 줄 요약 | Flutter에서 체감 |
|------|-----------|----------------|
| SRP | 한 클래스 = 한 책임 | Widget, ViewModel, Repository 분리 |
| OCP | 수정 없이 확장 | abstract class로 다형성 |
| LSP | 자식이 부모 계약 지키기 | 상속 설계 시 주의 |
| ISP | 인터페이스 잘게 쪼개기 | implements 다중 구현 |
| DIP | 추상에 의존, 주입 받기 | GetX/Provider로 DI |
| DRY | 반복 금지 | 공통 로직 유틸로 분리 |
| KISS | 단순하게 | 과도한 패턴 적용 금지 |

SOLID이 결국 말하는 건 "변경에 강한 코드를 짜라"는 거다. 그리고 DRY랑 KISS는 서로 균형을 잡아줘야 한다 — 중복을 없애되(DRY) 그 과정에서 너무 복잡해지면 안 되고(KISS). 실무에서 이 밸런스 잡는 게 제일 어렵다고 한다.
