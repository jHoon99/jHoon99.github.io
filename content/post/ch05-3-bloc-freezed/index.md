---
title: "Ch05-3. Todo 앱 BLoC + freezed 전환 - 이벤트 기반 상태관리"
slug: "ch05-3-bloc-freezed"
date: 2026-05-01
weight: 4
draft: false
tags: ["Flutter", "Dart", "State Management", "BLoC", "freezed", "sealed class", "Immutable", "Todo App"]
categories: ["Flutter"]
description: "Todo 앱을 BLoC + freezed로 전환하면서 Event-driven 아키텍처와 불변 상태의 장점을 체감한 기록"
---

Ch05 시리즈의 마지막이다. GetX에서 BLoC으로 전환한다. 핵심 변화는 2가지다: (1) 상태 변경을 **Event**로 제한하고, (2) Todo 모델을 freezed로 **immutable**하게 바꾸는 것. 코드량은 늘어나지만 "상태가 어디서 왜 바뀌었는지" 추적이 확실해진다.

## BLoC이란

**B**usiness **Lo**gic **C**omponent. UI에서 Event를 보내면, Bloc이 처리해서 새로운 State를 내보내는 단방향 흐름이다.

```
[UI] --Event--> [Bloc] --State--> [UI]
                  |
              비즈니스 로직
```

GetX에서는 `todoData.addTodo(todo)` 처럼 직접 메서드를 호출했다. BLoC에서는 `bloc.add(TodoAddEvent())` 처럼 이벤트를 보낸다. 뷰에서는 호출만 하고, Bloc 내부에서 이벤트를 받아 해당하는 로직을 수행하고, 상태를 변경하면, BlocBuilder가 돌아간다.

차이가 뭐냐: **모든 상태 변경이 Event로 기록된다**. 어떤 이벤트가 언제 발생했는지 로그로 추적할 수 있고, 디버깅이 쉬워진다. GetX는 "야 이거 바꿔"라고 직접 수정하는 거고, BLoC은 "이거 바꿔달라"는 요청서를 제출하는 거다.

Swift에서 TCA(The Composable Architecture)를 써봤다면 `Action → Reducer → State` 흐름과 거의 같은 패턴이다.

## freezed로 불변 모델 만들기

### mutable의 문제 (Ch05-2 복습)

GetX에서 Todo의 상태를 바꿀 때 이런 식이었다:

```dart
// GetX 시절: mutable 객체를 직접 수정
todo.status = TodoStatus.complete;
todoList.refresh();  // 수동 refresh 필요
```

문제점:
- 어디서 `todo.status`를 바꿨는지 추적 불가
- `refresh()` 까먹으면 UI 안 바뀜
- 같은 Todo 인스턴스를 여러 곳에서 참조하면 의도치 않은 사이드 이펙트

### freezed 적용

```dart
@freezed
class Todo with _$Todo {
  const factory Todo({
    required int id,
    required String title,
    required DateTime createdTime,
    DateTime? modifyTime,
    required DateTime dueDate,
    required TodoStatus status,
  }) = _Todo;
}
```

`const factory`로 선언하면 모든 필드가 `final`이 된다. 직접 수정이 불가능하다. 대신 `copyWith`로 새 인스턴스를 만들어야 한다.

```dart
// Before (mutable class)
todo.status = TodoStatus.complete;  // 직접 수정

// After (freezed)
final newTodo = todo.copyWith(status: TodoStatus.complete);  // 새 인스턴스 생성
```

Swift의 `struct`(value type)와 거의 같다. Swift에서는 struct가 기본 제공하는 `==`, `hashCode`, `copyWith`(없지만 멤버와이즈 이니셜라이저가 비슷한 역할)를 Dart에서는 freezed가 코드 생성으로 만들어준다.

처음에는 좀 번거로웠다. 기존처럼 switch문으로 상태를 변경하고 할당시켜도 안 됐다. 내부에서 상태를 수정할 수 없으니까 `copyWith`로 별도 수정해줘야 한다. 근데 이 불편함이 안전함의 대가다.

### BlocState도 freezed로

```dart
enum BlocStatus { initial, loading, success, error }

@freezed
class TodoBlocState with _$TodoBlocState {
  const factory TodoBlocState({
    required BlocStatus status,
    required List<Todo> todoList,
  }) = _TodoBlocState;
}
```

