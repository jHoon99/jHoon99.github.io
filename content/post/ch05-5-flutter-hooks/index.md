---
title: "Ch05-5. Flutter Hooks + Riverpod - SwiftUI @State가 여기 있었네"
slug: "ch05-5-flutter-hooks"
date: 2026-05-02
weight: 2
draft: false
tags: ["Flutter", "Dart", "flutter_hooks", "Riverpod", "HookConsumerWidget", "useState", "useEffect"]
categories: ["Flutter"]
description: "StatefulWidget의 보일러플레이트에서 벗어나 Flutter Hooks의 useState, useEffect, 자동 dispose 패턴을 정리하고, Riverpod과 조합하는 방법까지 다룬 기록"
---

강의에서 Hooks 부분은 눈으로만 봤는데, SwiftUI의 `@State`랑 너무 비슷해서 정리하지 않으면 아까울 것 같았다. StatefulWidget의 보일러플레이트가 싫었던 사람이라면 Hooks를 보는 순간 "이걸 왜 이제 알았지" 싶을 거다.

## StatefulWidget, 뭐가 불편한가

간단한 카운터 하나 만드는데 이만큼 써야 한다:

```dart
class CounterPage extends StatefulWidget {
  const CounterPage({super.key});

  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(child: Text('$_count')),
      floatingActionButton: FloatingActionButton(
        onPressed: () => setState(() => _count++),
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

`StatefulWidget` 클래스 + `State` 클래스, 2개를 만들어야 한다. `createState()` 보일러플레이트도 매번 써야 하고. 상태 변수가 하나인데도 이 정도다. TextEditingController나 AnimationController가 들어오면 `initState`에서 초기화하고 `dispose`에서 해제하는 코드까지 추가된다.

React에서 이 문제를 Hooks로 해결했고, **Riverpod을 만든 Remi Rousselet**이 Flutter 버전으로 `flutter_hooks`를 만들었다.

## HookWidget — 2개 클래스가 1개로

같은 카운터를 Hooks로 바꾸면:

```dart
class CounterPage extends HookWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    final count = useState(0);

    return Scaffold(
      body: Center(child: Text('${count.value}')),
      floatingActionButton: FloatingActionButton(
        onPressed: () => count.value++,
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

`StatefulWidget` + `State` 2개 클래스 → `HookWidget` 1개. `createState()` 보일러플레이트 사라짐. 상태 선언이 `build` 메서드 안에서 한 줄로 끝난다.

SwiftUI 개발자라면 바로 느낌 올 거다:

```swift
// SwiftUI
struct CounterPage: View {
    @State private var count = 0

    var body: some View {
        Text("\(count)")
        Button("+") { count += 1 }
    }
}
```

`@State`가 `useState`고, `.value`로 접근하는 것만 다르다.

## 핵심 Hooks

### useState — 로컬 상태 관리

```dart
final count = useState(0);        // int
final name = useState('');         // String
final isLoading = useState(false); // bool
```

`useState<T>(initialValue)`는 `ValueNotifier<T>`를 내부적으로 만들고, `.value`가 바뀌면 위젯을 자동으로 리빌드한다. `setState(() { ... })`를 직접 호출할 필요가 없다.

| Flutter Hooks | SwiftUI | 설명 |
|---------------|---------|------|
| `useState(0)` | `@State var count = 0` | 로컬 상태 선언 |
| `count.value` | `count` | 값 읽기 |
| `count.value++` | `count += 1` | 값 변경 → 자동 리빌드 |

### useEffect — 생명주기 처리

`initState` + `dispose`를 하나로 합친 것이다:

```dart
@override
Widget build(BuildContext context) {
  // 마운트 시 실행, 리턴 함수는 dispose 시 실행
  useEffect(() {
    final subscription = stream.listen(print);
    return subscription.cancel;  // cleanup = dispose
  }, []);  // [] = 마운트 시 1회만

  // 특정 값이 바뀔 때마다 실행
  final userId = useState(1);
  useEffect(() {
    fetchUser(userId.value);
    return null;  // cleanup 불필요
  }, [userId.value]);  // userId가 바뀔 때마다

  return ...;
}
```

두 번째 인자(keys)가 핵심이다:

| keys | 동작 | SwiftUI 대응 |
|------|------|-------------|
| `[]` | 마운트 시 1회 | `.onAppear` |
| `[value]` | value 변경 시 | `.onChange(of: value)` |
| 생략 | 매 빌드마다 | - |
| return 함수 | 위젯 제거 시 실행 | `.onDisappear` |

### useMemoized — 비싼 연산 캐싱

```dart
// 매 빌드마다 재계산되는 걸 방지
final expensive = useMemoized(() => heavyComputation(data), [data]);
```

keys가 바뀔 때만 재계산한다. SwiftUI의 캐싱은 뷰 자체가 struct라서 자동으로 되는 부분이 있는데, Flutter는 `build`가 매번 호출되니까 직접 메모이제이션해야 한다.

### Controller 계열 — 자동 dispose가 핵심

StatefulWidget에서 Controller를 쓰면 항상 이 패턴이 반복된다:

```dart
// StatefulWidget 방식: 초기화 + 해제를 직접 관리
class _MyState extends State<MyWidget> {
  late TextEditingController _controller;
  late AnimationController _animController;

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController();
    _animController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 300),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    _animController.dispose();
    super.dispose();
  }
}
```

Hooks로 바꾸면:

```dart
// Hooks: 한 줄이면 끝, dispose 자동
@override
Widget build(BuildContext context) {
  final controller = useTextEditingController();
  final animController = useAnimationController(
    duration: const Duration(milliseconds: 300),
  );
  final tabController = useTabController(initialLength: 3);
  final focusNode = useFocusNode();
  final scrollController = useScrollController();

  return ...;
}
```

| Hook | 대체하는 것 | 자동 dispose |
|------|-----------|-------------|
| `useTextEditingController()` | `TextEditingController` + dispose | O |
| `useAnimationController()` | `AnimationController` + TickerProvider + dispose | O |
| `useTabController()` | `TabController` + dispose | O |
| `useFocusNode()` | `FocusNode` + dispose | O |
| `useScrollController()` | `ScrollController` + dispose | O |

`initState`에서 만들고 `dispose`에서 해제하는 그 반복 패턴이 완전히 사라진다. 특히 `useAnimationController`는 `TickerProviderStateMixin`도 자동으로 처리해주니까 `with SingleTickerProviderStateMixin` 같은 mixin도 필요 없다.

### useRef — 리빌드 없이 값 유지

```dart
final renderCount = useRef(0);
renderCount.value++;  // 이걸 바꿔도 리빌드 안 됨
```

`useState`와 달리 값이 바뀌어도 리빌드를 트리거하지 않는다. 렌더링 횟수 추적, 이전 값 기억 같은 용도로 쓴다.

## Hook 규칙 3가지

React Hooks와 같은 규칙이다. 어기면 버그가 난다:

### 1. 조건문 안에서 호출 금지

```dart
// 나쁜 예
if (isLoggedIn) {
  final name = useState('');  // 조건에 따라 Hook 호출 순서가 바뀜
}

// 좋은 예
final name = useState('');
if (isLoggedIn) {
  // name.value 사용
}
```

### 2. 항상 같은 순서로 호출

```dart
// 항상 이 순서대로 호출돼야 함
final count = useState(0);
final name = useState('');
final isLoading = useState(false);
```

Hooks는 내부적으로 호출 순서로 상태를 추적한다. 순서가 바뀌면 엉뚱한 값이 매칭된다.

### 3. `use` 접두어

커스텀 Hook을 만들 때 `use`로 시작해야 한다. 컨벤션이자 Hooks임을 표시하는 약속이다.

## Custom Hook 만들기

반복되는 Hook 조합을 함수로 추출하면 된다:

```dart
// 디바운스된 값을 반환하는 커스텀 Hook
ValueNotifier<T> useDebounced<T>(T value, {Duration delay = const Duration(milliseconds: 500)}) {
  final debounced = useState(value);

  useEffect(() {
    final timer = Timer(delay, () => debounced.value = value);
    return timer.cancel;  // 이전 타이머 정리
  }, [value, delay]);

  return debounced;
}

// 사용
@override
Widget build(BuildContext context) {
  final searchText = useState('');
  final debouncedText = useDebounced(searchText.value);

  useEffect(() {
    searchApi(debouncedText.value);
    return null;
  }, [debouncedText.value]);

  return TextField(onChanged: (v) => searchText.value = v);
}
```

## 꿀팁 패턴들

### useMemoized + useFuture — API 캐싱

```dart
@override
Widget build(BuildContext context) {
  // useMemoized로 Future를 캐싱 → 리빌드해도 API 재호출 안 함
  final future = useMemoized(() => fetchUserData(), []);
  final snapshot = useFuture(future);

  if (snapshot.connectionState == ConnectionState.waiting) {
    return const CircularProgressIndicator();
  }

  if (snapshot.hasError) {
    return Text('에러: ${snapshot.error}');
  }

  return Text('${snapshot.data?.name}');
}
```

`useMemoized` 없이 `useFuture(fetchUserData())`를 쓰면 매 빌드마다 새 Future가 만들어져서 API가 무한 호출된다. `useMemoized`로 Future 자체를 캐싱하는 게 핵심이다.

### useDebounced — 검색 디바운싱

위의 Custom Hook 예제가 바로 이 패턴이다. 검색창에서 타이핑할 때마다 API를 호출하지 않고, 사용자가 타이핑을 멈춘 뒤 500ms 후에 한 번만 호출한다. 실무에서 검색 자동완성에 거의 필수다.

## Riverpod + Hooks 조합

여기서 진짜 빛난다. `hooks_riverpod` 패키지의 `HookConsumerWidget`을 쓰면 **Hooks(로컬 상태)** + **Riverpod(공유 상태)**를 한 위젯에서 쓸 수 있다.

### 역할 분리

| 역할 | 도구 | 예시 |
|------|------|------|
| 로컬 UI 상태 | Hooks (`useState`, `useAnimationController`) | 텍스트 입력, 애니메이션, 토글 |
| 공유 비즈니스 상태 | Riverpod (`ref.watch`, `ref.read`) | 유저 데이터, Todo 목록, API |

### 코드 예제

```dart
// HookConsumerWidget = HookWidget + ConsumerWidget
class TodoPage extends HookConsumerWidget {
  const TodoPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Hooks: 로컬 상태
    final controller = useTextEditingController();
    final focusNode = useFocusNode();
    final isExpanded = useState(false);

    // Riverpod: 공유 상태
    final todoList = ref.watch(todoDataProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Todo')),
      body: Column(
        children: [
          // 로컬 상태: 입력 필드 (이 화면에서만 필요)
          TextField(
            controller: controller,
            focusNode: focusNode,
            decoration: const InputDecoration(hintText: '할 일 입력'),
          ),
          ElevatedButton(
            onPressed: () {
              // Riverpod: 공유 상태 변경
              ref.read(todoDataProvider.notifier).addTodo(
                Todo(title: controller.text),
              );
              controller.clear();
              focusNode.requestFocus();
            },
            child: const Text('추가'),
          ),
          // Riverpod: 공유 상태 구독
          Expanded(
            child: ListView.builder(
              itemCount: todoList.length,
              itemBuilder: (context, index) => ListTile(
                title: Text(todoList[index].title),
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

`ConsumerStatefulWidget`에서는 `TextEditingController` 초기화 + dispose를 직접 했어야 했다. `HookConsumerWidget`에서는 `useTextEditingController()` 한 줄이면 끝이다. Riverpod의 `ref.watch`/`ref.read`도 그대로 쓸 수 있다.

### 위젯 타입 정리

| 위젯 | 상속 대상 | Hooks | Riverpod |
|------|---------|-------|----------|
| `StatelessWidget` | - | X | X |
| `HookWidget` | `StatelessWidget` | O | X |
| `ConsumerWidget` | `StatelessWidget` | X | O |
| `HookConsumerWidget` | `StatelessWidget` | O | O |

`HookConsumerWidget` 하나면 로컬 상태도 공유 상태도 다 된다. 실무에서 가장 많이 쓰는 조합이다.

## 생산성 라이브러리

| 패키지 | 역할 | 비고 |
|--------|------|------|
| `flutter_hooks` | 핵심 Hooks (`useState`, `useEffect` 등) | Remi Rousselet 제작 |
| `hooks_riverpod` | Hooks + Riverpod 조합 (`HookConsumerWidget`) | Riverpod 공식 |
| `flutter_use` | 추가 Hooks 모음 (`useDebounce`, `usePrevious` 등) | 커뮤니티 |

`flutter_hooks` + `hooks_riverpod`만 있으면 거의 모든 상황을 커버한다. `flutter_use`는 자주 쓰는 패턴을 미리 만들어둔 편의 패키지다.

## SwiftUI와 최종 비교

| 개념 | Flutter Hooks | SwiftUI |
|------|-------------|---------|
| 로컬 상태 | `useState` | `@State` |
| 값 접근 | `.value` | 직접 접근 |
| 생명주기 진입 | `useEffect(() {}, [])` | `.onAppear` |
| 값 변경 감지 | `useEffect(() {}, [value])` | `.onChange(of: value)` |
| 정리/해제 | `useEffect`의 return | `.onDisappear` |
| 컨트롤러 관리 | `useTextEditingController()` 등 (자동 dispose) | `@StateObject` (자동 관리) |
| 비싼 연산 캐싱 | `useMemoized` | 뷰가 struct라 자동 최적화 |
| 리빌드 없는 값 | `useRef` | 일반 `let` 변수 |
| 공유 상태 | Riverpod (`ref.watch`) | `@EnvironmentObject`, `@Observable` |

SwiftUI는 프로퍼티 래퍼(`@State`, `@Binding`, `@StateObject`)로 선언형 상태를 관리하고, Flutter Hooks는 `use` 함수로 같은 역할을 한다. 접근 방식은 다르지만 "선언적으로 상태를 관리하고, 변경 시 UI가 자동으로 반영된다"는 핵심은 동일하다.

## 내가 느낀 점

SwiftUI에서 `@State` 쓰던 경험이 있어서 `useState`가 바로 이해됐다. "아, 이거 `@State`랑 같은 거잖아." 그리고 `useTextEditingController`처럼 Controller를 자동으로 dispose해주는 게 진짜 편하다. StatefulWidget에서 `initState` + `dispose` 짝 맞추는 거 매번 귀찮았는데, 이게 한 줄로 끝나니까.

Riverpod과 조합하면 역할 분리가 깔끔해진다:
- **로컬 UI 상태** (텍스트 입력, 애니메이션, 토글) → Hooks
- **공유 비즈니스 상태** (Todo 목록, 유저 데이터, API) → Riverpod

`ConsumerStatefulWidget`에서 하던 걸 `HookConsumerWidget`으로 바꾸면 코드가 절반으로 줄어든다. StatefulWidget의 보일러플레이트에서 해방된 느낌이다.
