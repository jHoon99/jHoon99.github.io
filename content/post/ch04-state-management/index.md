---
title: "Ch04. 상태관리를 왜 해야 하는가 - MVC에서 선언형 UI까지"
slug: "ch04-state-management"
date: 2026-04-28
weight: 1
draft: false
tags: ["Flutter", "Dart", "State Management", "MVVM", "MVC", "GetX", "Provider", "BLoC", "Riverpod", "Declarative UI"]
categories: ["Flutter"]
description: "MVC → MVP → MVVM → 선언형 UI 흐름과 Scoped Model vs Static Model 비교를 정리한 기록"
---

Ch04부터는 상태관리다. 왜 상태관리를 해야 하는지, 어떤 방식들이 있는지 정리한다. 개인적으로 이 부분이 꽤 재밌었다.

## 상태관리를 왜 해야 하나

결론부터 말하면 **앱을 더 쉽게 개발하기 위해서**다.

1. 처음 개발할 때 빠르게 만들기 위해
2. 스펙이 바뀌었을 때 수정할 곳을 빠르게 찾기 위해

작은 앱에서는 `setState`만으로도 충분한데, 화면이 10개 넘어가고 여러 화면에서 같은 데이터를 공유해야 하면 어디서 상태를 관리하고 어떻게 전달할지가 문제가 된다. 이걸 체계적으로 정리한 게 상태관리 패턴이다.

## MVC에서 선언형 UI까지 — 왜 이렇게 바뀌었나

### MVC (Model-View-Controller)

1970년대에 나온 패턴이다. 근데 많이들 오해하는 게 있다:

- 원래 **작은 컴포넌트 단위**의 설계였다. 앱 전체 아키텍처용이 아니었다.
- MVVM에서 말하는 데이터 Observing이 **이미 포함**돼 있던 개념이다.
- Controller는 원래 키보드/마우스 입력을 처리하는 역할이었다.

문제는 모바일에서 Controller의 의미가 변질됐다는 거다. Android의 Activity, iOS의 UIViewController가 이미 Controller이면서 동시에 View였다. 화면 자체가 Controller 역할을 하는 짬뽕이 된 거다. 결과적으로 Controller가 뚱뚱해지면서(Massive View Controller라고 부른다) 유지보수가 힘들어졌다.

Swift에서 UIKit 개발해본 사람이면 ViewController에 네트워크 호출, 테이블뷰 delegate, 데이터 가공 로직까지 다 때려넣어본 경험이 있을 거다. 그게 바로 MVC의 한계다.

### MVP (Model-View-Presenter)

MVC의 문제를 해결하려고 나온 게 MVP다.

- Android는 View가 XML로 분리돼 있었고
- iOS는 View가 ViewController + Storyboard로 분리돼 있었다

이렇게 **코드와 분리된 View를 제어할 Presenter**가 필요했다. Presenter가 로직을 담당하고, View는 그리기만 한다.

근데 문제가 있었다. Presenter가 View에게 **일일이 명령을 내려야** 화면이 갱신됐다. "이 라벨 텍스트 바꿔", "이 버튼 숨겨", "이 리스트 리로드해"... 하나하나 다 지시해야 했다. 코드가 장황해지고 빠뜨리면 UI 버그가 났다.

### MVVM (Model-View-ViewModel)

MVP의 "일일이 명령" 문제를 해결한 게 MVVM이다.

```
ViewModel의 상태를 바꾸면 → View가 알아서 갱신된다
```

핵심은 **데이터 바인딩**이다:

- Android에서는 DataBinding 라이브러리가 나왔고
- iOS에서는 RxSwift 같은 Rx 라이브러리로 Observable 패턴을 구현했다

값만 세팅하면 알아서 뷰가 갱신되니까 UI 버그가 확 줄었다. 근데 완벽하진 않았다. 뷰가 내부적으로 코드와 분리돼 있었기 때문에(XML, Storyboard) **ViewModel과 View를 연결하는 바인딩 코드**가 노가다였다.

iOS에서 RxSwift 쓸 때를 떠올리면:

```swift
// iOS + RxSwift: 바인딩 노가다
viewModel.userName
    .bind(to: nameLabel.rx.text)
    .disposed(by: disposeBag)

viewModel.isLoading
    .bind(to: activityIndicator.rx.isAnimating)
    .disposed(by: disposeBag)

viewModel.items
    .bind(to: tableView.rx.items(cellIdentifier: "Cell")) { ... }
    .disposed(by: disposeBag)
```

프로퍼티 하나하나 다 바인딩해줘야 했다.

### 선언형 UI — 최종 보스

그리고 선언형 UI가 등장했다.

