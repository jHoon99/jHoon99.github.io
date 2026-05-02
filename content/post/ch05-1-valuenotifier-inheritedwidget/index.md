---
title: "Ch05-1. Todo 앱으로 상태관리 입문 - ValueNotifier + InheritedWidget"
slug: "ch05-1-valuenotifier-inheritedwidget"
date: 2026-05-02
weight: 3
draft: false
tags: ["Flutter", "Dart", "State Management", "ValueNotifier", "InheritedWidget", "Todo App"]
categories: ["Flutter"]
description: "Flutter 내장 도구만으로 Todo 앱의 상태관리를 구현하면서 ValueNotifier와 InheritedWidget의 동작 원리를 파헤친 기록"
---

Ch04에서 상태관리의 역사와 Scoped vs Static을 다뤘으니, 이제 실전이다. Todo 앱 하나를 만들고, 같은 앱을 3가지 상태관리 방식으로 리팩토링하는 시리즈다. 첫 번째는 Flutter 내장 도구만 쓴다. 외부 패키지 없이 원리부터 이해하는 게 목적이다.

## Todo 앱 구조

먼저 전체 프로젝트 구조를 보자.

```
lib/
├── app.dart                       // 앱 진입점
├── data/
│   └── memory/
│       ├── vo_todo.dart           // Todo 모델
│       ├── todo_status.dart       // 상태 enum
│       ├── todo_data_notifier.dart // ValueNotifier
│       └── todo_data_holder.dart  // InheritedWidget
├── screen/
│   └── main/
│       ├── s_main.dart            // 메인 화면 (탭 네비게이션)
│       ├── todo/
│       │   ├── f_todo.dart        // Todo 탭 전체
│       │   ├── w_todo_list.dart   // 리스트 위젯
│       │   ├── w_todo_item.dart   // 개별 아이템
│       │   └── w_todo_status.dart // 상태 표시 (Rive 애니메이션)
│       └── write/
│           ├── d_write_todo.dart  // 추가/수정 다이얼로그
│           └── vo_write_result.dart
```

파일 이름에 접두어가 붙는다. `s_`는 screen, `f_`는 fragment, `w_`는 widget, `d_`는 dialog, `vo_`는 value object. iOS에서 ViewController, ViewModel 같은 네이밍 컨벤션과 비슷한 느낌이다.

## Todo 모델

```dart
class Todo {
  int id;
  String title;
  final DateTime createdTime;
  DateTime? modifyTime;
  DateTime dueDate;
  TodoStatus status;

  Todo({
    required this.id,
    required this.title,
    required this.dueDate,
    this.status = TodoStatus.inComplete,
  }) : createdTime = DateTime.now();
}
```

그냥 평범한 mutable class다. `id`, `title`, `dueDate`, `status`를 가지고 있고, 직접 프로퍼티를 수정할 수 있다. Swift로 치면 `class`(reference type)에 `var` 프로퍼티들을 쓴 거다. 나중에 Ch05-3에서 이걸 freezed로 immutable하게 바꾸면 뭐가 달라지는지 비교할 예정이다.

```dart
enum TodoStatus {
  inComplete,
  onGoing,
  complete,
}
```

상태 흐름은 `inComplete → onGoing → complete` 순서다. complete에서 한 번 더 누르면 확인 다이얼로그가 뜨고, 승인하면 다시 inComplete로 돌아간다.

## UI 위젯들

UI는 간단하게 넘어가자. 핵심만 보면:

- **MainScreen**: 하단 탭 네비게이션 + FloatingActionButton으로 할 일 추가
- **TodoItem**: `Dismissible` 위젯으로 스와이프 삭제 지원. SwiftUI의 `.swipeActions`와 같은 역할인데, Flutter는 위젯 자체가 스와이프를 지원해서 편하다
- **TodoStatusWidget**: 상태에 따라 체크박스 표시. `onGoing`일 때는 Rive 애니메이션이 재생된다
- **WriteTodoDialog**: 할 일 추가/수정 다이얼로그. `todoForEdit`이 있으면 수정 모드, 없으면 추가 모드

```dart
// TodoItem에서 스와이프 삭제
Dismissible(
  key: ValueKey(todo.id),
  onDismissed: (direction) {
    // 여기서 삭제 로직 호출
  },
  child: RoundedContainer(
    child: Column(
      children: [
        Text(todo.dueDate.toString()),
        Row(
          children: [
            TodoStatusWidget(todo),
            Text(todo.title),
            const Spacer(),
            IconButton(
              onPressed: () {
                // 수정 로직 호출
              },
              icon: const Icon(EvaIcons.editOutline),
            )
          ],
        ),
      ],
    ),
  ),
)
```

## ValueNotifier — setState의 진화

### setState의 한계