State에 `BlocStatus`를 포함시켜서 로딩/에러 상태도 관리할 수 있다. `todoList`와 `status`를 하나의 immutable State 객체로 묶었다.

### build_runner로 코드 생성

```bash
flutter pub run build_runner build
```

이 명령어로 `vo_todo.freezed.dart`, `todo_bloc_state.freezed.dart`가 생성된다. `copyWith`, `==`, `hashCode`, `toString`이 전부 자동으로 만들어진다.

## Event 설계 — sealed class

```dart
sealed class TodoEvent {}

class TodoAddEvent extends TodoEvent {}

class TodoStatusUpdateEvent extends TodoEvent {
  final Todo updatedTodo;
  TodoStatusUpdateEvent(this.updatedTodo);
}

class TodoContentUpdateEvent extends TodoEvent {
  final Todo updatedTodo;
  TodoContentUpdateEvent(this.updatedTodo);
}

class TodoRemoveEvent extends TodoEvent {
  final Todo removedTodo;
  TodoRemoveEvent(this.removedTodo);
}
```

처음에는 `abstract class`로 했다가 `sealed class`로 바꿨다. sealed로 바꾸면 switch문에서 모든 이벤트를 빠짐없이 처리했는지 컴파일 타임에 체크해준다. 이벤트를 하나 추가했는데 핸들러를 안 만들면 컴파일 에러가 나니까 실수를 방지할 수 있다.

Swift의 `enum` + associated values와 거의 같은 역할이다:

```swift
// Swift equivalent
enum TodoAction {
    case add
    case updateStatus(Todo)
    case updateContent(Todo)
    case remove(Todo)
}
```

이벤트를 sealed class로 만드는 이유는 각 이벤트가 서로 다른 데이터를 가져야 할 때 유연하기 때문이다. `TodoAddEvent`는 데이터가 필요 없고(다이얼로그에서 받으니까), `TodoRemoveEvent`는 어떤 Todo를 삭제할지 알아야 하고, 이런 차이를 타입으로 표현할 수 있다.

## TodoBloc — 핵심 로직

### Bloc 클래스 구조

```dart
class TodoBloc extends Bloc<TodoEvent, TodoBlocState> {
  TodoBloc() : super(const TodoBlocState(
    status: BlocStatus.initial,
    todoList: <Todo>[],
  )) {
    on<TodoAddEvent>(_addTodo);
    on<TodoStatusUpdateEvent>(_changeTodoStatus);
    on<TodoContentUpdateEvent>(_editTodo);
    on<TodoRemoveEvent>(_removeTodo);
  }
}
```

`super(initialState)`로 초기 상태를 설정하고, `on<EventType>(handler)`로 각 이벤트별 핸들러를 등록한다. 이벤트가 들어오면 해당 핸들러가 실행되는 구조다.

### 추가

```dart
void _addTodo(TodoAddEvent event, Emitter<TodoBlocState> emit) async {
  final result = await WriteTodoDialog().show();
  if (result != null) {
    final oldTodoList = List.of(state.todoList);
    oldTodoList.add(
      Todo(
        id: DateTime.now().microsecondsSinceEpoch,
        title: result.text,
        dueDate: result.dateTime,
        createdTime: DateTime.now(),
        status: TodoStatus.inComplete,
      ),
    );
    emitNewList(oldTodoList, emit);
  }
}
```

핵심 패턴: `List.of(state.todoList)`로 기존 리스트를 복사 → 수정 → `emit`으로 새 State를 발행한다. BLoC에서는 `add`를 사용할 수 없기 때문에(state.todoList가 immutable) 새 List로 감싸서 조작해야 한다.

```dart
void emitNewList(List<Todo> oldTodoList, Emitter<TodoBlocState> emit) {
  emit(state.copyWith(todoList: oldTodoList));
}
```

`emit`을 호출하면 BLoC이 새 State를 내보내고, 구독 중인 `BlocBuilder`가 리빌드된다.

### 상태 전환 (가장 복잡한 부분)

```dart
void _changeTodoStatus(TodoStatusUpdateEvent event, Emitter<TodoBlocState> emit) async {
  final oldTodoList = List.of(state.todoList);
  final todo = event.updatedTodo;
  final todoIndex = oldTodoList.indexWhere((e) => e.id == todo.id);

  TodoStatus status = todo.status;
  switch (todo.status) {
    case TodoStatus.inComplete:
      status = TodoStatus.onGoing;
    case TodoStatus.onGoing:
      status = TodoStatus.complete;
    case TodoStatus.complete:
      final result = await ConfirmDialog('정말로 처음 상태로 변경하시겠어요?').show();
      result?.runIfSuccess((data) {
        status = TodoStatus.inComplete;
      });
  }
  oldTodoList[todoIndex] = todo.copyWith(status: status);
  emitNewList(oldTodoList, emit);
}
```

