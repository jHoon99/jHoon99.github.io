---
title: "Ch03-5. GoF 디자인 패턴 - Singleton, Factory, Builder, Command"
slug: "ch03-5-design-patterns"
date: 2026-04-28
draft: false
tags: ["Flutter", "Dart", "Design Patterns", "Singleton", "Factory", "Builder", "Command"]
categories: ["Flutter"]
description: "GoF 디자인 패턴 중 Singleton, Factory, Builder, Command를 Dart/Flutter 기준으로 정리한 기록"
---

강의에서 SOLID에 이어 GoF 디자인 패턴 4가지가 나왔다. GoF(Gang of Four)는 1994년에 나온 [Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns) 책에서 정리한 23개 객체지향 설계 패턴인데, 이번에 다룬 건 그 중 4개다. MVVM 같은 아키텍처 패턴과는 다르게, 클래스/객체 단위의 설계 기법이다.

## Singleton — 인스턴스 하나만

앱 전체에서 인스턴스가 딱 하나만 존재해야 할 때 쓴다. DB 커넥션, 네트워크 클라이언트, 로거 같은 것들이 대표적이다. 여러 개 만들면 리소스 낭비거나 상태가 꼬이는 경우에 필요하다.

```dart
class ApiClient {
  static final ApiClient _instance = ApiClient._();
  ApiClient._();  // 진짜 생성자는 private

  factory ApiClient() => _instance;  // 어디서 호출해도 같은 인스턴스
}

var a = ApiClient();
var b = ApiClient();
print(a == b);  // true — 같은 놈
```

`factory` 키워드가 포인트다. 일반 생성자는 호출할 때마다 새 인스턴스를 만드는데, `factory`는 기존 인스턴스를 돌려줄 수 있다. `ApiClient()` 처럼 생성자를 부르는 것 같지만 실제로는 `_instance`를 반환하는 거다.

Swift에서는 보통 `static let shared`로 싱글톤을 만든다:

```swift
// Swift
class ApiClient {
    static let shared = ApiClient()
    private init() {}
}

let client = ApiClient.shared
```

Dart는 `factory` 생성자 덕분에 `.shared` 없이 그냥 `ApiClient()`로 쓸 수 있다는 차이가 있다. 쓰는 쪽에서는 싱글톤인지 모르고 써도 되는 셈.

Flutter에서 GetX 쓰면 `Get.put()`/`Get.find()`가 사실상 싱글톤 관리를 해준다:

```dart
// GetX 방식 — 직접 싱글톤 안 만들어도 됨
Get.put(ApiClient());          // 등록
var client = Get.find<ApiClient>();  // 어디서든 같은 인스턴스
```

## Factory — 객체 생성을 위임

만드는 쪽이 **뭐가 나올지 결정**하는 패턴이다. 쓰는 쪽은 구체적인 클래스를 몰라도 된다.

### factory 생성자

Dart의 `factory` 키워드는 싱글톤 외에도 여러 용도로 쓸 수 있다.

```dart
// 1. fromJson — API 응답을 객체로 변환
class User {
  final String name;
  final int age;
  User(this.name, this.age);

  factory User.fromJson(Map<String, dynamic> json) {
    return User(json['name'], json['age']);
  }
}

var user = User.fromJson({'name': 'jHoon', 'age': 25});
```

`User.fromJson`이 Factory 패턴이다. JSON이라는 날것의 데이터를 받아서 알아서 `User` 객체를 만들어준다. API 연동하면 매번 쓰게 되는 패턴이다.

```dart
// 2. 조건에 따라 다른 서브클래스 반환
abstract class Payment {
  void pay(int amount);

  factory Payment(String method) {
    if (method == 'card') return CardPayment();
    if (method == 'cash') return CashPayment();
    throw 'unknown method';
  }
}

class CardPayment extends Payment {
  @override
  void pay(int amount) => print('카드 결제: $amount원');
}

class CashPayment extends Payment {
  @override
  void pay(int amount) => print('현금 결제: $amount원');
}

var payment = Payment('card');  // Payment로 불렀는데 CardPayment가 나옴
payment.pay(5000);  // 카드 결제: 5000원
```

`Payment('card')` 하면 `CardPayment`가 나오고, `Payment('cash')` 하면 `CashPayment`가 나온다. 쓰는 쪽은 `CardPayment` 클래스를 직접 알 필요가 없다.

사실 Dart SDK 자체가 이 패턴을 쓰고 있다. `List`가 abstract class인데 `List.generate()`나 `List.filled()`로 만들 수 있는 이유가 내부에 `factory` 생성자가 있기 때문이다.

### 앱에서 실제로 쓰는 빈도

앱 개발에서 `factory`를 직접 쓸 일은 대부분 **싱글톤**이랑 **fromJson** 두 가지다. 서브클래스 분기 같은 건 라이브러리나 프레임워크 만드는 사람이 쓰는 거라, "이런 게 있구나" 정도면 된다.

## Builder — 단계적으로 조립

파라미터가 많은 객체를 **하나씩 붙여가며 만드는** 패턴이다. 조립식 가구처럼.

