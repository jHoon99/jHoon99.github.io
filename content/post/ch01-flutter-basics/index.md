---
title: "Ch01. Flutter 기초 - 위젯부터 인스타 클론까지"
slug: "ch01-flutter-basics"
date: 2026-04-13
draft: false
tags: ["Flutter", "Widget", "Layout", "Navigation", "setState", "GoRouter", "Instagram Clone"]
categories: ["Flutter"]
description: "Flutter의 기본 위젯, 레이아웃, 상태관리, 네비게이션, 테마 설정을 익히고, 마지막에 인스타그램 클론 UI를 만들어본 기록"
---

iOS 개발하다가 Flutter 시작한 지 얼마 안 됐는데, 처음부터 정리해보려고 한다. Ch01은 기본 위젯부터 시작해서 최종적으로 인스타그램 클론 UI까지 만드는 과정이다.

## Container - 가장 기본적인 박스

SwiftUI에서 `.background`, `.border`, `.shadow` 이런 modifier 조합으로 스타일링하는 것과 비슷하게, Flutter에서는 `Container` + `BoxDecoration`으로 처리한다.

```dart
Container(
  width: 300,
  height: 300,
  decoration: BoxDecoration(
    color: Colors.pink.shade50,
    border: Border.all(color: Colors.red, width: 5),
    borderRadius: BorderRadius.circular(10),
    boxShadow: [
      BoxShadow(color: Colors.black, offset: Offset(6, 6))
    ],
  ),
  child: Center(
    child: Container(
      color: Colors.yellow,
      padding: EdgeInsets.symmetric(horizontal: 20),
      margin: EdgeInsets.symmetric(horizontal: 10),
      child: Text('Hello Container'),
    ),
  ),
)
```

SwiftUI에서는 `padding`이 modifier 체이닝 순서에 따라 안/밖이 달라지는데, Flutter는 `padding`과 `margin`이 명확히 분리돼 있어서 오히려 직관적인 편이다.

## 레이아웃 - Column, Row, Flexible

SwiftUI의 `VStack`, `HStack`이 Flutter에서는 `Column`, `Row`다. 거의 1:1 대응이라 어색하지 않았다.

```dart
Column(
  mainAxisAlignment: MainAxisAlignment.center,
  children: [
    Row(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Container(width: 100, height: 80, color: Colors.red),
        Container(width: 100, height: 80, color: Colors.blue),
        Container(width: 100, height: 80, color: Colors.green),
      ],
    ),
    Container(width: 300, height: 90, color: Colors.grey),
  ],
)
```

비율 배치는 `Flexible`의 `flex` 속성으로 한다. SwiftUI의 `.frame(maxHeight: .infinity)` 같은 것보다 깔끔함.

```dart
Column(
  children: [
    Flexible(flex: 1, child: Container(color: Colors.red)),
    Flexible(flex: 2, child: Container(color: Colors.blue)),
    Flexible(flex: 3, child: Container(color: Colors.green)),
    Flexible(flex: 4, child: Container(color: Colors.yellow)),
  ],
)
```

1:2:3:4 비율로 화면을 나눠줌. `Expanded`는 `Flexible`의 `fit: FlexFit.tight` 버전이라 남은 공간을 무조건 채운다.

## Stack - 위젯 겹치기

SwiftUI의 `ZStack`이 Flutter에서는 `Stack`이다. 자식 위젯의 위치는 `Align`이나 `Positioned`로 잡는다.

```dart
Stack(
  children: [
    Container(width: 500, height: 500, color: Colors.black),
    Container(width: 400, height: 400, color: Colors.red),
    Container(width: 300, height: 300, color: Colors.blue),
    Align(
      alignment: Alignment.topRight,
      child: Container(width: 200, height: 200, color: Colors.green),
    ),
  ],
)
```

`Positioned`는 `top`, `left`, `right`, `bottom` 값으로 절대 위치를 지정하고, `Align`은 상대 위치를 지정하는 차이가 있다.

## StatelessWidget vs StatefulWidget

