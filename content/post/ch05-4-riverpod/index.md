---
title: "Ch05-4. Todo 앱 Riverpod 전환 - 이게 진짜 사기다"
slug: "ch05-4-riverpod"
date: 2026-05-02
weight: 3
draft: false
tags: ["Flutter", "Dart", "State Management", "Riverpod", "StateNotifier", "ConsumerWidget", "FutureProvider", "Todo App"]
categories: ["Flutter"]
description: "BLoC으로 만든 Todo 앱을 Riverpod으로 전환하면서 StateNotifier, ref.watch, ProviderScope, FutureProvider의 .when 패턴까지 정리한 기록"
---

Ch05 시리즈 마지막이다. BLoC에서 Riverpod으로 전환한다. 솔직히 BLoC의 Event 기반 구조가 MVI 느낌이라 꽤 마음에 들었는데, Riverpod 써보니까 생산성이 말이 안 된다. 뷰에서 상태별 분기가 `.when` 하나로 되고, BLoC에서 필요했던 Event 클래스, State 래퍼, Emitter 전달 같은 보일러플레이트가 싹 사라진다.

> 이 글의 코드는 **flutter_riverpod 2.6.1** 기준이다. 최신 Riverpod 3.0과의 차이점은 글 하단에 정리했다.

## BLoC → Riverpod, 뭐가 바뀌나

| 영역 | BLoC | Riverpod |
|------|------|----------|
| 상태 클래스 | `Bloc<Event, State>` 상속 | `StateNotifier<T>` 상속 |
| 이벤트 | `sealed class TodoEvent` 별도 정의 | 없음. 메서드 직접 호출 |
| State 래퍼 | `@freezed TodoBlocState` | 직접 `List<Todo>` |
| 등록 | `BlocProvider(create: ...)` | `StateNotifierProvider(...)` 전역 선언 |
| UI 위젯 | `StatelessWidget` + `BlocBuilder` | `ConsumerWidget` |
| 접근 | `context.read<TodoBloc>()` | `ref.read(provider.notifier)` |
| 구독 | `BlocBuilder<TodoBloc, TodoBlocState>` | `ref.watch(provider)` |
| 상태 발행 | `emit(state.copyWith(...))` | `state = newValue` |

요약하면 BLoC에서 **Event 클래스 + State 래퍼 + Emitter 패턴**이 전부 사라지고, **직접 메서드 호출 + state 재할당**으로 단순해진다.

## StateNotifierProvider — 핵심 구조

### Provider 선언 (전역)

```dart
final todoDataProvider =
    StateNotifierProvider<TodoDataHolder, List<Todo>>(
        (ref) => TodoDataHolder());
```

이게 끝이다. BLoC에서는 `BlocProvider`로 위젯 트리에 감싸야 했는데, Riverpod은 전역 변수처럼 선언한다. "provider가 전역으로 선언되었다고 데이터도 전역으로 같이 쓰는 게 아니다" — 이건 뒤에서 Scope로 설명한다.

### StateNotifier 구현

```dart
class TodoDataHolder extends StateNotifier<List<Todo>> {
  TodoDataHolder() : super(<Todo>[]);

  void addTodo() async {
    final result = await WriteTodoDialog().show();
    if (result != null) {
      state.add(
        Todo(
          id: DateTime.now().microsecondsSinceEpoch,
          title: result.text,
          dueDate: result.dateTime,
        ),
      );
      state = List.of(state); // 새 리스트로 재할당 → 리빌드 트리거
    }
  }

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
    state = List.of(state);
  }

  void editTodo(Todo todo) async {
    final result = await WriteTodoDialog(todoForEdit: todo).show();
    if (result != null) {
      todo.title = result.text;
      todo.dueDate = result.dateTime;
      state = List.of(state);
    }
  }

  void removeTodo(Todo todo) {
    state.remove(todo);
    state = List.of(state);
  }
}
```

