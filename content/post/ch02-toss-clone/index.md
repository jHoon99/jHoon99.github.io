---
title: "Ch02. 토스 앱 클론 - 실전 UI와 상태관리"
slug: "ch02-toss-clone"
date: 2026-04-23
draft: false
tags: ["Flutter", "GetX", "BottomNavigationBar", "CustomScrollView", "Theme", "Animation", "Toss Clone"]
categories: ["Flutter"]
description: "토스 앱 클론코딩으로 멀티탭 네비게이션, GetX 상태관리, 다크모드 테마 시스템, 검색 자동완성, 애니메이션까지 실전 Flutter UI를 구현한 기록"
---

Ch01에서 기본기를 다졌으니 이번엔 토스 앱을 클론하면서 실전 UI를 만들어봤다. 멀티탭 네비게이션, 상태관리, 테마 시스템처럼 실제 앱에서 꼭 필요한 것들 위주로 정리한다.

## 멀티탭 네비게이션 - IndexedStack

토스처럼 하단 5개 탭을 만들 때 가장 먼저 고민되는 게 "탭을 전환할 때 이전 탭의 상태를 유지할 것인가"다. SwiftUI에서는 `TabView` 안에 각각 `NavigationStack`을 넣으면 알아서 상태가 유지되는데, Flutter는 직접 해줘야 한다.

