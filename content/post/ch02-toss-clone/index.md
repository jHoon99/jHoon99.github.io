---
title: "Ch02. 토스 앱 클론 - 실전 UI와 상태관리"
slug: "ch02-toss-clone"
date: 2026-04-26
draft: false
tags: ["Flutter", "GetX", "BottomNavigationBar", "CustomScrollView", "Theme", "Animation", "Toss Clone"]
categories: ["Flutter"]
description: "토스 앱 클론코딩으로 멀티탭 네비게이션, GetX 상태관리, 다크모드 테마 시스템, 검색 자동완성, 애니메이션까지 실전 Flutter UI를 구현한 기록"
---

Ch01에서 기본기를 다졌으니 Ch02에서는 토스 앱을 클론하면서 실전 UI를 만들어봤다. 탭 네비게이션, 상태관리, 테마 시스템 같은 실제 앱에 필요한 것들을 다룬다.

## 프로젝트 구조

```
lib/
├── main.dart                    // 앱 진입점
├── app.dart                     // 루트 위젯, 테마 설정
├── screen/
│   ├── main/
│   │   ├── s_main.dart          // 메인 화면 (BottomNav + IndexedStack)
│   │   ├── w_menu_drawer.dart   // 사이드 메뉴 (테마 전환)
│   │   └── tab/
│   │       ├── home/            // 홈 탭
│   │       ├── stock/           // 주식 탭 + 검색 + 설정
│   │       ├── benefit/         // 혜택 탭
│   │       ├── tosspay/         // 토스페이 탭
│   │       └── all/             // 전체 탭
│   └── notification/            // 알림 화면
└── common/
    ├── widget/                  // 공용 위젯 (Arrow, RoundedContainer 등)
    ├── theme/                   // 다크/라이트 테마 시스템
    └── dart/extension/          // BuildContext, 숫자 포맷 등 extension
```

SwiftUI에서는 파일 구조 자유도가 높은데, Flutter도 마찬가지다. 이 프로젝트에서는 `s_`(Screen), `f_`(Fragment), `w_`(Widget), `vo_`(Value Object) 접두사로 역할을 구분하는 컨벤션을 썼다. 실무에서는 feature 폴더 단위로 `screens/`, `widgets/` 하위에 넣는 게 더 일반적이다.

## 멀티탭 네비게이션

토스처럼 하단 5개 탭을 만들면서 각 탭이 독립적인 네비게이션 스택을 가져야 했다. SwiftUI의 `TabView` 안에 각각 `NavigationStack`을 넣는 것과 같은 구조다.

```dart
class MainScreenState extends State<MainScreen> {
  TabItem _currentTab = TabItem.home;
  final List<GlobalKey<NavigatorState>> navigatorKeys = [];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(
        index: _currentIndex,
        children: tabs.mapIndexed((tab, index) => TabNavigator(
          navigatorKey: navigatorKeys[index],
          tabItem: tab,
        )).toList(),
      ),
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _currentIndex,
        onTap: _handleOnTapNavigationBarItem,
        items: navigationBarItems(context),
        type: BottomNavigationBarType.fixed,
      ),
    );
  }
}
```

여기서 `IndexedStack`과 `GlobalKey<NavigatorState>`가 포인트다. `IndexedStack`은 모든 탭을 메모리에 유지하면서 현재 탭만 보여주고, 각 탭에 독립 `NavigatorState`를 줘서 탭 내 push/pop이 다른 탭에 영향을 안 준다.

뒤로가기 처리도 직접 해야 한다:

```dart
void _handleBackPressed(bool didPop) {
  if (!didPop) {
    // 1. 현재 탭에서 pop 가능하면 pop
    if (_currentTabNavigationKey.currentState?.canPop() == true) {
      Nav.pop(_currentTabNavigationKey.currentContext!);
      return;
    }
    // 2. 홈 탭이 아니면 홈으로 이동
    if (_currentTab != TabItem.home) {
      _changeTab(tabs.indexOf(TabItem.home));
    }
  }
}
```

SwiftUI에서는 `TabView` + `NavigationStack`이 알아서 해주는 걸, Flutter에서는 직접 관리해야 하는 부분이 많다.

## Scaffold - 화면의 뼈대

Flutter에서 새 화면(push 대상)을 만들 때는 거의 무조건 `Scaffold`를 쓴다. AppBar, body, BottomNavigationBar, Drawer, FloatingActionButton 등을 자동으로 배치해주고, 키보드가 올라올 때 화면 리사이즈도 해준다.