BLoC에서는 `emit(state.copyWith(todoList: oldTodoList))`로 새 State를 발행했는데, Riverpod에서는 `state = List.of(state)`로 직접 재할당한다. `state`에 새 값을 넣으면 구독 중인 위젯이 자동으로 리빌드된다.

BLoC과 비교하면:

```dart
// BLoC: Event를 정의하고, 핸들러를 등록하고, Emitter로 발행
sealed class TodoEvent {}
class TodoAddEvent extends TodoEvent {}

class TodoBloc extends Bloc<TodoEvent, TodoBlocState> {
  TodoBloc() : super(...) {
    on<TodoAddEvent>(_addTodo);  // 핸들러 등록
  }
  void _addTodo(TodoAddEvent event, Emitter<TodoBlocState> emit) {
    emit(state.copyWith(todoList: newList));  // Emitter로 발행
  }
}

// Riverpod: 그냥 메서드 쓰면 됨
class TodoDataHolder extends StateNotifier<List<Todo>> {
  void addTodo() {
    state = List.of(state);  // state 재할당이 곧 발행
  }
}
```

Event 클래스 정의 → 핸들러 등록 → Emitter 전달 → emit 호출. 이 4단계가 **state 재할당 한 줄**로 줄었다.

### Extension으로 접근 축약

```dart
extension TodoListHolderProvider on WidgetRef {
  TodoDataHolder get readTodoHolder => read(todoDataProvider.notifier);
}
```

`ref.read(todoDataProvider.notifier).addTodo()` 대신 `ref.readTodoHolder.addTodo()`로 쓸 수 있다. BLoC에서 `context.readTodoBloc`으로 축약했던 것과 같은 패턴이다.

## ConsumerWidget — UI에서 사용

### ProviderScope 설정

```dart
// s_main.dart
class MainScreenWrapper extends StatelessWidget {
  const MainScreenWrapper({super.key});

  @override
  Widget build(BuildContext context) {
    return const ProviderScope(child: MainScreen());
  }
}
```

`ProviderScope`은 Riverpod의 DI 컨테이너다. 이 안에 있는 위젯들이 Provider에 접근할 수 있다. BLoC의 `BlocProvider`와 같은 역할이지만, 여러 Provider를 개별적으로 감쌀 필요 없이 하나의 `ProviderScope`만 있으면 된다.

### ConsumerWidget으로 구독

```dart
// w_todo_list.dart
class TodoList extends ConsumerWidget {
  const TodoList({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todoList = ref.watch(todoDataProvider);  // 구독

    return todoList.isEmpty
        ? const Expanded(
            child: Center(
              child: Text('할 일을 작성해 보세요.',
                  style: TextStyle(fontSize: 32)),
            ),
          )
        : Column(
            children: todoList.map((e) => TodoItem(e)).toList(),
          );
  }
}
```

BLoC에서는 `BlocBuilder<TodoBloc, TodoBlocState>` 위젯으로 감싸야 했는데, Riverpod은 `ConsumerWidget`을 상속하면 `build` 메서드에 `WidgetRef ref`가 들어온다. `ref.watch(provider)`로 구독하면 state가 바뀔 때마다 자동 리빌드된다.

### ref.watch vs ref.read

```dart
// w_todo_item.dart
class TodoItem extends ConsumerWidget {
  final Todo todo;

  const TodoItem(this.todo, {super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Dismissible(
      key: ValueKey(todo.id),
      onDismissed: (direction) {
        ref.readTodoHolder.removeTodo(todo);  // ref.read: 한 번 읽기
      },
      child: RoundedContainer(
        child: Row(
          children: [
            TodoStatusWidget(todo),
            Text(todo.title),
            const Spacer(),
            IconButton(
              onPressed: () {
                ref.readTodoHolder.editTodo(todo);  // ref.read: 이벤트용
              },
              icon: const Icon(EvaIcons.editOutline),
            )
          ],
        ),
      ),
    );
  }
}
```