여기서 쓰는 게 `IndexedStack`이다. [Flutter 공식 문서](https://api.flutter.dev/flutter/widgets/IndexedStack-class.html)에 따르면, `Stack`의 서브클래스로 `index`에 해당하는 자식만 화면에 그리되 **나머지 자식도 메모리에 유지**한다. 즉 탭을 왔다갔다 해도 스크롤 위치나 입력 값이 그대로 남아있다.

```dart
class MainScreen extends StatefulWidget {
  @override
  State<MainScreen> createState() => _MainScreenState();
}

class _MainScreenState extends State<MainScreen> {
  int _tabIndex = 0;
  final _pages = [HomePage(), StockPage(), BenefitPage()];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(
        index: _tabIndex,
        children: _pages,
      ),
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _tabIndex,
        onTap: (i) => setState(() => _tabIndex = i),
        type: BottomNavigationBarType.fixed,
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: '홈'),
          BottomNavigationBarItem(icon: Icon(Icons.candlestick_chart), label: '주식'),
          BottomNavigationBarItem(icon: Icon(Icons.star), label: '혜택'),
        ],
      ),
    );
  }
}
```

`PageView`랑 비교하면 차이가 명확하다:

| | IndexedStack | PageView |
|--|--|--|
| 상태 유지 | 자동 (전부 메모리 유지) | `AutomaticKeepAliveClientMixin` 필요 |
| 전환 애니메이션 | 없음 (즉시 전환) | 스와이프 애니메이션 |
| 지연 로딩 | 안 됨 (전부 빌드) | 됨 (보이는 것만 빌드) |

탭이 3~5개 정도면 `IndexedStack`으로 충분한데, 탭 수가 많아지면 메모리를 잡아먹으니까 주의해야 한다.

### 탭마다 독립 네비게이션 스택

토스 앱에서 주식 탭에서 종목 상세 화면으로 들어갔다가 홈 탭으로 전환하고, 다시 주식 탭으로 돌아오면 종목 상세가 그대로 남아있다. 이걸 구현하려면 탭마다 별도의 `Navigator`를 줘야 한다.

```dart
// 각 탭에 독립 Navigator를 부여
class TabNavigator extends StatelessWidget {
  final GlobalKey<NavigatorState> navigatorKey;
  final Widget rootPage;

  const TabNavigator({required this.navigatorKey, required this.rootPage});

  @override
  Widget build(BuildContext context) {
    return Navigator(
      key: navigatorKey,
      onGenerateRoute: (_) => MaterialPageRoute(builder: (_) => rootPage),
    );
  }
}
```

`GlobalKey<NavigatorState>`로 각 탭의 네비게이터에 접근할 수 있어서, 뒤로가기 버튼을 누르면 현재 탭 내부에서만 pop이 일어난다. SwiftUI의 `TabView` 안에 각각 `NavigationStack`을 넣는 것과 같은 구조인데, Flutter는 `Navigator`를 직접 배치하는 거라 좀 더 수작업이 많다.

뒤로가기 처리도 `PopScope`로 직접 해야 한다. Flutter는 Android 14의 Predictive Back 제스처를 지원하기 위해 `WillPopScope`를 deprecated 시키고 [`PopScope`](https://api.flutter.dev/flutter/widgets/PopScope-class.html)로 교체했다. `canPop`으로 pop 가능 여부를 **미리** 선언하고, `onPopInvokedWithResult`에서 실제 처리를 한다.

```dart
PopScope(
  canPop: false,
  onPopInvokedWithResult: (didPop, _) {
    if (didPop) return;
    // 현재 탭에서 pop 가능하면 탭 내부 pop
    if (navigatorKeys[_tabIndex].currentState?.canPop() == true) {
      navigatorKeys[_tabIndex].currentState!.pop();
      return;
    }
    // 홈 탭이 아니면 홈으로 이동
    if (_tabIndex != 0) {
      setState(() => _tabIndex = 0);
    }
  },
  child: // ...
)
```

SwiftUI에서는 이런 뒤로가기 분기 처리를 할 일이 거의 없는데, Flutter에서는 Android 하드웨어 백 버튼 때문에 필수다.

## CustomScrollView와 Sliver

주식 탭처럼 스크롤하면 AppBar가 접히는 효과를 만들려면 `CustomScrollView` + `Sliver`를 써야 한다.

`Sliver`는 Flutter에서 스크롤 가능한 영역의 조각을 뜻한다. 일반 위젯이 `BoxConstraints`로 레이아웃하는 것과 달리, Sliver는 `SliverConstraints`라는 별도 프로토콜을 써서 **스크롤 위치에 따라 동적으로** 크기와 위치를 결정한다. [Flutter 공식 가이드](https://docs.flutter.dev/ui/layout/scrolling/slivers)에서 자세히 설명하고 있다.

왜 `ListView`로는 안 되냐면, `ListView`는 자체적으로 하나의 스크롤 컨텍스트를 만든다. 접히는 AppBar + 리스트 + 그리드를 하나의 스크롤로 묶으려면 같은 스크롤 컨텍스트를 공유해야 하는데, 그게 `CustomScrollView`다.

```dart
CustomScrollView(
  slivers: [
    SliverAppBar(
      pinned: true,          // 스크롤해도 AppBar 고정
      expandedHeight: 120,
      flexibleSpace: FlexibleSpaceBar(
        title: Text('주식'),
      ),
      actions: [
        IconButton(icon: Icon(Icons.search), onPressed: () {}),
        IconButton(icon: Icon(Icons.settings), onPressed: () {}),
      ],
    ),
    SliverToBoxAdapter(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Text('S&P 500  3,919.29', style: TextStyle(fontSize: 24)),
      ),
    ),
    SliverList(
      delegate: SliverChildBuilderDelegate(
        (context, index) => ListTile(
          title: Text(stockNames[index]),
          trailing: Text(stockPrices[index]),
        ),
        childCount: stockNames.length,
      ),
    ),
  ],
)
```

SwiftUI에서는 `ScrollView` 안에 `.toolbar`를 넣으면 접히는 효과가 자동으로 되는데, Flutter는 Sliver 조합으로 직접 구성해야 한다. 대신 세밀한 제어가 가능하다.

`SliverAppBar`의 옵션을 정리하면:

| 속성 | 동작 |
|------|------|
| `pinned: true` | 스크롤해도 AppBar가 상단에 고정 |
| `floating: true` | 위로 스크롤하면 AppBar가 바로 나타남 |
| `snap: true` | floating과 함께 쓰면 중간 상태 없이 딱 열리거나 닫힘 |
| `expandedHeight` | 완전히 펼쳤을 때 높이 |

`SliverToBoxAdapter`는 일반 위젯을 Sliver 안에 넣을 때 쓰는 어댑터다. Sliver 프로토콜을 모르는 일반 위젯(Text, Container 등)을 감싸서 `CustomScrollView`에 넣을 수 있게 해준다.

### TabBar + CustomScrollView 조합

주식 탭 안에 "내 주식" / "오늘의 발견" 같은 하위 탭을 넣을 때 주의할 점이 있다. 보통 `TabBar` + `TabBarView`를 쓰는데, `TabBarView`는 내부적으로 `PageView`를 사용하기 때문에 `CustomScrollView` 안에서 높이 계산 문제가 생긴다.

그래서 `TabBarView` 대신 `currentIndex`로 직접 전환하는 방식을 썼다:

```dart
class _StockPageState extends State<StockPage>
    with SingleTickerProviderStateMixin {
  late TabController _tabController;

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 2, vsync: this);
    _tabController.addListener(() => setState(() {}));
  }

  @override
  void dispose() {
    _tabController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return CustomScrollView(
      slivers: [
        SliverAppBar(pinned: true, title: Text('주식')),
        SliverToBoxAdapter(
          child: TabBar(
            controller: _tabController,
            tabs: [Tab(text: '내 주식'), Tab(text: '오늘의 발견')],
          ),
        ),
        SliverToBoxAdapter(
          child: _tabController.index == 0
              ? MyStockList()
              : TodayDiscoveryList(),
        ),
      ],
    );
  }
}
```

`TabController`의 `addListener`로 탭 변경을 감지하고 `setState`로 리빌드한다. `dispose`에서 컨트롤러 정리하는 것도 잊으면 안 된다.

## GetX 상태관리

검색 자동완성 기능에서 GetX를 써봤다. GetX는 상태관리 + 의존성 주입 + 라우팅을 한 패키지로 제공하는데, 여기서는 상태관리와 DI만 사용했다.

### 반응형 상태 - .obs와 Obx

GetX의 반응형 시스템은 `.obs`로 시작한다. 변수 뒤에 `.obs`를 붙이면 값 변경을 추적하는 반응형 타입(`Rx<T>`)이 된다. Provider의 `ChangeNotifier`나 BLoC의 `Stream`과 달리 GetX는 자체 반응형 시스템을 쓴다.

```dart
class SearchController extends GetxController {
  final results = <String>[].obs;          // RxList<String>
  final query = ''.obs;                    // RxString

  @override
  void onInit() {
    super.onInit();
    // debounce: 입력 멈추고 300ms 후에 검색 실행
    debounce(query, (_) => _doSearch(), time: 300.ms);
  }

  void updateQuery(String text) => query.value = text;

  void _doSearch() {
    if (query.value.isEmpty) {
      results.clear();
      return;
    }
    results.value = allItems
        .where((item) => item.contains(query.value))
        .toList();
  }
}
```

`Obx`로 감싸면 내부에서 사용하는 `.obs` 변수가 바뀔 때만 해당 위젯이 리빌드된다. `setState`처럼 전체를 다시 그리는 게 아니라 `Obx` 블록만 갱신하니까 효율적이다.

```dart
Obx(() => ListView.builder(
  itemCount: controller.results.length,
  itemBuilder: (_, i) => ListTile(title: Text(controller.results[i])),
))
```

하나 편한 점은, 같은 값으로 세팅하면 리빌드가 안 일어난다. 내부적으로 이전 값과 비교해서 실제로 바뀔 때만 UI를 갱신하는 거다.

### 의존성 주입 - Get.put과 Get.find

GetX의 DI는 전역 컨테이너 방식이다. `Get.put()`으로 등록하고 `Get.find()`로 어디서든 꺼내 쓴다.

```dart
// 화면 진입 시 등록
class _SearchScreenState extends State<SearchScreen> {
  @override
  void initState() {
    super.initState();
    Get.put(SearchController());
  }

  @override
  void dispose() {
    Get.delete<SearchController>();
    super.dispose();
  }
}

// 자식 위젯에서 사용
class ResultListView extends StatelessWidget {
  final controller = Get.find<SearchController>();

  @override
  Widget build(BuildContext context) {
    return Obx(() => ListView.builder(
      itemCount: controller.results.length,
      itemBuilder: (_, i) => ListTile(title: Text(controller.results[i])),
    ));
  }
}
```

SwiftUI 비교로 정리하면:

| GetX | SwiftUI | 역할 |
|------|---------|------|
| `GetxController` | `ObservableObject` | 상태 클래스 |
| `.obs` | `@Published` | 변경 감지 |
| `Obx(() =>)` | 자동 | UI 리빌드 |
| `Get.put()` | `.environmentObject()` | 등록 |
| `Get.find()` | `@EnvironmentObject` | 접근 |

차이는 `@EnvironmentObject`가 위젯 트리 안에서만 접근 가능한 반면, `Get.find()`는 `BuildContext` 없이 어디서든 전역 접근이 된다는 것이다. 편하긴 한데 남용하면 의존성 추적이 어려워진다.

### Mixin으로 접근 축약

여러 위젯에서 같은 컨트롤러를 `Get.find()`로 가져오는 게 반복되면 mixin으로 묶을 수 있다:

```dart
mixin SearchDataProvider {
  SearchController get searchData => Get.find<SearchController>();
}

class AutoCompleteList extends StatelessWidget with SearchDataProvider {
  @override
  Widget build(BuildContext context) {
    return Obx(() => ListView.builder(
      itemCount: searchData.results.length,
      itemBuilder: (_, i) => Text(searchData.results[i]),
    ));
  }
}
```

GetX에는 `GetView<T>`라는 내장 클래스도 있어서, 상속하면 `controller`로 바로 접근할 수 있다. 다만 하나의 컨트롤러 타입만 바인딩되니까 여러 컨트롤러가 필요하면 mixin이 낫다.

### GetX에 대한 솔직한 생각

GetX가 편한 건 맞는데, 커뮤니티에서 논쟁이 좀 있다. 메인테이너가 한 명이라 업데이트가 느리고, 전역 상태 접근이 너무 쉬워서 코드가 커지면 의존성 파악이 어려워진다는 비판이 있다. 요즘은 새 프로젝트에서 Riverpod을 많이 쓰는 추세인 것 같다. 나도 다음 프로젝트에서는 Riverpod을 써볼 예정이다.

## 다크모드 - InheritedWidget 기반 테마

Flutter 기본 `ThemeData`만으로도 다크모드를 할 수 있지만, 토스 앱처럼 커스텀 컬러가 많으면 `InheritedWidget`로 직접 테마 시스템을 만드는 게 편하다.

`InheritedWidget`은 위젯 트리를 통해 데이터를 하위로 전파하는 메커니즘이다. [Flutter 공식 문서](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)에 따르면, 내부적으로 각 Element에 `InheritedWidget` 해시 테이블을 유지해서 O(1)로 조회가 가능하다. `Provider` 패키지도 사실 이걸 감싼 래퍼다.

```dart
// 1. 색상 추상 클래스
abstract class AppColors {
  Color get background;
  Color get text;
  Color get card;
  Color get accent;
}

class LightColors implements AppColors {
  Color get background => Colors.white;
  Color get text => Colors.black87;
  Color get card => Colors.grey.shade100;
  Color get accent => Color(0xFF3182F6);  // 토스 블루
}

class DarkColors implements AppColors {
  Color get background => Color(0xFF1B1B1E);
  Color get text => Colors.white;
  Color get card => Color(0xFF2C2C2E);
  Color get accent => Color(0xFF5B9BF5);
}
```

```dart
// 2. InheritedWidget으로 트리에 전파
class ThemeHolder extends InheritedWidget {
  final AppColors appColors;
  final VoidCallback onToggle;

  const ThemeHolder({
    required this.appColors,
    required this.onToggle,
    required super.child,
  });

  static ThemeHolder of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<ThemeHolder>()!;
  }

  @override
  bool updateShouldNotify(ThemeHolder old) => appColors != old.appColors;
}
```

```dart
// 3. extension으로 축약
extension ThemeContext on BuildContext {
  AppColors get appColors => ThemeHolder.of(this).appColors;
  VoidCallback get toggleTheme => ThemeHolder.of(this).onToggle;
}

// 사용
Container(color: context.appColors.background)
Text('제목', style: TextStyle(color: context.appColors.text))
```

`Theme.of(context).colorScheme.primary` 이렇게 길게 쳐야 하는 걸 `context.appColors.accent`로 줄인 거다. SwiftUI의 `@Environment(\.colorScheme)`과 비슷한 역할이지만, Flutter는 `InheritedWidget` + extension 조합으로 직접 만드는 거라 초기 작업이 좀 필요하다.

테마 전환은 상위 `StatefulWidget`에서 `setState`로 `InheritedWidget`을 리빌드하면 된다. `updateShouldNotify`가 `true`를 반환하면 `dependOnInheritedWidgetOfExactType`을 호출한 모든 하위 위젯이 자동으로 리빌드된다.

## 애니메이션 - flutter_animate

Flutter 기본 애니메이션은 `AnimationController` + `StatefulWidget` + `TickerProviderStateMixin` + `dispose`까지 써야 해서 보일러플레이트가 상당하다. [`flutter_animate`](https://pub.dev/packages/flutter_animate) 패키지를 쓰면 한 줄로 끝난다.

```dart
// flutter_animate - 체이닝 API
Column(
  children: [
    Text('토스뱅크').animate().fadeIn(duration: 600.ms).slideY(begin: 0.3),
    BankAccountCard().animate().fadeIn(delay: 200.ms).slideX(begin: -0.1),
  ],
)
```

기본 `AnimationController` 방식이랑 비교하면:

```dart
// 기본 방식 - 이만큼 써야 한다
class _MyWidgetState extends State<MyWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _fadeAnim;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this, duration: 600.ms);
    _fadeAnim = Tween(begin: 0.0, end: 1.0).animate(_controller);
    _controller.forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return FadeTransition(opacity: _fadeAnim, child: Text('토스뱅크'));
  }
}
```

`flutter_animate`는 이 모든 걸 `.animate().fadeIn()` 한 줄로 처리한다. 효과를 체이닝하면 기본적으로 동시에 실행되고, `.then()`을 쓰면 순차 실행으로 바꿀 수 있다.

```dart
// 동시 실행: fade + slide가 같이
widget.animate().fadeIn().slide()

// 순차 실행: fade 끝나고 200ms 뒤에 slide
widget.animate()
  .fadeIn(duration: 500.ms)
  .then(delay: 200.ms)
  .slide()
```

SwiftUI의 `.transition(.opacity)`, `.animation(.easeInOut)` 같은 선언적 방식과 비슷한데, 체이닝이 가능해서 복합 애니메이션 조합이 더 유연한 편이다.

## 자주 만나는 레이아웃 에러들

Flutter 하면서 이것저것 삽질한 것들을 정리해둔다.

**1. Row/Column 안에서 Unbounded 에러**

```dart
// 에러: Row 안의 TextField가 너비를 못 잡음
Row(children: [TextField()])

// 해결: Expanded로 남은 공간 차지하게
Row(children: [Expanded(child: TextField())])
```

`Row`는 자식에게 무한 너비를 허용하는데 `TextField`는 가능한 너비를 전부 쓰려고 해서 충돌이 난다.

**2. 스크롤 안에 스크롤 - 높이 충돌**

```dart
// 에러: SingleChildScrollView 안에 ListView가 무한 높이 요구
SingleChildScrollView(
  child: Column(children: [ListView.builder(...)]),
)

// 해결: shrinkWrap + 스크롤 비활성화
ListView.builder(
  shrinkWrap: true,                        // 콘텐츠 높이만큼만
  physics: NeverScrollableScrollPhysics(), // 스크롤은 부모가
)
```

Ch01 인스타 클론에서도 나온 패턴인데, `shrinkWrap`은 성능에 영향을 주니까 아이템이 많으면 `CustomScrollView` + `SliverList`를 쓰는 게 맞다.

**3. GestureDetector 빈 영역 터치 안 됨**

```dart
// 안 됨: Spacer 영역은 터치 이벤트가 안 먹힘
GestureDetector(
  onTap: () {},
  child: Row(children: [Text('메뉴'), Spacer(), Icon(Icons.arrow_right)]),
)

// 해결: behavior 설정
GestureDetector(
  behavior: HitTestBehavior.opaque,  // 투명 영역도 터치 감지
  onTap: () {},
  child: Row(children: [Text('메뉴'), Spacer(), Icon(Icons.arrow_right)]),
)
```

기본적으로 `GestureDetector`는 자식이 그려진 영역만 터치를 감지한다. `HitTestBehavior.opaque`를 주면 빈 공간도 터치 영역에 포함된다.

## dispose 잊으면 메모리 누수

`initState`에서 만든 리소스는 `dispose`에서 반드시 정리해야 한다. 안 하면 메모리 누수가 생긴다.

```dart
@override
void initState() {
  super.initState();
  _scrollController = ScrollController();
  _textController = TextEditingController();
  _tabController = TabController(length: 3, vsync: this);
}

@override
void dispose() {
  _scrollController.dispose();
  _textController.dispose();
  _tabController.dispose();
  super.dispose();
}
```

정리 대상: `TextEditingController`, `ScrollController`, `AnimationController`, `TabController`, `Timer`, `StreamSubscription`. 규칙은 간단하다 - **내가 만든 건 내가 정리한다.**

SwiftUI에서는 `@StateObject`가 알아서 해주는 부분인데, Flutter는 수동이다. 번거롭지만 정확히 뭘 정리하는지 보이니까 디버깅할 때는 오히려 낫다.

## 정리

| 주제 | SwiftUI | Flutter |
|------|---------|---------|
| 멀티탭 | TabView + NavigationStack | IndexedStack + GlobalKey\<NavigatorState\> |
| 뒤로가기 제어 | 기본 제공 | PopScope (canPop + onPopInvokedWithResult) |
| 접히는 헤더 | ScrollView + .toolbar | CustomScrollView + SliverAppBar |
| 리스트 in 스크롤 | List + ScrollView | SliverList or shrinkWrap |
| 상태관리 | @State, @Published | setState, GetX (.obs + Obx) |
| 의존성 주입 | @EnvironmentObject | Get.put() / Get.find() |
| 다크모드 | @Environment(\.colorScheme) | InheritedWidget + context extension |
| 생명주기 정리 | @StateObject 자동 | dispose에서 수동 정리 |
| 애니메이션 | .transition + .animation | flutter_animate or AnimationController |

전체적으로 SwiftUI 대비 보일러플레이트가 확실히 많다. 특히 네비게이션 직접 관리하는 부분이랑 dispose 수동 정리가 번거로운데, 그만큼 코드에 마법이 없어서 흐름 파악은 더 쉬운 것 같다. GetX를 쓰면 `StatelessWidget`만으로도 거의 다 되니까 익숙해지면 개발 속도가 꽤 빨라진다.

다음 Ch03에서는 Dart 문법을 좀 더 깊게 파볼 예정이다.