SwiftUI에서는 `@State`로 상태를 선언하면 끝인데, Flutter는 `StatelessWidget`과 `StatefulWidget`이 명확히 나뉜다.

```dart
// 상태 없는 위젯 - 한 번 그리면 끝
class ExampleStateless extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(color: Colors.red);
  }
}

// 상태 있는 위젯 - setState로 업데이트
class ExampleStateful extends StatefulWidget {
  final int index;
  const ExampleStateful({required this.index});

  @override
  State<ExampleStateful> createState() => _ExampleStatefulState();
}

class _ExampleStatefulState extends State<ExampleStateful> {
  late int _index;

  @override
  void initState() {
    super.initState();
    _index = widget.index;
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () {
        setState(() {
          _index = _index == 5 ? 0 : _index + 1;
        });
      },
      child: Container(
        color: Colors.blue,
        child: Center(child: Text('index: $_index')),
      ),
    );
  }
}
```

처음에 `setState` 쓰는 게 좀 번거로웠는데, SwiftUI의 `@State`가 내부적으로 해주는 걸 Flutter는 직접 명시하는 거라고 생각하니까 이해됐다. `initState`는 SwiftUI의 `.onAppear`, `dispose`는 `.onDisappear`랑 비슷한 라이프사이클이다.

## 입력 위젯들

Checkbox, Radio, Slider, Switch 같은 기본 입력 위젯들도 전부 `StatefulWidget` + `setState` 패턴이다.

```dart
// Slider 예제
class TestSlider extends StatefulWidget {
  @override
  State<TestSlider> createState() => _TestSliderState();
}

class _TestSliderState extends State<TestSlider> {
  double value = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('$value'),
        Slider(
          value: value,
          onChanged: (newValue) => setState(() => value = newValue),
          divisions: 100,
          activeColor: Colors.red,
        ),
      ],
    );
  }
}
```

iOS 스타일을 쓰고 싶으면 `CupertinoSwitch`, `CupertinoContextMenu` 같은 Cupertino 위젯도 있다. Material이랑 Cupertino를 섞어서 쓸 수 있는 게 Flutter의 장점인듯.

## 네비게이션 - GoRouter

Flutter의 기본 Navigator도 있지만, `go_router` 패키지가 훨씬 편하다. SwiftUI의 `NavigationStack` + `.navigationDestination`과 비슷한 느낌.

```dart
MaterialApp.router(
  routerConfig: GoRouter(
    initialLocation: '/',
    routes: [
      GoRoute(
        path: '/',
        name: 'home',
        builder: (context, _) => const HomeWidget(),
      ),
      GoRoute(
        path: '/new',
        name: 'new',
        builder: (context, _) => const NewPage(),
      ),
    ],
  ),
)
```

화면 이동은 `context.pushNamed('new')`, 뒤로가기는 `context.pop()`. 직관적이다.

`BottomNavigationBar`는 SwiftUI의 `TabView`와 같은 역할인데, `currentIndex`와 `onTap`으로 탭 전환을 직접 관리한다.

```dart
BottomNavigationBar(
  currentIndex: index,
  onTap: (newIndex) => setState(() => index = newIndex),
  items: [
    BottomNavigationBarItem(icon: Icon(Icons.home_filled), label: '홈'),
    BottomNavigationBarItem(icon: Icon(Icons.search), label: '검색'),
  ],
)
```

## ThemeData

앱 전체 스타일을 한 곳에서 관리하는 건 SwiftUI랑 비슷하다. `ColorScheme`이랑 `TextTheme`을 설정해두면 `Theme.of(context)`로 어디서든 가져다 쓸 수 있다.

```dart
MaterialApp(
  theme: ThemeData(
    colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
    textTheme: TextTheme(
      bodyMedium: TextStyle(fontWeight: FontWeight.normal, fontSize: 28),
    ),
  ),
)

// 사용할 때
final textTheme = Theme.of(context).textTheme;
Text('Press Count', style: textTheme.bodyMedium);
```