| 메서드 | 용도 | 리빌드 |
|--------|------|--------|
| `ref.watch(provider)` | 상태 구독 (UI 그릴 때) | O |
| `ref.read(provider.notifier)` | 메서드 호출 (이벤트용) | X |

BLoC의 `context.watch` / `context.read`와 정확히 같은 개념이다. 차이라면 BLoC은 `context`에서 꺼내고, Riverpod은 `ref`에서 꺼낸다.

### ConsumerStatefulWidget

Rive 애니메이션처럼 lifecycle 메서드가 필요하면 `ConsumerStatefulWidget`을 쓴다:

```dart
class TodoStatusWidget extends ConsumerStatefulWidget {
  final Todo todo;

  const TodoStatusWidget(this.todo, {super.key});

  @override
  ConsumerState<TodoStatusWidget> createState() => _TodoStatusWidgetState();
}

class _TodoStatusWidgetState extends ConsumerState<TodoStatusWidget> {
  // initState, dispose 등 lifecycle 사용 가능
  // ref도 사용 가능

  @override
  Widget build(BuildContext context) {
    return Tap(
      onTap: () {
        ref.readTodoHolder.changeTodoStatus(widget.todo);
      },
      child: SizedBox(
        width: 50,
        height: 50,
        child: switch (widget.todo.status) {
          TodoStatus.complete => Checkbox(value: true, onChanged: null),
          TodoStatus.inComplete => const Checkbox(value: false, onChanged: null),
          TodoStatus.onGoing => _riveReady
              ? RiveWidget(controller: _controller!, fit: Fit.cover)
              : const SizedBox(),
        },
      ),
    );
  }
}
```

| 위젯 타입 | BLoC 대응 | 언제 쓰나 |
|-----------|-----------|----------|
| `ConsumerWidget` | `StatelessWidget` + `BlocBuilder` | 단순 UI |
| `ConsumerStatefulWidget` | `StatefulWidget` + `BlocBuilder` | lifecycle 필요 |

## ProviderScope — 스코프 개념

Provider가 전역으로 선언되었다고 데이터도 전역으로 같이 쓰는 게 아니다. `ProviderScope`으로 나눠주면 별도의 데이터를 가지게 된다.

```dart
// 같은 Provider인데 Scope가 다르면 다른 데이터
ProviderScope(
  overrides: [todoDataProvider.overrideWith(() => TodoDataHolder())],
  child: ScreenA(),  // 이 안에서는 독립된 TodoDataHolder
)

ProviderScope(
  overrides: [todoDataProvider.overrideWith(() => TodoDataHolder())],
  child: ScreenB(),  // 여기도 독립된 TodoDataHolder
)
```

Ch04에서 다뤘던 Scoped Model의 진짜 실현이다. BLoC에서는 `BlocProvider`를 위젯 트리 특정 위치에 넣어서 스코프를 만들었는데, Riverpod은 `ProviderScope`의 `overrides`로 더 유연하게 스코프를 지정할 수 있다.

유튜브 앱에서 영상마다 다른 댓글 리스트를 보여줘야 한다면:

```dart
// BLoC: 각 화면에 BlocProvider를 별도로 감싸야 함
BlocProvider(
  create: (_) => CommentBloc(videoId: 'abc123'),
  child: CommentSection(),
)

// Riverpod: ProviderScope로 override
ProviderScope(
  overrides: [commentProvider.overrideWith(() => CommentNotifier(videoId: 'abc123'))],
  child: CommentSection(),
)
```

사실 결과적으로 비슷하지만, Riverpod은 Provider 선언이 전역이니까 어떤 Provider들이 있는지 한눈에 파악이 된다. BLoC은 각 화면의 `BlocProvider`를 찾아다녀야 한다.

## FutureProvider — 비동기 상태의 끝판왕

Todo 앱에서는 안 썼지만, 실제 앱을 만들면 API 호출이 필수다. 여기서 Riverpod의 진가가 나온다.

### 기본 사용법

```dart
final userProvider = FutureProvider<User>((ref) async {
  final response = await http.get(Uri.parse('https://api.com/user'));
  return User.fromJson(jsonDecode(response.body));
});
```

