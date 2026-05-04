---
title: "Ch06. Flutter 네트워킹 - API부터 Riverpod 연동까지 한방 정리"
slug: "ch06-flutter-networking"
date: 2026-05-05
draft: false
tags: ["Flutter", "Dart", "REST API", "Dio", "Retrofit", "Freezed", "Riverpod", "JWT", "json_serializable"]
categories: ["Flutter"]
description: "REST API 기본 개념부터 JWT 인증, Dio, Retrofit, Freezed, Riverpod 연동까지 — Flutter에서 서버와 통신하는 전체 파이프라인을 한 포스트로 정리한 기록"
---

Ch05에서 상태관리를 끝냈다. ValueNotifier부터 Riverpod까지 같은 Todo 앱을 5번 갈아엎으면서 "뷰와 상태를 어떻게 분리하는가"에 집중했다면, Ch06은 **"그 상태를 서버에서 어떻게 가져오는가"**에 집중한다.

지금까지의 Todo 앱은 로컬 데이터였다. 하지만 실제 앱은 서버에서 데이터를 받아오고, 유저 인증도 하고, 에러도 처리해야 한다. 이번 챕터에서 다루는 전체 파이프라인을 먼저 보면:

```
뷰 → ref.watch → Provider → Retrofit → Dio → 인터셉터 → 서버
서버 → JSON → fromJson → Provider 상태 → .when() → 뷰
```

이걸 하나씩 쌓아 올린다.

## 1. API 기본 개념

### API란?

앱과 서버의 **대화 방법**이다. 식당으로 치면 메뉴판 — "이 URL로 이런 데이터를 이런 형식으로 보내면, 이런 응답을 줄게"라는 약속.

iOS에서 URLSession으로 직접 요청을 날려본 경험이 있는데, Flutter도 본질은 같다. URL + HTTP 메서드 + 바디 → 서버 → 응답.

### REST API

**RE**presentational **S**tate **T**ransfer. URL로 자원을 표현하고, HTTP 메서드로 행위를 구분하는 규칙이다.

```
GET    /todos       → 전체 조회 (Read)
GET    /todos/1     → 1번 조회 (Read)
POST   /todos       → 생성 (Create)
PUT    /todos/1     → 수정 (Update)
DELETE /todos/1     → 삭제 (Delete)
```

"RESTful하다"는 건 이 규칙을 잘 지킨다는 뜻이다. URL에 동사가 들어가거나(`/getTodos`), 전부 POST로 처리하면 RESTful하지 않다.

### HTTP 메서드 = CRUD

| HTTP 메서드 | CRUD | 설명 |
|------------|------|------|
| GET | Read | 데이터 조회 (바디 없음) |
| POST | Create | 데이터 생성 |
| PUT | Update | 전체 수정 |
| PATCH | Update | 부분 수정 |
| DELETE | Delete | 삭제 |

iOS의 `URLRequest.httpMethod`로 설정하던 것과 동일한 개념.

### HTTP 상태 코드

서버의 응답 결과를 세 자리 숫자로 알려준다.

- **2xx 성공** — `200 OK`, `201 Created`, `204 No Content`
- **4xx 내 잘못** — `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`
- **5xx 서버 잘못** — `500 Internal Server Error`, `503 Service Unavailable`

핵심은 **401과 403의 차이**. 401은 "너 누구야" (인증 실패), 403은 "너인 건 알겠는데 권한 없어" (인가 실패).

### JSON

서버와 주고받는 데이터 형식. XML도 있지만 요즘은 거의 JSON이다.

```json
{
  "id": 1,
  "title": "Flutter 공부",
  "completed": false
}
```

Dart에서는 `Map<String, dynamic>`으로 파싱된다. 이걸 직접 다루면 타입 안전성이 없으니, 뒤에서 Freezed로 모델 클래스를 자동 생성한다.

### HTTP vs HTTPS

HTTP에 **SSL/TLS 암호화**를 얹은 게 HTTPS. 요즘은 HTTPS가 기본이고, iOS의 ATS(App Transport Security)처럼 Flutter도 실무에선 HTTPS만 쓴다.