```dart
// Screen(push 대상) → Scaffold 필요
class SettingScreen extends StatefulWidget { ... }
// build에서
return Scaffold(
  appBar: AppBar(title: Text('설정')),
  body: ListView(...),
);

// Fragment(탭 안의 조각) → 상위에 이미 Scaffold가 있으므로 불필요
class StockFragment extends StatefulWidget { ... }
// build에서
return CustomScrollView(...);  // Scaffold 없이 바로
```

SwiftUI에서는 `NavigationView`가 비슷한 역할을 하지만 키보드 avoidance는 시스템이 자동으로 해준다. Flutter는 `Scaffold`가 그 역할을 담당한다.

## 홈 화면 - RefreshIndicator와 애니메이션

토스 홈 화면은 당겨서 새로고침, 은행 계좌 목록, 커스텀 앱바로 구성했다.

```dart
class HomeFragment extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        RefreshIndicator(
          onRefresh: () async {
            await sleepAsync(500.ms);
          },
          child: SingleChildScrollView(
            child: Column(
              children: [
                BigButton('토스뱅크', onTap: () => ...),
                RoundedContainer(
                  child: Column(
                    children: [
                      const Text('자산'),
                      ...bankAccounts.map(
                        (e) => BankAccountWidget(account: e),
                      ),
                    ],
                  ),
                ),
              ],
            ).animate().slide(duration: 1000.ms).fadeIn(),
          ),
        ),
        const TossAppBar(),
      ],
    );
  }
}
```

`flutter_animate` 패키지로 화면 진입 시 슬라이드+페이드 애니메이션을 한 줄로 추가했다. SwiftUI의 `.transition(.slide)` 같은 느낌인데, 체이닝이 가능해서 `.animate().slide().fadeIn().scale()` 이런 식으로 조합할 수 있다.

## 주식 탭 - CustomScrollView와 SliverAppBar

주식 탭은 `CustomScrollView` + `Sliver` 조합으로 만들었다. 스크롤에 따라 AppBar가 접히는 효과를 구현할 때 필요하다.

```dart
CustomScrollView(
  slivers: [
    SliverAppBar(
      actions: [
        ImageButton(onTap: () {}, imagePath: '$basePath/icon/stock_search.png'),
        ImageButton(onTap: () {}, imagePath: '$basePath/icon/stock_calendar.png'),
        ImageButton(onTap: () {}, imagePath: '$basePath/icon/stock_settings.png'),
      ],
    ),
    SliverToBoxAdapter(
      child: Column(
        children: [
          title,
          tabbar,
          currentIndex == 0
            ? const MyStockFragment()
            : const TodayDiscoveryFragment(),
        ],
      ),
    ),
  ],
)
```

탭 전환은 `TabBar` + `TabController`를 쓰되, `TabBarView` 대신 `currentIndex`로 직접 전환했다. `TabBarView`는 내부적으로 `PageView`를 쓰는데, `CustomScrollView` 안에서 높이 문제가 생기기 때문이다.

## GetX 상태관리 - 검색 자동완성

검색 기능에서 GetX를 써봤다. GetX는 상태관리 + 의존성 주입 + 라우팅을 한 패키지로 제공한다.

```dart
// Controller - 상태와 비즈니스 로직
class SearchStockData extends GetxController {
  List<SimpleStock> stocks = [];
  RxList<String> searchHistoryList = <String>[].obs;   // 반응형 리스트
  RxList<SimpleStock> autoCompleteList = <SimpleStock>[].obs;

  @override
  void onInit() {
    super.onInit();
    loadLocalStockJson();
  }

  void search(String keyword) {
    if (keyword.isEmpty) {
      autoCompleteList.clear();
      return;
    }
    autoCompleteList.value = stocks
        .where((e) => e.stockName.contains(keyword))
        .toList();
  }
}
```

`.obs`를 붙이면 반응형이 된다. 값이 바뀌면 `Obx`로 감싼 UI가 자동으로 리빌드된다.