이걸 뷰에서 쓰면:

```dart
class UserPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProvider);

    return Scaffold(
      body: Center(
        child: userAsync.when(
          loading: () => CircularProgressIndicator(),
          error: (e, _) => Text('에러: $e'),
          data: (user) => Text(user.name),
        ),
      ),
    );
  }
}
```

`.when` 하나로 로딩/에러/데이터 분기가 끝난다. 세 가지 상태를 빠뜨릴 수가 없어서 안전하기도 하다. Swift에서 비슷하게 하려면:

```swift
// Swift - 직접 분기 처리
var body: some View {
    if isLoading {
        ProgressView()
    } else if let error {
        Text("에러: \(error)")
    } else if let user {
        Text(user.name)
    }
}
```

이걸 `.when` 하나로 끝내는 거다.

### 화면 일부만 분기하기

`.when`이 화면 전체를 교체하는 것만은 아니다. 부분에도 쓸 수 있다:

```dart
Scaffold(
  appBar: AppBar(title: Text('마이페이지')),  // 고정
  body: Column(
    children: [
      Text('항상 보이는 영역'),  // 고정

      // 이 부분만 로딩/에러/데이터 처리
      ref.watch(userProvider).when(
        loading: () => CircularProgressIndicator(),
        error: (e, _) => Text('로드 실패'),
        data: (user) => UserCard(user: user),
      ),

      Text('여기도 항상 보임'),  // 고정
    ],
  ),
)
```

프로필 페이지, 상품 목록, 검색 결과 같은 데서 엄청 자주 쓰는 패턴이다. `loading`이나 `error` 분기에도 커스텀 위젯을 자유롭게 넣을 수 있다:

```dart
ref.watch(userProvider).when(
  loading: () => ShimmerSkeleton(),  // 스켈레톤 UI
  error: (e, _) => ErrorRetryWidget(
    message: '불러오기 실패',
    onRetry: () => ref.invalidate(userProvider),  // 재시도
  ),
  data: (user) => UserProfileCard(user: user),
)
```

### ref.listen — 오버레이 로딩

`.when`은 화면의 특정 영역을 교체하는 방식이다. 기존 화면 위에 오버레이로 로딩을 띄우고 싶다면 `ref.listen`을 쓴다:

```dart
class LoginPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 상태 변화를 '듣고' 오버레이 처리
    ref.listen(loginProvider, (prev, next) {
      if (next.isLoading) {
        showDialog(
          context: context,
          builder: (_) => Center(child: CircularProgressIndicator()),
        );
      } else {
        Navigator.of(context).pop();  // 로딩 닫기
        next.whenOrNull(
          error: (e, _) => ScaffoldMessenger.of(context)
              .showSnackBar(SnackBar(content: Text('실패: $e'))),
          data: (user) => Navigator.pushReplacement(...),
        );
      }
    });

    // 메인 UI는 그대로
    return Scaffold(
      body: Column(
        children: [
          TextField(...),
          ElevatedButton(
            onPressed: () => ref.read(loginProvider.notifier).login(),
            child: Text('로그인'),
          ),
        ],
      ),
    );
  }
}
```

| 상황 | 방식 |
|------|------|
| 화면 영역이 로딩/에러/데이터로 교체 | `ref.watch` + `.when` |
| 기존 화면 위에 오버레이 로딩 | `ref.listen` + `showDialog` |
| 데이터만 필요하고 로딩 무시 | `.valueOrNull` |

`when`은 화면 교체용, `listen`은 이벤트 반응용이다.

## @riverpod — 코드 제너레이션

지금까지 `StateNotifierProvider<TodoDataHolder, List<Todo>>((ref) => ...)` 같은 긴 타입을 직접 썼는데, `@riverpod`을 쓰면 이 보일러플레이트도 자동 생성된다:

```dart
// @riverpod 방식: 함수 쓰듯이 로직만 짜면 끝
@riverpod
Future<User> user(ref) async {
  final response = await http.get(Uri.parse('https://api.com/user'));
  return User.fromJson(jsonDecode(response.body));
}

// build_runner 돌리면 userProvider가 자동 생성됨
// 뷰에서 ref.watch(userProvider).when(...) 으로 사용
```

알아야 하는 건 3개뿐이다:
1. `@riverpod` 붙이고 로직 짜기
2. `build_runner` 돌려서 코드 생성
3. 뷰에서 `ref.watch` / `ref.read`로 사용

`StateNotifierProvider<TodoNotifier, List<Todo>>((ref) => ...)` 이런 보일러플레이트를 직접 안 짜도 된다. 요즘 새 프로젝트는 대부분 `@riverpod` 방식을 쓴다.

## BLoC vs Riverpod 최종 비교

### 코드 비교

| 항목 | BLoC | Riverpod |
|------|------|----------|
| 방식 | Event → Bloc → State | 직접 메서드 호출 |
| 이벤트 정의 | `sealed class TodoEvent` + 하위 클래스 4개 | 없음 |
| State 래퍼 | `@freezed TodoBlocState` (BlocStatus 포함) | 직접 `List<Todo>` |
| 등록 | `BlocProvider` (위젯 트리) | `StateNotifierProvider` (전역 선언) |
| UI 위젯 | `BlocBuilder` 래퍼 | `ConsumerWidget` 상속 |
| 상태 발행 | `emit(state.copyWith(...))` | `state = newValue` |
| 비동기 분기 | 직접 처리 | `.when(loading, error, data)` |
| Event 추적 | BlocObserver로 가능 | 직접 로깅 필요 |
| 보일러플레이트 | 많음 | 적음 |
| 스코프 | BlocProvider 위치 | ProviderScope overrides |

### 같은 "Todo 삭제" 동작

```dart
// BLoC: Event 정의 → 핸들러 등록 → Event dispatch
// 1. Event 정의
class TodoRemoveEvent extends TodoEvent {
  final Todo removedTodo;
  TodoRemoveEvent(this.removedTodo);
}

// 2. 핸들러 등록
on<TodoRemoveEvent>(_removeTodo);

// 3. 핸들러 구현
void _removeTodo(TodoRemoveEvent event, Emitter<TodoBlocState> emit) {
  final oldTodoList = List<Todo>.from(state.todoList);
  oldTodoList.removeWhere((e) => e.id == event.removedTodo.id);
  emit(state.copyWith(todoList: oldTodoList));
}

// 4. UI에서 dispatch
context.readTodoBloc.add(TodoRemoveEvent(todo));


// Riverpod: 메서드 하나
// 1. 메서드 구현
void removeTodo(Todo todo) {
  state.remove(todo);
  state = List.of(state);
}

// 2. UI에서 호출
ref.readTodoHolder.removeTodo(todo);
```

BLoC은 4단계, Riverpod은 2단계. 추적성 vs 생산성의 트레이드오프다.

### 언제 뭘 쓸까

| 상황 | 추천 |
|------|------|
| 대규모 팀 (10명+) | BLoC — Event 히스토리 추적, 팀 컨벤션 강제 |
| 소규모 팀 / 개인 | Riverpod — 생산성, 적은 보일러플레이트 |
| API 호출 많은 앱 | Riverpod — `.when` 패턴이 압도적 |
| Event 로깅 필수 | BLoC — BlocObserver |
| 프로토타입 | Riverpod (또는 GetX) |

2026년 기준으로 새 프로젝트는 대부분 Riverpod이 기본 선택이고, BLoC은 대규모 엔터프라이즈에서 쓰이는 추세다.

## 4가지 방식 총 비교

Ch05 전체를 통해 같은 Todo 앱을 4가지 방식으로 만들어봤다:

| | ValueNotifier | GetX | BLoC | Riverpod |
|--|-------------|------|------|----------|
| 방식 | Scoped (내장) | Static | Scoped | Scoped |
| 상태 변경 | `notifyListeners()` | `.obs` 자동 | `emit()` | `state =` 재할당 |
| UI 구독 | `ValueListenableBuilder` | `Obx` | `BlocBuilder` | `ref.watch` |
| 비동기 분기 | 직접 처리 | 직접 처리 | 직접 처리 | `.when` 자동 |
| 보일러플레이트 | 중간 | 적음 | 많음 | 적음 |
| 추적성 | 낮음 | 낮음 | 높음 | 중간 |
| 메모리 관리 | 자동 | 수동 | 자동 | 자동 |

## Riverpod 3.0 — 2025년 9월 기준 변경점

이 글에서 쓴 코드는 Riverpod 2.x 기반이다. 2025년 9월에 Riverpod 3.0이 나왔는데, 핵심 개념(`ref.watch`, `ref.read`, `.when`, `ProviderScope`)은 그대로고 보일러플레이트가 더 줄었다.

| | Riverpod 2.x (이 글) | Riverpod 3.0 |
|--|---|---|
| Provider 선언 | `StateNotifierProvider<T, S>((ref) => ...)` 직접 작성 | `@riverpod` 붙이면 자동 생성 |
| 상태 클래스 | `StateNotifier<T>` | `Notifier<T>` (`StateNotifier` deprecated) |
| 비동기 | `FutureProvider` 별도 선언 | `@riverpod Future<T>` 함수로 통일 |
| Mutation | 없음 | `@mutation`으로 폼 제출/버튼 액션의 로딩/에러 자동 관리 (experimental) |
| AsyncValue | 일반 클래스 | `sealed class` — 패턴 매칭 시 빠뜨리면 컴파일 에러 |

3.0에서 가장 큰 변화는 `StateNotifier`가 `Notifier`로 대체된 것과, `@riverpod` 코드 제너레이션이 기본이 된 것이다:

```dart
// 2.x: 직접 선언
final todoDataProvider = StateNotifierProvider<TodoDataHolder, List<Todo>>(
    (ref) => TodoDataHolder());

class TodoDataHolder extends StateNotifier<List<Todo>> {
  TodoDataHolder() : super([]);
  // ...
}

// 3.0: @riverpod + Notifier
@riverpod
class TodoDataHolder extends _$TodoDataHolder {
  @override
  List<Todo> build() => [];  // 초기 상태

  void addTodo(Todo todo) {
    state = [...state, todo];
  }
}
// build_runner가 todoDataHolderProvider 자동 생성
```

Mutation API는 아직 experimental이라 API가 바뀔 수 있지만, 폼 제출 같은 액션의 로딩/성공/에러 상태를 자동으로 관리해주는 기능이다. 정식 출시되면 `ref.listen`으로 수동 처리하던 부분이 더 간결해질 예정이다.

## 내가 느낀 점

BLoC의 Event 기반 구조가 사실상 Scope 역할을 하는 거라고 느꼈다. 어떤 이벤트가 어떤 상태를 바꾸는지 명확히 분리되니까. 근데 Riverpod은 Provider 단위로 관심사를 나누면서도 코드량이 훨씬 적다.

```dart
// 관심사 분리: Provider 단위로 나누면 됨
final todoDataProvider = StateNotifierProvider<TodoDataHolder, List<Todo>>(...);
final userProvider = FutureProvider<User>(...);
final settingsProvider = StateNotifierProvider<SettingsNotifier, Settings>(...);
```

젤 좋은 건 뷰에서 상태별 화면 분기가 `.when` 하나로 되는 점이다. 로딩/에러/데이터를 빠뜨릴 수 없게 강제하니까 안전하고, 각 분기에 아무 위젯이나 넣을 수 있으니까 커스텀도 자유롭다.

앱이 커져서 상태가 많아지면 Provider를 나누면 되고, Scope가 필요하면 `ProviderScope`의 `overrides`로 해결된다. 개인 프로젝트에서 쓴다면 Riverpod이 정답인 것 같다.