## 2. 인증 — 토큰 기반 인증

### 토큰이란?

로그인 성공 시 서버가 발급하는 **증명서**. 이후 모든 API 요청에 이 토큰을 함께 보내서 "나 로그인한 사람이야"를 증명한다.

### JWT (JSON Web Token) 구조

```
xxxxx.yyyyy.zzzzz
헤더   .내용   .서명
```

- **헤더**: 알고리즘, 토큰 타입
- **내용 (Payload)**: 유저 ID, 만료 시간 등 (Base64 디코딩하면 읽을 수 있음 → 민감 정보 넣으면 안 됨)
- **서명**: 서버만 검증 가능한 서명 (위조 방지)

### Access Token vs Refresh Token

| | Access Token | Refresh Token |
|-|-------------|---------------|
| 수명 | 짧음 (15분~1시간) | 김 (2주~1개월) |
| 용도 | API 요청 시 첨부 | Access Token 재발급 |
| 탈취 시 | 피해 제한적 (곧 만료) | 위험 (새 Access 발급 가능) |
| 저장 | 메모리 or SharedPreferences | SharedPreferences (암호화) |

### 자동 갱신 흐름

```
1. API 요청 → 401 Unauthorized (Access Token 만료)
2. Refresh Token으로 /auth/refresh 요청
3. 새 Access Token 발급
4. 원래 요청 재시도
```

이 흐름을 매번 수동으로 하면 미친다. 뒤에서 Dio 인터셉터로 자동화한다.

### 토큰 저장

```dart
// 저장
final prefs = await SharedPreferences.getInstance();
await prefs.setString('accessToken', token);

// 읽기
final token = prefs.getString('accessToken');
```

> **SharedPreferences**는 iOS의 UserDefaults와 같다. 간단한 키-값 저장. 실무에선 `flutter_secure_storage`로 암호화 저장을 쓰기도 한다.

### Firebase Auth와의 차이

| | Firebase Auth | JWT 직접 구현 |
|-|--------------|--------------|
| 토큰 관리 | SDK가 자동 | 직접 관리 |
| 인터셉터 | 불필요 | 필요 |
| 유연성 | 제한적 | 자유로움 |
| 서버 | Firebase 종속 | 자체 서버 |

Firebase Auth는 토큰 갱신을 SDK가 알아서 해준다. 편하지만 서버 로직을 직접 통제할 수 없다. 자체 서버가 있으면 JWT 직접 관리가 일반적.

## 3. Dio — HTTP 클라이언트

### http vs Dio

Flutter에는 기본 `http` 패키지가 있다. 하지만 실무에선 거의 Dio를 쓴다.

| | http (기본) | Dio |
|-|-----------|-----|
| iOS 비유 | URLSession | Alamofire |
| 인터셉터 | 없음 | 있음 |
| 토큰 자동 첨부 | 직접 구현 | 인터셉터로 자동 |
| 에러 핸들링 | 기본 | 풍부 |
| 파일 업로드 | 번거로움 | FormData 지원 |

### BaseUrl 설정

```dart
final dio = Dio(
  BaseOptions(
    baseUrl: 'https://api.example.com',
    connectTimeout: Duration(seconds: 5),
    receiveTimeout: Duration(seconds: 3),
  ),
);
```

한 번 설정해두면 이후 요청에서 `dio.get('/todos')`처럼 경로만 쓰면 된다.

### 인터셉터 — 핵심 기능

인터셉터는 **모든 요청/응답을 가로채서 가공**하는 미들웨어다.

```dart
class AuthInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // 모든 요청에 토큰 자동 첨부
    final token = TokenStorage.accessToken;
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      // Access Token 만료 → Refresh Token으로 갱신
      final newToken = await refreshToken();
      if (newToken != null) {
        // 토큰 갱신 성공 → 원래 요청 재시도
        err.requestOptions.headers['Authorization'] = 'Bearer $newToken';
        final response = await dio.fetch(err.requestOptions);
        return handler.resolve(response);
      }
    }
    handler.next(err);
  }
}
```