```dart
// 화면 - 등록과 사용
class _SearchFragmentState extends State<SearchFragment> {
  @override
  void initState() {
    Get.put(SearchStockData());       // 전역 컨테이너에 등록
    super.initState();
  }

  @override
  void dispose() {
    Get.delete<SearchStockData>();     // 화면 나갈 때 정리
    super.dispose();
  }
}

// 자식 위젯 - Get.find로 접근
class SearchAutoCompleteList extends StatelessWidget {
  final searchData = Get.find<SearchStockData>();  // 어디서든 꺼냄

  @override
  Widget build(BuildContext context) {
    return Obx(() => ListView.builder(             // 자동 리빌드
      itemCount: searchData.autoCompleteList.length,
      itemBuilder: (context, index) {
        final stock = searchData.autoCompleteList[index];
        return Text(stock.stockName);
      },
    ));
  }
}
```

SwiftUI 비교로 정리하면:

| GetX | SwiftUI | 역할 |
|------|---------|------|
| `GetxController` | `ObservableObject` | 상태 담는 클래스 |
| `.obs` / `RxList` | `@Published` | 변경 감지 |
| `Obx(() =>)` | 자동 | UI 리빌드 |
| `Get.put()` | `.environmentObject()` | 등록 |
| `Get.find()` | `@EnvironmentObject` | 접근 |

차이점은 `@EnvironmentObject`는 위젯 트리 안에서만 접근 가능한 반면, `Get.find()`는 한번 `put`하면 어디서든 전역 접근 가능하다는 것이다. 편하지만 남용하면 상태 추적이 어려워진다. 요즘은 Riverpod이 더 많이 쓰인다.

## 알림 화면 - SliverList와 timeago

```dart
class NotificationScreen extends StatefulWidget { ... }

// build
return Scaffold(
  body: CustomScrollView(
    slivers: [
      const SliverAppBar(title: Text('알림')),
      SliverList(
        delegate: SliverChildBuilderDelegate(
          (context, index) => NotificationItemWidget(
            notification: notificationDummies[index],
          ),
          childCount: notificationDummies.length,
        ),
      ),
    ],
  ),
);
```

시간 표시에는 `timeago` 패키지를 사용했다. `DateTime`을 "27분 전", "1시간 전" 같은 상대 시간으로 바꿔준다.

```dart
Text(timeago.format(notification.time, locale: 'ko'))
// → "27분 전", "1시간 전"
```

## 설정 화면 - Switch와 setState

설정 화면은 간단하게 `Switch`와 `setState`로 구현했다.

```dart
class _SettingScreenState extends State<SettingScreen> {
  bool isPushOn = false;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('설정')),
      body: ListView(
        children: [
          SwitchMenu(
            '푸시 설정',
            isPushOn,
            onTap: (isOn) => setState(() => isPushOn = isOn),
          ),
        ],
      ),
    );
  }
}
```

`SwitchMenu`는 `void Function(bool)` 콜백을 받아서 상위에 값 변경을 알려주는 구조다. SwiftUI의 `@Binding`과 비슷한 역할인데, Flutter는 콜백으로 직접 전달한다.

## 다크모드 - 테마 시스템

테마를 `InheritedWidget` 기반으로 만들어서 앱 전체에서 `context.appColors`로 접근할 수 있게 했다.

```dart
// 테마 정의
enum CustomTheme {
  dark(DarkAppColors(), DarkAppShadows()),
  light(LightAppColors(), LightAppShadows());

  const CustomTheme(this.appColors, this.appShadows);
  final AbstractThemeColors appColors;
  final AbsThemeShadows appShadows;
}

// extension으로 축약
extension ContextExtension on BuildContext {
  AbstractThemeColors get appColors => CustomThemeHolder.of(this).appColors;
}

// 사용
Container(color: context.appColors.seedColor)
Icon(Icons.home, color: context.appColors.iconButton)
```

`Theme.of(context).colorScheme.primary` 이렇게 길게 안 쳐도 `context.appColors.primary`로 바로 접근 가능하다. 실무에서는 `flex_color_scheme` 패키지로 테마를 한방에 설정하는 것도 좋은 방법이다.

## StatefulWidget vs StatelessWidget

Flutter에서는 상태를 어디에 두느냐에 따라 위젯 타입이 나뉜다.

```dart
// StatefulWidget - 외부 입력값은 껍데기에, 상태는 State에
class MyWidget extends StatefulWidget {
  final String title;     // 외부에서 받는 값 → 여기
  const MyWidget({required this.title});
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  int count = 0;          // 내부 상태 → 여기

  @override
  void initState() { ... }   // 초기화 (viewDidLoad)
  @override
  void dispose() { ... }     // 정리 (deinit)
  @override
  Widget build(BuildContext context) {
    return Text('${widget.title}: $count');  // widget.으로 껍데기 접근
  }
}
```