## 인스타그램 클론 - 배운 거 총동원

Ch01 마무리로 인스타그램 클론 UI를 만들었다. 위에서 배운 거 거의 다 써먹은 프로젝트다.

### 구조

```
lib/
├── main.dart          // 앱 진입점, 테마, 하단 탭
├── body.dart          // 탭별 화면 라우팅
└── screen/
    ├── home_screen.dart   // 피드, 스토리
    └── search_screen.dart // 검색, 그리드
```

### 메인 - 테마와 탭 바

```dart
MaterialApp(
  home: const InstaCloneHome(),
  theme: ThemeData(
    colorScheme: const ColorScheme.light(
      primary: Colors.white,
      secondary: Colors.black,
    ),
    bottomNavigationBarTheme: const BottomNavigationBarThemeData(
      showSelectedLabels: false,
      showUnselectedLabels: false,
      selectedItemColor: Colors.black,
    ),
  ),
)
```

인스타그램 특유의 흰 배경 + 검정 아이콘 스타일을 `ColorScheme`으로 잡았다. AppBar에는 `google_fonts` 패키지로 Lobster Two 폰트를 적용했다.

### 스토리 영역 - 가로 스크롤

```dart
class StoryArea extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return SingleChildScrollView(
      scrollDirection: Axis.horizontal,
      child: Row(
        children: List.generate(
          10,
          (index) => UserStory(userName: 'User $index'),
        ),
      ),
    );
  }
}
```

SwiftUI에서 `ScrollView(.horizontal)` 안에 `HStack` 넣는 것과 같은 패턴. `List.generate`로 더미 데이터를 만들어서 넣었다.

### 피드 리스트

```dart
class FeedData {
  final String userName;
  final int likeCount;
  final String content;
  const FeedData({required this.userName, required this.likeCount, required this.content});
}

class FeedList extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      shrinkWrap: true,
      physics: NeverScrollableScrollPhysics(),
      itemBuilder: (context, index) => FeedItem(feedDataList[index]),
      itemCount: feedDataList.length,
    );
  }
}
```

여기서 `shrinkWrap: true`랑 `NeverScrollableScrollPhysics()`가 핵심이다. 부모가 `SingleChildScrollView`니까 `ListView`가 자체 스크롤하면 충돌한다. `shrinkWrap`으로 콘텐츠 크기만큼만 차지하게 하고, 스크롤은 부모한테 위임하는 거다.

### 검색 화면 - GridView

```dart
GridView.count(
  crossAxisCount: 3,
  mainAxisSpacing: 4,
  crossAxisSpacing: 4,
  shrinkWrap: true,
  physics: const NeverScrollableScrollPhysics(),
  children: gridItem.map((color) => Container(color: color)).toList(),
)
```

3열 그리드. SwiftUI의 `LazyVGrid`와 비슷한데, `GridView.count`가 더 간결하다.

## 정리

| 주제 | SwiftUI | Flutter |
|------|---------|---------|
| 박스 스타일링 | modifier 체이닝 | Container + BoxDecoration |
| 수직/수평 배치 | VStack / HStack | Column / Row |
| 비율 배치 | .frame + GeometryReader | Flexible / Expanded |
| 겹치기 | ZStack | Stack + Align/Positioned |
| 상태관리 | @State | StatefulWidget + setState |
| 화면전환 | NavigationStack | GoRouter |
| 전역 테마 | .environment | ThemeData + Theme.of(context) |

전반적으로 SwiftUI보다 보일러플레이트가 좀 더 많긴 한데, 그만큼 명시적이라서 코드를 읽을 때 뭘 하는지 파악이 더 쉬운 것 같다. 특히 레이아웃 시스템은 SwiftUI보다 예측 가능해서 좋았음. `GeometryReader` 같은 트릭 안 써도 `Flexible`로 비율 잡으면 깔끔하게 떨어진다.

다음 Ch02에서는 토스 앱 클론 만들면서 좀 더 실전적인 UI를 다룰 예정이다.