이걸 Dio에 등록하면:

```dart
dio.interceptors.add(AuthInterceptor());
```

이제 **모든 API 요청에 토큰이 자동으로 붙고, 401이 오면 자동으로 갱신해서 재시도**한다. 인터셉터 없이 이걸 하려면 모든 API 호출마다 토큰 로직을 넣어야 한다.

### 앱 시작 시 1회 초기화

Dio 인스턴스는 앱 전체에서 하나만 만들어서 공유한다. 보통 DI(의존성 주입) 또는 Provider로 관리.

## 4. Retrofit — API 코드 자동 생성

### Retrofit이란?

Dio 위에 얹는 **코드 자동 생성 도구**. API 인터페이스만 정의하면 구현체를 `build_runner`가 만들어준다.

### 없을 때 vs 있을 때

**Dio만 쓸 때 (수동):**

```dart
Future<List<Todo>> getTodos() async {
  final response = await dio.get('/todos');
  return (response.data as List)
      .map((json) => Todo.fromJson(json))
      .toList();
}

Future<Todo> createTodo(Todo todo) async {
  final response = await dio.post('/todos', data: todo.toJson());
  return Todo.fromJson(response.data);
}
```

API가 20개면 이런 코드가 20개. 실수도 잦고 지루하다.

**Retrofit 쓸 때 (자동):**

```dart
@RestApi(baseUrl: 'https://api.example.com')
abstract class TodoApi {
  factory TodoApi(Dio dio) = _TodoApi;

  @GET('/todos')
  Future<List<Todo>> getTodos();

  @POST('/todos')
  Future<Todo> createTodo(@Body() Todo todo);

  @PUT('/todos/{id}')
  Future<Todo> updateTodo(@Path('id') int id, @Body() Todo todo);

  @DELETE('/todos/{id}')
  Future<void> deleteTodo(@Path('id') int id);
}
```

선언만 하면 `_TodoApi` 구현체가 자동 생성된다. HTTP 메서드와 어노테이션이 직관적이라 API 문서를 보면서 그대로 옮기면 된다.

### 어노테이션 정리

| 어노테이션 | 역할 | 예시 |
|----------|------|------|
| `@RestApi` | 기본 URL 설정 | `@RestApi(baseUrl: '...')` |
| `@GET` | GET 요청 | `@GET('/todos')` |
| `@POST` | POST 요청 | `@POST('/todos')` |
| `@PUT` | PUT 요청 | `@PUT('/todos/{id}')` |
| `@DELETE` | DELETE 요청 | `@DELETE('/todos/{id}')` |
| `@Path` | URL 경로 변수 | `@Path('id') int id` |
| `@Body` | 요청 바디 | `@Body() Todo todo` |
| `@Query` | 쿼리 파라미터 | `@Query('page') int page` |
| `@Header` | 헤더 | `@Header('Authorization') String token` |

### build_runner 실행

```bash
dart run build_runner build
```

`.g.dart` 파일이 생성된다. 코드가 바뀌면 다시 돌려야 하고, `watch` 모드로 자동 감지도 가능:

```bash
dart run build_runner watch
```

## 5. Freezed + json_serializable — 모델 클래스

### 왜 필요한가

서버에서 오는 JSON을 `Map<String, dynamic>`으로 쓰면:

```dart
final title = json['title'] as String; // 타입 캐스팅 필수
final id = json['id'] as int;          // 키 오타나면 런타임 에러
```

타입 안전성이 없고, 필드가 10개면 지옥이다. Freezed + json_serializable은 이 문제를 **코드 자동 생성**으로 해결한다.

### Freezed 모델 정의

```dart
@freezed
class Todo with _$Todo {
  const factory Todo({
    required int id,
    required String title,
    @Default(false) bool completed,
    DateTime? dueDate,
  }) = _Todo;

  factory Todo.fromJson(Map<String, dynamic> json) => _$TodoFromJson(json);
}
```

`build_runner`를 돌리면 자동으로 생성되는 것들:

- **`fromJson` / `toJson`** — JSON ↔ 객체 변환
- **`copyWith`** — 불변 객체의 일부 필드만 변경한 새 객체 생성
- **`==` / `hashCode`** — 값 비교 (동일 값이면 같은 객체로 판단)
- **`toString`** — 디버깅용 출력

### @freezed vs @unfreezed

| | @freezed | @unfreezed |
|-|---------|-----------|
| 불변성 | 불변 (immutable) | 가변 (mutable) |
| 수정 방법 | `copyWith` (새 객체) | 직접 필드 수정 |
| 용도 | 상태관리 모델, API 응답 | 폼 입력 등 자주 바뀌는 임시 데이터 |

**실무에선 거의 `@freezed`만 쓴다.** 상태관리에서 불변 객체가 변경 감지에 유리하기 때문.

### Swift Codable과 비교

| | Flutter (Freezed) | Swift (Codable) |
|-|-------------------|-----------------|
| 정의 | `@freezed class Todo` | `struct Todo: Codable` |
| JSON 변환 | `fromJson` / `toJson` | `JSONDecoder` / `JSONEncoder` |
| 불변성 | `const factory` | `let` 프로퍼티 |
| 코드 생성 | `build_runner` 필요 | 컴파일러가 자동 |
| 부분 수정 | `copyWith` | 직접 구현 필요 |

Swift의 Codable이 컴파일러 레벨에서 해주는 걸, Flutter는 build_runner로 해결한다. 좀 번거롭지만 결과물은 비슷하고, copyWith까지 공짜로 주니 오히려 편한 면도 있다.

### copyWith 예시

```dart
final todo = Todo(id: 1, title: 'Flutter 공부', completed: false);

// 완료 상태만 변경한 새 객체
final done = todo.copyWith(completed: true);

// 원본은 변경되지 않음
print(todo.completed);  // false
print(done.completed);  // true
```

## 6. Riverpod 연동 — Provider로 서버 데이터 관리

Ch05에서 Riverpod의 기본을 다뤘으니, 여기서는 **서버 데이터와 어떻게 연동하는가**에 집중한다.

### 함수형 Provider = 읽기 (GET)

```dart
@riverpod
Future<List<Todo>> todoList(Ref ref) async {
  final api = ref.watch(todoApiProvider);
  return api.getTodos();
}
```

함수형으로 선언하면 **읽기 전용**. GET 요청으로 데이터를 가져오는 용도. 자동으로 `AsyncValue`가 되어 로딩/에러/성공 상태를 `.when()`으로 처리할 수 있다.

### 클래스형 Provider = CRUD

```dart
@riverpod
class TodoListNotifier extends _$TodoListNotifier {
  @override
  Future<List<Todo>> build() async {
    final api = ref.watch(todoApiProvider);
    return api.getTodos();
  }

  Future<void> addTodo(Todo todo) async {
    final api = ref.read(todoApiProvider);
    await api.createTodo(todo);
    ref.invalidateSelf(); // 목록 새로고침
  }

  Future<void> deleteTodo(int id) async {
    final api = ref.read(todoApiProvider);
    await api.deleteTodo(id);
    ref.invalidateSelf();
  }
}
```

클래스형으로 선언하면 **메서드를 통해 쓰기(C/U/D) 가능**. `ref.invalidateSelf()`를 호출하면 `build()`가 다시 실행되어 서버에서 최신 데이터를 가져온다.

### ref.invalidate — 데이터 새로고침

```dart
// Provider 외부에서 새로고침
ref.invalidate(todoListNotifierProvider);

// Provider 내부에서 자기 자신 새로고침
ref.invalidateSelf();
```

"캐시를 버리고 다시 가져와"라는 의미. Pull-to-refresh 같은 기능에 딱이다.

### Provider끼리 참조

```dart
@riverpod
TodoApi todoApi(Ref ref) {
  final dio = ref.watch(dioProvider);
  return TodoApi(dio);
}

@riverpod
Future<List<Todo>> todoList(Ref ref) async {
  final api = ref.watch(todoApiProvider); // 다른 Provider 참조
  return api.getTodos();
}
```