GetX에서는 `todo.status = status`로 직접 수정했는데, BLoC에서는 `todo.copyWith(status: status)`로 새 인스턴스를 만들고, `indexWhere`로 찾아서 교체한다. immutable이라 기존 객체를 수정할 수 없으니까 새 객체로 대체하는 거다.

GetX에서 `todoList.refresh()`를 수동으로 호출해야 했던 문제가 여기서 사라진다. 새 리스트를 `emit`하면 BLoC이 자동으로 상태 변경을 감지한다.

### 삭제

```dart
void _removeTodo(TodoRemoveEvent event, Emitter<TodoBlocState> emit) {
  final oldTodoList = List<Todo>.from(state.todoList);
  final todo = event.removedTodo;
  oldTodoList.removeWhere((e) => e.id == todo.id);
  emitNewList(oldTodoList, emit);
}
```

### 수정

```dart
void _editTodo(TodoContentUpdateEvent event, Emitter<TodoBlocState> emit) async {
  final todo = event.updatedTodo;
  final result = await WriteTodoDialog(todoForEdit: todo).show();
  if (result != null) {
    final oldTodoList = List<Todo>.from(state.todoList);
    oldTodoList[oldTodoList.indexOf(todo)] = todo.copyWith(
      title: result.text,
      dueDate: result.dateTime,
      modifyTime: DateTime.now(),
    );
    emitNewList(oldTodoList, emit);
  }
}
```

## UI에서 BLoC 사용

### BlocProvider로 등록

```dart
// app.dart
BlocProvider(
  create: (BuildContext context) => TodoBloc(),
  child: MaterialApp(
    home: const MainScreen(),
  ),
)
```

InheritedWidget을 직접 만들 필요 없이 `BlocProvider`가 대신해준다. 내부적으로 InheritedWidget 기반이지만 보일러플레이트가 없다. GetX의 `Get.put()`과 비교하면, BlocProvider는 위젯이 dispose될 때 자동으로 Bloc도 close해준다. 메모리 관리를 신경 쓸 필요가 없다.

### BlocBuilder로 구독

```dart
// w_todo_list.dart
BlocBuilder<TodoBloc, TodoBlocState>(
  builder: (context, state) {
    return state.todoList.isEmpty
        ? const Center(child: Text('할 일을 작성해 보세요.'))
        : Column(
            children: state.todoList.map((e) => TodoItem(e)).toList(),
          );
  },
)
```

`state`로 현재 상태 전체를 받는다. GetX의 `Obx`처럼 `.obs` 변수를 직접 접근하는 게 아니라, Bloc이 emit한 State 객체를 통해 데이터에 접근한다.

### context.read vs context.watch

```dart
// context_extension.dart
extension ContextExtension on BuildContext {
  TodoBloc get readTodoBloc => read();   // 이벤트 보낼 때 (리빌드 안 함)
  TodoBloc get watchTodoBloc => watch(); // 상태 구독할 때 (리빌드 함)
}
```

- `read`: 한 번 읽고 끝. 이벤트를 보낼 때 사용한다. 리빌드를 트리거하지 않는다.
- `watch`: 상태 변경을 구독한다. State가 바뀌면 해당 위젯이 리빌드된다.

```dart
// 이벤트 보내기: read (리빌드 불필요)
context.readTodoBloc.add(TodoAddEvent());
context.readTodoBloc.add(TodoRemoveEvent(todo));
context.readTodoBloc.add(TodoStatusUpdateEvent(todo));

// 상태 구독: BlocBuilder 안에서 자동으로 처리
```

주로 이벤트를 보낼 때는 `read`, UI를 그릴 때는 `BlocBuilder` 안에서 state를 받아 쓴다.

### 실제 위젯에서 사용

```dart
// w_todo_item.dart - 스와이프 삭제
Dismissible(
  key: ValueKey(todo.id),
  onDismissed: (direction) {
    context.readTodoBloc.add(TodoRemoveEvent(todo));
  },
  child: RoundedContainer(
    child: Row(
      children: [
        TodoStatusWidget(todo),
        Text(todo.title),
        const Spacer(),
        IconButton(
          onPressed: () {
            context.readTodoBloc.add(TodoContentUpdateEvent(todo));
          },
          icon: const Icon(EvaIcons.editOutline),
        )
      ],
    ),
  ),
)
```

