---
title: "Ch05-2. Todo 앱 GetX로 전환 - .obs와 Obx의 마법"
slug: "ch05-2-getx-state-management"
date: 2026-04-30
weight: 5
draft: false
tags: ["Flutter", "Dart", "State Management", "GetX", "Obx", "RxList", "Todo App"]
categories: ["Flutter"]
description: "ValueNotifier로 만든 Todo 앱을 GetX로 전환하면서 .obs 반응형 시스템의 편리함과 Static Model의 트레이드오프를 정리한 기록"
---

Ch05-1에서 ValueNotifier + InheritedWidget으로 만든 Todo 앱을 GetX로 전환한다. Ch04에서 다뤘던 Static Model의 실전 적용 버전이다. 결론부터 말하면, 일일이 업데이트 걸어줄 필요 없이 심플하게 상태관리가 가능해진다. 근데 그 대가가 있다.

## 뭐가 바뀌나

바꾸는 곳은 정해져 있다. 상태 클래스, 등록, 구독, 접근. 이 4가지만 바꾸면 된다.

| 영역 | Before (ValueNotifier) | After (GetX) |
|------|----------------------|-------------|
| 상태 클래스 | `ValueNotifier` 상속 | `GetxController` 상속 |
| 상태 변수 | `List<Todo>` + `notifyListeners()` | `RxList<Todo>` (`.obs`) |
| UI 구독 | `ValueListenableBuilder` | `Obx(() =>)` |
| 등록 | `InheritedWidget` 직접 구현 | `Get.put()` |
| 접근 | `context.dependOn...` | `Get.find()` |

## .obs — 자동 변경 감지

### 변환 핵심

```dart
// Before: ValueNotifier
class TodoDataNotifier extends ValueNotifier<List<Todo>> {
  TodoDataNotifier() : super([]);

  void addTodo(Todo todo) {
    value.add(todo);
    notifyListeners();  // 수동 호출 필수!
  }
}

// After: GetX
class TodoDataHolder extends GetxController {
  final RxList<Todo> todoList = <Todo>[].obs;

  void addTodo(Todo todo) {
    todoList.add(todo);
    // notifyListeners 불필요! .obs가 변경을 자동 감지
  }
}
```

핵심 차이는 `.obs`다. `.obs`를 붙이면 `Rx<T>` 타입으로 래핑되면서, `add`, `remove`, `[]=` 같은 List 조작을 전부 intercept한다. 변경이 생기면 자동으로 구독자에게 알림을 보내기 때문에 ValueNotifier에서 `notifyListeners()` 까먹어서 UI가 안 바뀌는 버그가 사라진다.

### 다양한 타입에 .obs

```dart
final count = 0.obs;           // RxInt
final name = ''.obs;           // RxString
final items = <Todo>[].obs;    // RxList<Todo>
final user = User().obs;       // Rx<User>
```

기본 타입은 `.value`로 접근하고, List는 직접 메서드를 호출할 수 있다. Swift에서 `@Published`가 프로퍼티 단위로 감시하는 것과 비슷한데, `.obs`는 변수 단위로 감시한다.

### 같은 값이면 리빌드 안 함

```dart
final count = 0.obs;
count.value = 0;  // 같은 값 → Obx 리빌드 안 함
count.value = 1;  // 다른 값 → Obx 리빌드
```

ValueNotifier의 `notifyListeners()`는 값이 같아도 무조건 알림을 보내는데, GetX는 내부적으로 이전 값과 비교해서 불필요한 리빌드를 막아준다.

## Obx — 반응형 UI 빌더

```dart
// Before: ValueListenableBuilder
ValueListenableBuilder<List<Todo>>(
  valueListenable: todoNotifier,
  builder: (context, todoList, child) {
    return Column(
      children: todoList.map((e) => TodoItem(e)).toList(),
    );
  },
)

// After: Obx
Obx(() => Column(
  children: todoDataHolder.todoList.map((e) => TodoItem(e)).toList(),
))
```

`Obx`가 내부에서 어떤 `.obs` 변수를 읽었는지 자동으로 추적한다. 해당 변수가 바뀌면 `Obx` 블록만 리빌드된다. SwiftUI에서 `@Published` 프로퍼티가 바뀌면 `body`가 리빌드되는 것과 같은 원리인데, `Obx` 단위로 범위를 좁힐 수 있어서 더 세밀한 제어가 가능하다.

코드량으로만 보면 `ValueListenableBuilder` 대비 거의 반으로 줄었다. builder 패턴의 보일러플레이트가 사라진 게 크다.

## InheritedWidget 제거 — Get.put / Get.find

### 등록

```dart
// Before: InheritedWidget으로 위젯 트리 감싸기
TodoDataHolder(
  notifier: TodoDataNotifier(),
  child: MaterialApp(
    home: const MainScreen(),
  ),
)

// After: Get.put 한 줄
@override
void initState() {
  super.initState();
  Get.put(TodoDataHolder());
}
```

InheritedWidget 클래스를 만들고, 위젯 트리에 감싸고, `of(context)` 패턴을 구현하는 보일러플레이트가 전부 사라진다. `Get.put()` 한 줄이면 끝이다.

### 접근

```dart
// Before: context 필요
context.todoHolder.addTodo(todo);

// After: context 불필요
Get.find<TodoDataHolder>().addTodo(todo);
```