`ref.watch`로 다른 Provider를 참조하면, 의존하는 Provider가 바뀔 때 자동으로 다시 실행된다. Provider끼리의 의존성 그래프를 Riverpod이 알아서 관리해준다.

### 함수형 vs 클래스형 정리

| | 함수형 Provider | 클래스형 Provider |
|-|---------------|----------------|
| 선언 | `@riverpod Future<T> func(Ref ref)` | `@riverpod class Notifier extends _$Notifier` |
| 용도 | 읽기 전용 (GET) | CRUD 전체 |
| 상태 변경 | 불가 (외부에서 `invalidate`만) | 메서드로 직접 변경 |
| 복잡도 | 단순 | 약간 복잡 |

**규칙**: 읽기만 하면 함수형, 쓰기도 하면 클래스형.

## 7. 전체 데이터 흐름 정리

이제 전체 파이프라인을 한 번에 보자.

### 요청 (뷰 → 서버)

```
1. 뷰에서 ref.watch(todoListNotifierProvider)
2. Provider의 build() 실행
3. Retrofit의 getTodos() 호출
4. Dio가 HTTP 요청 생성
5. AuthInterceptor가 토큰 자동 첨부
6. 서버로 GET /todos 전송
```

### 응답 (서버 → 뷰)

```
1. 서버가 JSON 응답
2. Dio가 응답 수신
3. Retrofit이 fromJson으로 List<Todo> 변환
4. Provider 상태에 저장 (AsyncData)
5. ref.watch가 변경 감지
6. 뷰의 .when()이 data 분기로 렌더링
```

### 에러 흐름 (토큰 만료)

```
1. 서버가 401 응답
2. Dio의 AuthInterceptor가 가로챔
3. Refresh Token으로 새 Access Token 발급
4. 원래 요청 재시도
5. 성공하면 정상 응답 흐름으로
6. 실패하면 Provider가 AsyncError 상태
7. 뷰의 .when()이 error 분기로 렌더링
```

### 각 레이어의 역할

```
┌─────────────┐
│     뷰      │  ref.watch + .when()으로 상태별 UI
├─────────────┤
│  Provider   │  상태 관리 + 캐싱 + 새로고침
├─────────────┤
│  Retrofit   │  API 인터페이스 → 구현 자동 생성
├─────────────┤
│    Dio      │  HTTP 통신 + 인터셉터 (토큰, 로깅)
├─────────────┤
│   Freezed   │  불변 모델 + JSON 변환 자동 생성
├─────────────┤
│    서버     │  REST API 제공
└─────────────┘
```

각 레이어가 자기 역할만 하고, 위아래로만 통신한다. 뷰는 Dio를 몰라도 되고, Retrofit은 인터셉터를 몰라도 된다. **관심사의 분리**가 자연스럽게 이루어진다.

## 느낀 점

Ch05에서 상태관리의 "왜"를 이해했다면, Ch06은 "무엇을" 관리하는가에 대한 답이다. 로컬 리스트가 아니라 **서버에서 오는 실제 데이터**를 다루니까 비로소 앱다운 앱이 된다.

처음에 Dio, Retrofit, Freezed, json_serializable, build_runner... 패키지가 너무 많아서 "이게 다 필요해?"라는 생각이 들었다. 하지만 하나씩 쌓아보니 각각이 **한 가지 문제를 확실히 해결**하고 있었다:

- **Dio** → HTTP 통신 + 인터셉터
- **Retrofit** → API 보일러플레이트 제거
- **Freezed** → 타입 안전한 불변 모델
- **Riverpod** → 서버 데이터의 상태 관리

iOS에서 Alamofire + Moya + Codable 조합을 쓰던 것과 구조적으로 비슷하다. 프레임워크는 달라도 **"네트워크 레이어를 추상화하고, 모델을 자동 생성하고, 상태로 관리한다"**는 패턴은 동일하다.

다음 챕터에서는 이걸 실제 앱에 적용해본다.