`setState`는 해당 위젯 내부에서만 작동한다. 부모에서 만든 todoList를 여러 자식 위젯이 공유해야 할 때, 콜백으로 내려보내거나 prop drilling을 해야 한다. SwiftUI에서 `@Binding`을 n단계 내려보내는 고통과 같다.

### ValueNotifier란

```dart
class TodoDataNotifier extends ValueNotifier<List<Todo>> {
  TodoDataNotifier() : super([]); // 최초 빈 리스트로 시작

  void addTodo(Todo todo) {
    value.add(todo);
    notifyListeners();
  }

  void notify() {
    notifyListeners();
  }
}
```

`ValueNotifier<T>`는 Flutter 내장 클래스다. `value`가 바뀌면 `notifyListeners()`를 호출해서 구독 중인 위젯을 리빌드시킨다.

Swift랑 비교하면:

| Flutter | SwiftUI |
|---------|---------|
| `ValueNotifier<T>` | `ObservableObject` + `@Published` |
| `notifyListeners()` | `objectWillChange.send()` (수동) |
| `value` | `@Published var value: T` |

핵심 차이는, Swift의 `@Published`는 값이 바뀌면 **자동으로** 알림을 보내는데, Flutter의 `ValueNotifier`에서는 List 내부를 수정하면(`value.add()`) 참조 자체는 안 바뀌니까 **수동으로** `notifyListeners()`를 호출해야 한다. 이걸 까먹으면 UI가 안 바뀌는 버그가 생긴다.

### ValueListenableBuilder로 UI 연결

```dart
ValueListenableBuilder<List<Todo>>(
  valueListenable: todoNotifier,
  builder: (context, todoList, child) {
    return Column(
      children: todoList.map((e) => TodoItem(e)).toList(),
    );
  },
)
```

SwiftUI에서 `@ObservedObject`를 쓰면 `body`가 자동 리빌드되는 것과 같은 역할이다. `builder` 안에서 `todoList`가 바뀔 때마다 UI가 다시 그려진다. `child` 파라미터는 리빌드할 필요 없는 자식을 캐싱하는 용도인데, 이건 성능 최적화 포인트다.

## InheritedWidget — 위젯 트리로 데이터 전파

### 왜 필요한가

ValueNotifier만으로는 "누가 이 Notifier를 들고 있느냐"가 문제다. 생성자로 내려보내면 결국 prop drilling이 다시 발생한다. InheritedWidget은 위젯 트리에 데이터를 심어두고, 하위 어디서든 꺼내 쓸 수 있게 해주는 메커니즘이다.

사실 이 프로젝트에서 이미 다크모드 테마 시스템으로 InheritedWidget을 쓰고 있었다:

```dart
// 이미 있던 테마용 InheritedWidget
class CustomThemeHolder extends InheritedWidget {
  final AbstractThemeColors appColors;
  final CustomTheme theme;
  final Function(CustomTheme) changeTheme;

  static CustomThemeHolder of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<CustomThemeHolder>()!;
  }

  @override
  bool updateShouldNotify(CustomThemeHolder old) {
    return theme != old.theme;
  }
}
```

같은 원리를 상태관리에 적용하는 거다.

### TodoDataHolder 구현

```dart
class TodoDataHolder extends InheritedWidget {
  final TodoDataNotifier notifier;

  const TodoDataHolder({
    required this.notifier,
    required Widget child,
    Key? key,
  }) : super(key: key, child: child);

  static TodoDataNotifier of(BuildContext context) {
    return context
        .dependOnInheritedWidgetOfExactType<TodoDataHolder>()!
        .notifier;
  }

  @override
  bool updateShouldNotify(TodoDataHolder old) {
    return notifier != old.notifier;
  }
}
```

`of(context)` 패턴은 Flutter에서 아주 관용적인 접근법이다. `Theme.of(context)`, `MediaQuery.of(context)` 등 전부 이 패턴이다. 내부적으로 `context.dependOnInheritedWidgetOfExactType`은 Element의 해시 테이블을 조회해서 O(1)로 찾는다.

SwiftUI 비교:

| Flutter | SwiftUI |
|---------|---------|
| `InheritedWidget` + `of(context)` | `@EnvironmentObject` |
| 직접 InheritedWidget 클래스 작성 | Property Wrapper가 자동 처리 |
| `context.dependOn...` | `@EnvironmentObject var vm: ViewModel` |

SwiftUI가 확실히 간편하다. Flutter에서는 InheritedWidget 클래스를 직접 만들어야 하는데, SwiftUI는 `@EnvironmentObject`만 붙이면 끝이다. 근데 원리는 같다.

### context extension으로 축약

```dart
extension ContextExtension on BuildContext {
  TodoDataHolder get todoHolder => TodoDataHolder.of(this);
}

// 사용
context.todoHolder.addTodo(todo);
```