`Get.find()`로 어디서든 접근할 수 있다. context가 필요 없으니까 StatelessWidget이든, 일반 클래스든, 어디서든 상태에 접근 가능하다.

mixin으로 더 축약할 수도 있다:

```dart
mixin TodoDataProvider {
  TodoDataHolder get todoData => Get.find<TodoDataHolder>();
}

// 사용
class SomeWidget extends StatelessWidget with TodoDataProvider {
  @override
  Widget build(BuildContext context) {
    todoData.addTodo(todo);  // 깔끔
  }
}
```

## 상태 변경 로직 — mutable의 함정

### 추가, 삭제

```dart
void addTodo(Todo todo) {
  todoList.add(todo);
  // 끝. RxList가 자동으로 감지한다.
}

void removeTodo(Todo todo) {
  todoList.remove(todo);
  // 역시 자동.
}
```

List의 `add`, `remove`는 RxList가 감지해서 자동으로 UI가 갱신된다.

### 상태 전환 — refresh()가 필요한 순간

```dart
void changeTodoStatus(Todo todo) async {
  switch (todo.status) {
    case TodoStatus.inComplete:
      todo.status = TodoStatus.onGoing;
    case TodoStatus.onGoing:
      todo.status = TodoStatus.complete;
    case TodoStatus.complete:
      final result = await ConfirmDialog('정말로 처음 상태로 변경하시겠어요?').show();
      result?.runIfSuccess((data) {
        todo.status = TodoStatus.inComplete;
      });
  }
  todoList.refresh();  // 이거 없으면 UI 안 바뀜!
}
```

여기서 함정이 하나 있다. `todo.status`를 직접 바꾸면 Todo 객체의 프로퍼티만 변경되고, List 자체는 변경이 없다. RxList 입장에서는 "List에 아무 변화 없는데?"라고 판단하기 때문에 `refresh()`를 수동으로 호출해야 한다.

mutable 객체를 직접 수정할 때 생기는 문제다. ValueNotifier에서 `notifyListeners()`를 수동으로 호출해야 했던 것과 본질적으로 같은 문제가 형태만 바뀌어서 다시 등장한 거다.

### 수정도 마찬가지

```dart
void editTodo(Todo todo, WriteTodoResult result) {
  todo.title = result.text;
  todo.dueDate = result.dateTime;
  todo.modifyTime = DateTime.now();
  todoList.refresh();  // 수동 refresh
}
```

## GetX의 편리함과 위험성

### 코드량 비교

| 항목 | ValueNotifier + InheritedWidget | GetX |
|------|-------------------------------|------|
| 상태 클래스 | ValueNotifier 상속 + InheritedWidget 별도 클래스 | GetxController 하나 |
| 등록 | InheritedWidget 래퍼로 위젯 트리 감싸기 | `Get.put()` 한 줄 |
| 구독 | `ValueListenableBuilder` (builder 패턴) | `Obx(() =>)` |
| 변경 알림 | `notifyListeners()` 수동 | `.obs` 자동 |
| 접근 | `context.dependOnInheritedWidgetOfExactType` | `Get.find()` |

확실히 코드량은 GetX가 압도적으로 적다.

### 근데 왜 GetX를 안 쓰는 프로젝트가 있을까?

Ch04에서 다뤘던 Static Model의 위험성이 그대로 적용된다:

1. **`Get.find()`로 어디서든 접근/수정 가능** → 상태 변경 추적이 어려움
2. **메모리 관리를 개발자가 직접** → `Get.delete()` 안 하면 메모리 누수
3. **mutable 상태 + 전역 접근** → 디버깅 지옥이 될 수 있음

iOS 개발할 때 싱글톤 패턴의 교훈과 정확히 같다. `AppState.shared.user = newUser`를 아무 데서나 부르면 어디서 바뀌었는지 추적이 안 되는 것처럼, GetX의 `Get.find()`도 어디서든 상태를 수정할 수 있기 때문에 같은 문제가 생긴다.

Swift 커뮤니티가 싱글톤에서 DI(의존성 주입)로 이동한 것처럼, Flutter 커뮤니티에서도 GetX에서 Scoped 방식(Provider, BLoC, Riverpod)으로 이동하는 추세가 있다. 물론 작은 앱이나 프로토타이핑에는 GetX가 여전히 최고의 선택이다.

## 정리

| 개념 | ValueNotifier | GetX |
|------|-------------|------|
| 방식 | Scoped (내장) | Static (패키지) |
| 변경 알림 | `notifyListeners()` 수동 | `.obs` 자동 |
| UI 구독 | `ValueListenableBuilder` | `Obx(() =>)` |
| 등록/접근 | InheritedWidget / `of(context)` | `Get.put()` / `Get.find()` |
| context 필요 | 필요 | 불필요 |
| 메모리 관리 | 위젯 트리가 자동 해제 | 개발자가 직접 `Get.delete()` |

GetX는 확실히 편하다. `notifyListeners()` 안 불러도 되고, InheritedWidget 만들 필요도 없다. 근데 mutable 객체의 `refresh()` 함정이 있고, 앱이 커지면 상태 추적이 힘들어진다. 다음 Ch05-3에서는 BLoC으로 전환한다. Event-driven 구조가 이 추적 문제를 어떻게 해결하는지, 그리고 mutable Todo를 freezed로 immutable하게 만들면 `refresh()` 같은 함정이 어떻게 사라지는지 비교해보자.