SwiftUI에서는 `let title`과 `@State var count`를 한 곳에 선언하는데, Flutter는 클래스를 두 개로 나눠서 구분한다. GetX 같은 상태관리를 쓰면 상태를 Controller에 넣고 `StatelessWidget`만으로도 대부분 처리 가능하다.

### 생명주기와 dispose

`initState`에서 만든 것은 `dispose`에서 정리해야 한다. 안 하면 메모리 누수가 생긴다.

```dart
@override
void initState() {
  super.initState();
  controller = TextEditingController();
}

@override
void dispose() {
  controller.dispose();    // 반드시 정리
  super.dispose();
}
```

정리가 필요한 주요 대상: `TextEditingController`, `AnimationController`, `ScrollController`, `Timer`, `StreamSubscription`. **직접 만들었으면 dispose에서 정리한다.** 이것만 기억하면 된다.

## 커스텀 위젯 - 재사용 컴포넌트

자주 쓰는 UI 요소를 위젯으로 분리했다.

```dart
// 방향 화살표
class Arrow extends StatelessWidget {
  final AxisDirection direction;
  const Arrow({this.direction = AxisDirection.right});

  @override
  Widget build(BuildContext context) {
    return Icon(Icons.arrow_forward_ios, size: 16);  // 방향에 따라 회전
  }
}

// 둥근 컨테이너
class RoundedContainer extends StatelessWidget {
  final Widget child;
  final double radius;
  final EdgeInsets padding;

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: padding,
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(radius),
      ),
      child: child,
    );
  }
}
```

이런 작은 컴포넌트들을 `common/widget/`에 모아두면 어디서든 가져다 쓸 수 있다. SwiftUI에서 커스텀 View를 분리하는 것과 같다.

## 레이아웃 자주 만나는 에러들

Flutter 하면서 자주 만난 레이아웃 에러들을 정리해둔다.

**1. Unbounded height/width**
```dart
// ❌ Row 안에 TextField → 너비를 모름
Row(children: [TextField()])

// ✅ Expanded로 감싸기
Row(children: [Expanded(child: TextField())])
```

**2. 스크롤 위젯 안에 스크롤 위젯**
```dart
// ❌ ListView 안에 ListView → 높이 충돌
ListView(children: [ListView.builder(...)])

// ✅ shrinkWrap + NeverScrollableScrollPhysics
ListView(children: [
  ListView.builder(
    shrinkWrap: true,
    physics: NeverScrollableScrollPhysics(),
  ),
])
```

**3. GestureDetector 빈 영역 터치 안 됨**
```dart
// ❌ Spacer 영역 터치 안 먹힘
GestureDetector(
  onTap: () {},
  child: Row(children: [Text('...'), Spacer()]),
)

// ✅ behavior 추가
GestureDetector(
  behavior: HitTestBehavior.opaque,
  onTap: () {},
  child: Row(children: [Text('...'), Spacer()]),
)
```

## 정리

| 주제 | SwiftUI | Flutter |
|------|---------|---------|
| 멀티탭 | TabView + NavigationStack | IndexedStack + GlobalKey<NavigatorState> |
| 화면 뼈대 | NavigationView | Scaffold |
| 당겨서 새로고침 | .refreshable | RefreshIndicator |
| 스크롤 + 접히는 헤더 | ScrollView + .toolbar | CustomScrollView + SliverAppBar |
| 상태관리 | @State, @Published | setState, GetX(.obs + Obx) |
| 의존성 주입 | @EnvironmentObject | Get.put() / Get.find() |
| 다크모드 | @Environment(\.colorScheme) | InheritedWidget + context.appColors |
| 생명주기 | onAppear / onDisappear | initState / dispose |
| 커스텀 컴포넌트 | struct SomeView: View | class SomeWidget extends StatelessWidget |
| 입력 위젯 바인딩 | $value (Binding) | onChanged 콜백 |

SwiftUI 대비 보일러플레이트가 확실히 많다. 특히 StatefulWidget의 클래스 2개 구조나, 네비게이션 직접 관리하는 부분이 번거롭다. 하지만 그만큼 명시적이라 코드를 읽을 때 흐름 파악은 더 쉬운 것 같다. GetX 같은 상태관리를 쓰면 StatelessWidget만으로 거의 다 해결되니까 익숙해지면 오히려 편한 부분도 있다.