매번 `TodoDataHolder.of(context).notifier.addTodo()`라고 쓰면 너무 기니까 extension으로 축약한다. 테마에서도 `context.appColors`로 이미 같은 패턴을 쓰고 있었다.

### App에서 Provide

```dart
// app.dart
CustomThemeApp(
  child: TodoDataHolder(
    notifier: TodoDataNotifier(),
    child: MaterialApp(
      home: const MainScreen(),
    ),
  ),
)
```

`MaterialApp` 위에 InheritedWidget을 감싸서 앱 전체에서 접근 가능하게 한다. SwiftUI에서 `.environmentObject(todoVM)`을 최상위에 넣는 것과 같은 구조다.

## mounted — 비동기에서 context 안전하게 쓰기

상태관리와 별개로 중요한 포인트가 하나 있었다. 비동기 작업 중에 위젯이 dispose되면 `context`를 쓸 수 없는 문제다.

```dart
void _loadRive() async {
  _cachedFile = await File.asset('assets/rive/fire_button.riv');

  // 비동기 작업이 끝났는데 위젯이 이미 사라졌다면?
  if (_cachedFile != null && mounted) {  // mounted로 체크!
    _controller = RiveWidgetController(_cachedFile!);
    setState(() => _riveReady = true);
  }
}
```

`await` 후에 `setState`를 부르는데, 그 사이에 사용자가 화면을 나갔다면 위젯이 dispose된 상태다. 이때 `setState`를 호출하면 에러가 난다. `mounted` 프로퍼티로 위젯이 아직 살아있는지 체크하는 게 안전하다.

iOS에서 `[weak self]`로 self가 nil인지 체크하는 것과 비슷한 맥락이다:

```swift
// Swift
Task { [weak self] in
    let data = await fetchData()
    guard let self else { return }  // self가 살아있는지 체크
    self.updateUI(data)
}
```

```dart
// Flutter
void someAsyncWork() async {
  final data = await fetchData();
  if (mounted) {  // 위젯이 살아있는지 체크
    setState(() => _data = data);
  }
}
```

## 상태 변경 흐름 정리

### 추가

```dart
void _addTodo() async {
  final result = await WriteTodoDialog().show();
  if (result != null) {
    context.todoHolder.addTodo(
      Todo(
        id: DateTime.now().microsecondsSinceEpoch,
        title: result.text,
        dueDate: result.dateTime,
      ),
    );
  }
}
```

다이얼로그에서 결과를 받아와서 Notifier에 추가한다. `WriteTodoDialog`는 제네릭 타입으로 `WriteTodoResult`를 반환하는 구조다.

### 상태 전환

```dart
void _changeTodoStatus(Todo todo) async {
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
  context.todoHolder.notify();  // 수동으로 알림
}
```

mutable이니까 `todo.status`를 직접 바꾸고, `notify()`로 UI를 갱신한다. `ConfirmDialog`는 `SimpleResult` 패턴으로 결과를 체이닝한다. Swift의 `Result<Success, Failure>` 패턴과 비슷하다.

### 삭제

```dart
// Dismissible의 onDismissed에서
context.todoHolder.value.remove(todo);
context.todoHolder.notify();
```

스와이프로 삭제하면 리스트에서 제거하고 알림을 보낸다.

## 이 접근법의 장단점

**장점:**
- 외부 패키지 0개. Flutter 기본 API만으로 구현
- InheritedWidget 동작 원리를 이해하면 Provider, BLoC 등 모든 Scoped 방식의 기반을 이해한 거다
- 간단한 앱에는 충분

**한계:**
- `notifyListeners()` 수동 호출을 까먹기 쉬움
- mutable 객체를 직접 수정하니까 어디서 바뀌었는지 추적이 어려움
- 비즈니스 로직이 Notifier, UI 여기저기에 흩어질 수 있음
- InheritedWidget을 직접 만드는 보일러플레이트가 꽤 있음

## 정리

| 개념 | Flutter (ValueNotifier) | SwiftUI |
|------|------------------------|---------|
| 상태 홀더 | `ValueNotifier` | `ObservableObject` |
| 변경 알림 | `notifyListeners()` 수동 | `@Published` 자동 |
| UI 구독 | `ValueListenableBuilder` | `@ObservedObject` / body 자동 |
| 트리 전파 | `InheritedWidget` + `of(context)` | `@EnvironmentObject` |
| 비동기 안전 | `mounted` 체크 | `[weak self]` / `guard let self` |

Flutter 내장만으로 상태관리를 해봤다. 동작은 하는데 `notifyListeners()` 수동 호출이 좀 거슬린다. 다음 Ch05-2에서는 GetX로 전환하면서 이 수동 호출이 `.obs`로 바뀌면 얼마나 코드가 줄어드는지 비교해보자.