```dart
// Builder 패턴이 필요한 상황 (Java 스타일)
class HttpRequest {
  String method;
  String url;
  Map<String, String> headers;
  int timeout;

  HttpRequest._(this.method, this.url, this.headers, this.timeout);
}

class HttpRequestBuilder {
  String _method = 'GET';
  String _url = '';
  Map<String, String> _headers = {};
  int _timeout = 30;

  HttpRequestBuilder setMethod(String m) { _method = m; return this; }
  HttpRequestBuilder setUrl(String u) { _url = u; return this; }
  HttpRequestBuilder addHeader(String k, String v) { _headers[k] = v; return this; }
  HttpRequestBuilder setTimeout(int t) { _timeout = t; return this; }

  HttpRequest build() => HttpRequest._(_method, _url, _headers, _timeout);
}

// 사용
var request = HttpRequestBuilder()
    .setMethod('POST')
    .setUrl('https://api.com/users')
    .addHeader('Authorization', 'Bearer abc')
    .setTimeout(10)
    .build();
```

근데 이거, Dart에서는 **named parameter**로 같은 걸 할 수 있다:

```dart
class HttpRequest {
  final String method;
  final String url;
  final Map<String, String> headers;
  final int timeout;

  HttpRequest({
    this.method = 'GET',
    required this.url,
    this.headers = const {},
    this.timeout = 30,
  });
}

var request = HttpRequest(
  method: 'POST',
  url: 'https://api.com/users',
  headers: {'Authorization': 'Bearer abc'},
  timeout: 10,
);
```

Java에는 named parameter가 없어서 Builder 클래스가 필수였는데, Dart/Swift는 언어 차원에서 지원하니까 Builder를 별도로 만들 필요가 거의 없다.

사실 Flutter의 Widget 생성자가 전부 이 방식이다:

```dart
// Flutter Widget = 사실상 Builder 패턴
AlertDialog(
  title: Text('삭제'),
  content: Text('정말 삭제할까?'),
  actions: [
    TextButton(onPressed: () {}, child: Text('취소')),
    TextButton(onPressed: () {}, child: Text('확인')),
  ],
);
```

named parameter로 하나씩 조립하는 거니까 Builder랑 본질은 같다. Dart에서 Builder 패턴을 직접 구현할 일은 거의 없다.

## Command — 동작을 객체로 감싸기

**실행할 동작 자체를 객체로 만들어서** 저장하거나 되돌리거나 할 수 있게 하는 패턴이다. 리모컨 버튼이라고 생각하면 된다. 버튼에 동작을 매핑해두고, 누른 기록을 저장하면 undo도 가능하다.

```dart
abstract class Command {
  void execute();
  void undo();
}

class AddTextCommand extends Command {
  final List<String> document;
  final String text;
  AddTextCommand(this.document, this.text);

  @override
  void execute() => document.add(text);

  @override
  void undo() => document.removeLast();
}

class DeleteTextCommand extends Command {
  final List<String> document;
  late String _deleted;
  DeleteTextCommand(this.document);

  @override
  void execute() {
    _deleted = document.removeLast();
  }

  @override
  void undo() => document.add(_deleted);
}

// 사용: 실행 이력을 스택으로 관리
var doc = <String>[];
var history = <Command>[];

// 텍스트 추가
var cmd1 = AddTextCommand(doc, '첫 번째 줄');
cmd1.execute();
history.add(cmd1);
// doc: ['첫 번째 줄']

var cmd2 = AddTextCommand(doc, '두 번째 줄');
cmd2.execute();
history.add(cmd2);
// doc: ['첫 번째 줄', '두 번째 줄']

// Ctrl+Z — 되돌리기
history.removeLast().undo();
// doc: ['첫 번째 줄']
```

메모장, 그림판 같은 앱에서 undo/redo가 이 패턴이다.

Flutter에서 `onPressed`에 함수를 넘기는 것도 넓게 보면 Command다:

```dart
ElevatedButton(
  onPressed: () => cart.addItem(product),  // 동작을 함수로 감싸서 전달
  child: Text('담기'),
)
```

다만 undo 기능이 필요 없으면 이렇게 콜백으로 충분하고, Command 클래스를 따로 만들 일은 거의 없다. 문서 편집기나 드로잉 앱처럼 실행 취소가 핵심인 앱에서나 정식으로 쓰는 패턴이다.

## 정리

| 패턴 | 한 줄 요약 | 앱에서 쓰는 빈도 |
|------|-----------|----------------|
| Singleton | 인스턴스 하나만 | 자주 — 서비스, DB, 네트워크 |
| Factory | 만드는 걸 위임 | 자주 — fromJson, 싱글톤 |
| Builder | 단계적 조립 | 거의 안 씀 — named parameter가 대신함 |
| Command | 동작을 객체로 | 거의 안 씀 — undo 필요할 때만 |

GoF 패턴 23개 중 4개만 다뤘는데, 앱 개발에서 자주 쓰는 건 Singleton이랑 Factory 정도다. Builder는 Dart 언어가 이미 해결해주고 있고, Command는 특수한 경우에만 필요하다. 나머지 GoF 패턴 중에는 Observer(Stream, ChangeNotifier)가 Flutter에서 가장 많이 쓰이는데 이건 상태관리랑 같이 정리하는 게 나을 것 같다.