- Android → **Jetpack Compose**
- iOS → **SwiftUI**
- 크로스플랫폼 → **Flutter**, **React**

[Flutter 공식 문서](https://docs.flutter.dev/flutter-for/declarative)에서 설명하는 핵심 차이는 이거다:

```dart
// 명령형 (Imperative) — 어떻게 바꿀지 지시
button.setColor(red);
button.setText('완료');
container.removeChild(oldChild);
container.addChild(newChild);

// 선언형 (Declarative) — 어떤 상태인지 선언
return ElevatedButton(
  style: ButtonStyle(backgroundColor: MaterialStateProperty.all(Colors.red)),
  onPressed: onTap,
  child: Text('완료'),
);
```

명령형은 "빨간색으로 바꿔, 텍스트 바꿔, 자식 교체해" 하고 일일이 지시하는 거고, 선언형은 "이 상태일 때 UI는 이렇게 생겼다"고 선언만 하면 프레임워크가 알아서 그려준다.

SwiftUI에서 `@State`가 바뀌면 `body`가 다시 그려지는 것과 완전히 같은 원리다:

```swift
// SwiftUI
struct CounterView: View {
    @State var count = 0
    var body: some View {
        Button("\(count)") { count += 1 }
    }
}
```

```dart
// Flutter
class CounterWidget extends StatefulWidget {
  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () => setState(() => count++),
      child: Text('$count'),
    );
  }
}
```

선언형 UI가 나오면서 **View의 패턴 구조를 논의할 필요가 없어졌다.** MVC냐 MVP냐 MVVM이냐가 아니라, **상태와 데이터를 어떻게 관리할지**가 더 중요해진 거다. 그게 바로 State Management다.

### 흐름 정리

```
MVC: 원래 작은 컴포넌트용이었는데 모바일에서 Controller가 비대해짐
  ↓
MVP: View와 로직을 Presenter로 분리. 근데 일일이 명령해야 함
  ↓
MVVM: 데이터 바인딩으로 자동 갱신. 근데 바인딩 코드가 노가다
  ↓
선언형 UI: 상태만 바꾸면 끝. 이제 "상태를 어떻게 관리할까"가 핵심
```

## Scoped Model vs Static Model

Flutter 상태관리 구조는 크게 두 가지로 나뉜다.

### Scoped Model

상태의 **범위를 제한**하는 방식이다. 특정 화면이나 위젯 트리 안에서만 상태가 유효하다.

```dart
// Provider 예시: 이 위젯 하위에서만 CartModel에 접근 가능
ChangeNotifierProvider(
  create: (_) => CartModel(),
  child: ShoppingPage(),  // 이 안에서만 CartModel 사용 가능
)
```

특징:
- 해당 화면이 사라지면 **자동으로 메모리 해제**된다
- 자식 위젯에서 **id 없이도** 상위 데이터를 참조할 수 있다
- 내부적으로 Flutter의 [InheritedWidget](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)을 활용한다

대표적인 Scoped 방식: **Provider**, **BLoC**, **Riverpod**

### Static Model

상태가 **전역으로 떠 있는** 방식이다. 앱 어디서든 접근할 수 있다.

```dart
// GetX 예시: 어디서든 접근 가능
Get.put(UserController());

// A 화면에서
Get.find<UserController>().user.value;

// B 화면에서도
Get.find<UserController>().user.value;  // 같은 인스턴스
```

특징:
- **메모리 관리를 개발자가 직접** 해야 한다
- 자식에서 데이터 참조할 때 **id(tag)가 항상 필요**하다
- Scoped보다 **구현이 훨씬 쉽다**
- 어디서든 접근/수정 가능하니까 **상태가 꼬일 위험**이 있다

대표적인 Static 방식: **GetX**

### 뭐가 다른지 예시로 보면

유튜브 앱이라고 치자. 영상 재생 화면은 코드상으로는 같은 `VideoScreen`인데, 어떤 영상이냐에 따라 동영상 URL, 댓글, 좋아요 수가 전부 다르다.

**Scoped 방식**에서는:

```dart
// 각 영상 화면의 상위에 해당 영상의 상태를 넣어줌
ChangeNotifierProvider(
  create: (_) => VideoState(videoId: 'abc123'),
  child: VideoScreen(),  // 이 안에서는 abc123 영상의 상태만 보임
)
```

Riverpod에서는 `family`라는 기능으로 더 깔끔하게 처리한다:

```dart
// Riverpod: videoId별로 자동으로 다른 상태 인스턴스 생성
final videoProvider = FutureProvider.family<Video, String>((ref, videoId) {
  return fetchVideo(videoId);
});

// 사용: 같은 provider인데 id가 다르면 다른 상태
ref.watch(videoProvider('abc123'));  // 영상 1의 상태
ref.watch(videoProvider('xyz789'));  // 영상 2의 상태 (별도)
```

**Static 방식**(GetX)에서는:

```dart
// tag로 구분해서 따로 관리
Get.put(VideoController(), tag: 'abc123');
Get.put(VideoController(), tag: 'xyz789');

// 사용할 때 tag로 찾아와야 함
Get.find<VideoController>(tag: 'abc123');
```

둘 다 가능하다. 근데 Scoped는 화면 닫으면 알아서 정리되고, Static은 개발자가 직접 `Get.delete(tag: 'abc123')` 해줘야 한다.

### 또 다른 예시 — ID가 없는 경우

데스크탑 앱에서 "새 문서" 버튼을 3번 눌렀다고 치자. 각 문서는 아직 저장 전이라 고유 ID가 없다. 이때는 Scoped가 자연스럽다:

```dart
// Scoped: 각 문서 화면이 자기 스코프 안에 상태를 가짐
Navigator.push(context, MaterialPageRoute(
  builder: (_) => ChangeNotifierProvider(
    create: (_) => DocumentState(),  // 각각 독립된 상태
    child: DocumentScreen(),
  ),
));
```

Static으로도 가능은 하다 — 임시 ID를 생성해서 tag로 관리하면 된다. 근데 굳이 그럴 필요 없이 Scoped가 더 깔끔한 케이스다.

### 그래서 뭘 써야 하나

정답은 **스펙에 따라 다르다**. 하지만 일반적인 기준은 있다:

| | Scoped Model | Static Model |
|--|-------------|-------------|
| 메모리 관리 | 자동 해제 | 직접 관리 |
| 데이터 접근 | 위젯 트리 통해서 | 어디서든 직접 |
| 구현 난이도 | 상대적으로 복잡 | 쉬움 |
| 상태 안정성 | 범위가 제한돼서 안전 | 어디서든 수정 가능해서 꼬일 수 있음 |
| 테스트 | 스코프 단위로 격리 가능 | 전역 상태라 격리 어려움 |
| 대표 패키지 | Provider, BLoC, Riverpod | GetX |

Swift 개발자 입장에서 비유하면:
- **Scoped** = SwiftUI에서 `@StateObject`를 뷰 계층에 맞게 넣는 것
- **Static** = 싱글톤으로 전역 접근하는 것 (`AppState.shared`)

iOS 개발할 때도 싱글톤 남발하면 테스트 힘들고 상태 꼬이는 걸 경험해봤을 텐데, Flutter에서도 똑같다. GetX가 쉬운 건 맞지만 앱이 커지면 Scoped 방식이 관리하기 편하다.

## 상태관리 패키지 현황

[Flutter 공식 문서](https://docs.flutter.dev/data-and-backend/state-mgmt/options)에서도 여러 접근법을 소개하고 있다. 현재 주요 패키지들의 포지션을 정리하면:

| 패키지 | 방식 | 특징 |
|--------|------|------|
| **Provider** | Scoped | Flutter 팀 추천 입문용. InheritedWidget 래퍼 |
| **BLoC** | Scoped | Event → State 단방향 흐름. 엔터프라이즈 앱에 적합 |
| **Riverpod** | Scoped | Provider의 진화형. 컴파일 타임 안전성. family로 키 기반 관리 |
| **GetX** | Static | 전역 접근, 쉬운 구현. tag로 인스턴스 구분 |

Flutter 공식 입장은 "setState로 시작하고, 복잡해지면 패키지를 도입하라"다. 어떤 패키지가 절대적으로 좋다기보다는 앱 규모와 팀 상황에 맞는 걸 고르는 게 맞다.

## 정리

상태관리의 역사를 보면 결국 한 방향으로 흘러왔다:

```
"UI를 어떻게 그릴까" → "상태를 어떻게 관리할까"
```

MVC에서 선언형 UI까지 오는 동안 패턴이 계속 바뀐 이유는 **"상태가 바뀌면 UI가 알아서 반영되게"** 하고 싶었기 때문이다. 선언형 UI가 그걸 해결했고, 이제 남은 문제는 그 상태를 어떤 범위에서 어떻게 관리할지다.

Scoped냐 Static이냐는 결국 트레이드오프다. 쉬운 걸 원하면 Static(GetX), 안전한 걸 원하면 Scoped(Provider/BLoC/Riverpod). iOS 개발할 때 싱글톤 vs 의존성 주입 고민했던 것과 본질적으로 같은 문제다.