```dart
// s_main.dart - FAB으로 추가
FloatingActionButton(
  onPressed: () {
    context.readTodoBloc.add(TodoAddEvent());
  },
  child: const Icon(EvaIcons.plus),
)
```

UI에서는 `add(Event)`로 이벤트를 보내기만 한다. 로직은 전부 Bloc 내부에 있다. 이게 관심사 분리다.

## Rive 애니메이션 (보너스)

상태관리와 직접 관련은 없지만, Todo 상태가 `onGoing`일 때 Rive 애니메이션이 재생되는 게 꽤 재밌었다.

```dart
child: switch (widget.todo.status) {
  TodoStatus.complete => Checkbox(value: true, onChanged: null),
  TodoStatus.inComplete => const Checkbox(value: false, onChanged: null),
  TodoStatus.onGoing => _riveReady
      ? RiveWidget(controller: _controller!, fit: Fit.cover)
      : const SizedBox(),
}
```

Dart 3의 switch expression으로 상태별 위젯을 깔끔하게 분기한다. `_cachedFile`을 static으로 캐싱해서 .riv 파일을 한 번만 로드하는 것도 포인트다.

## 3가지 방식 최종 비교

### 코드 비교

| 항목 | ValueNotifier | GetX | BLoC |
|------|-------------|------|------|
| 방식 | Scoped (내장) | Static | Scoped (패키지) |
| 상태 객체 | mutable | mutable | immutable (freezed) |
| 변경 알림 | `notifyListeners()` 수동 | `.obs` 자동 | `emit()` |
| UI 구독 | `ValueListenableBuilder` | `Obx(() =>)` | `BlocBuilder` |
| 등록 | InheritedWidget 직접 구현 | `Get.put()` | `BlocProvider` |
| 접근 | `of(context)` | `Get.find()` | `context.read/watch` |
| 상태 변경 | 메서드 직접 호출 | 메서드 직접 호출 | Event dispatch |
| 추적성 | 낮음 | 낮음 | 높음 (Event 로깅) |
| 보일러플레이트 | 중간 | 적음 | 많음 |
| 메모리 관리 | 위젯 트리 자동 | 개발자 직접 | 위젯 트리 자동 |

### 같은 "Todo 추가" 동작

```dart
// ValueNotifier
context.todoHolder.addTodo(todo);

// GetX
Get.find<TodoDataHolder>().addTodo(todo);

// BLoC
context.readTodoBloc.add(TodoAddEvent());
```

### Swift/iOS 대응표

| Flutter | Swift/iOS |
|---------|-----------|
| ValueNotifier | `ObservableObject` + `@Published` |
| InheritedWidget | `@EnvironmentObject` |
| GetX (`.obs` + `Obx`) | `Combine` + `@Published` (싱글톤) |
| BLoC (Event → State) | TCA (`Action` → `Reducer` → `State`) |
| freezed | `struct` (value type) |
| sealed class | `enum` + associated value |
| BlocProvider | DI Container (Scoped) |

### 어떤 걸 써야 하나

- **프로토타입/작은 앱** → GetX가 빠르다
- **중간 규모** → Provider나 Riverpod이 적당하다
- **대규모/팀 프로젝트** → BLoC이 Event 추적, 테스트 용이성에서 유리하다
- **Flutter 원리 학습** → ValueNotifier + InheritedWidget부터 이해하자

개인 프로젝트에서 정답은 없다. 트레이드오프를 이해하고 고르면 된다.

## 정리

Ch05 시리즈를 통해 같은 Todo 앱을 3가지 방식으로 만들어봤다. 어떤 방식이든 하는 일은 같다. 상태를 만들고, 변경하고, UI에 반영하는 것. 차이는 "변경을 얼마나 통제할 것인가", "추적성을 얼마나 확보할 것인가"에 있다.

iOS 개발할 때도 RxSwift → Combine → TCA 순서로 더 구조화된 방향으로 흘러왔는데, Flutter에서도 비슷한 흐름을 체감했다. Ch04에서 이론을 정리하고 Ch05에서 실전을 해보니까 왜 상태관리가 중요한지 확실히 느꼈다.
