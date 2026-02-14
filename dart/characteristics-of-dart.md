# 1. Dart의 특징

- **Multi-platform**: 모바일, 데스크톱, 서버, 웹 어디서든 실행 가능.
- **JIT & AOT**: 개발 중엔 빠른 피드백(Hot Reload), 배포 시엔 빠른 성능을 제공.
- **Single-Thread**: 기본적으로 싱글 스레드 언어이며 Event Loop를 기반으로 비동기 처리.

## 1-1. 멀티 플랫폼 전략: 어떻게 어디서든 실행될까?

Dart는 설계 당시부터 Client-Optimized(클라이언트 최적화)를 목표로 했습니다. 핵심은 **컴파일 타겟의 유연성**에 있습니다.

### 1) 네이티브 플랫폼 (Mobile, Desktop)

- **Dart VM**: 개발 중에는 자바의 JVM처럼 Dart VM 위에서 코드가 실행됩니다.
- **머신 코드 생성**: 배포 시에는 코드를 각 OS 환경에 맞는 **ARM**이나 **x64** 머신 코드로 직접 컴파일합니다. 중간 매개체 없이 하드웨어와 직접 소통하므로 네이티브 앱과 동일한 성능을 냅니다.

### 2) 웹 플랫폼 (Web)

- **Dart2JS**: Dart 코드를 고도로 최적화된 **JavaScript** 코드로 변환합니다.
- **WebAssembly (Wasm)**: 최근에는 JS를 거치지 않고 Wasm으로 직접 컴파일하여 웹에서도 네이티브급 성능을 낼 수 있도록 진화하고 있습니다.

## 1-2. JIT & AOT: 생산성과 성능의 공존

Dart의 가장 큰 마법은 개발할 때는 JIT(Just-In-Time)를 쓰고, 배포할 때는 AOT(Ahead-Of-Time)를 쓴다는 점입니다.

### 1) JIT (Just-In-Time): 실시간 컴파일

- **개념**: 앱이 실행되는 동안 필요한 시점에 실시간으로 코드를 컴파일합니다.
- **장점**: **Hot Reload**가 가능합니다. 소스 코드를 수정하면 전체 앱을 다시 빌드할 필요 없이, 바뀐 부분의 중간 코드(Bytecode)만 실행 중인 앱에 주입하면 즉시 반영됩니다.
- **단점**: 실행 중에 컴파일러가 계속 돌아가야 하므로 앱 실행 속도와 초기 구동(Startup time)이 상대적으로 느립니다.

### 2) AOT (Ahead-Of-Time): 사전 컴파일

- **개념**: 앱을 배포하기 전(빌드 타임)에 미리 타겟 장치의 기계어로 전부 컴파일해 버립니다.
- **장점**: 컴파일러가 이미 최적화를 끝낸 상태이므로 **실행 속도가 매우 빠르고 버벅임(Jank)이 거의 없습니다.** 앱을 켜자마자 즉시 실행됩니다.
- **단점**: 빌드 시간이 오래 걸리고, 코드를 수정할 때마다 전체를 다시 컴파일해야 하므로 개발 효율이 떨어집니다.

### 💡 왜, 어떻게 같이 쓰나?

- **개발 모드(Debug)**: **JIT**를 사용합니다. 성능보다는 개발자의 생산성(Hot Reload)이 중요하기 때문입니다.
- **배포 모드(Release)**: **AOT**를 사용합니다. 개발자 편의성보다는 사용자의 체감 성능과 부드러운 애니메이션(60/120fps)이 최우선이기 때문입니다.

## 1-3. 싱글 스레드와 이벤트 루프

Dart는 기본적으로 **싱글 스레드** 언어입니다. 하지만 우리가 수많은 API 통신과 UI 애니메이션을 동시에 처리할 수 있는 이유는 **이벤트 루프** 덕분입니다.

### 1) Isolate: Dart의 실행 단위

Dart는 '스레드' 대신 Isolate(아이솔레이트)라는 개념을 씁니다.

- **독립된 메모리**: 각 Isolate는 자신만의 고유한 메모리를 가집니다. 다른 Isolate와 메모리를 공유하지 않습니다.
- **장점**: 메모리 공유가 없으므로 멀티 스레드 프로그래밍에서 흔히 발생하는 **경쟁이나 데드락(Deadlock) 문제가 원천적으로 발생하지 않습니다.**

### 2) 이벤트 루프의 동작 원리

Isolate는 시작되자마자 **이벤트 루프**를 실행합니다. 루프는 두 개의 큐(Queue)를 무한히 돌며 확인합니다.

1. **Microtask Queue**: 가장 높은 우선순위를 가집니다. "지금 당장 처리해야 할 짧은 작업"들이 담깁니다. 이 큐가 비워질 때까지 다음 단계로 넘어가지 않습니다.
2. **Event Queue**: API 응답, 타이머 알림, 사용자 터치, 화면 그리기 신호 등이 담깁니다.

```dart
[동작 순서]
1. main() 함수 실행
2. Microtask Queue 확인 및 처리 (전부 비워질 때까지)
3. Event Queue에서 작업 하나 꺼내서 처리
4. 다시 Microtask Queue 확인 (새로 생긴 게 있다면 처리)
5. 반복...
```

### 3) 비동기 처리 (Future)

Future 객체는 나중에 Event Queue에 담길 예약표라고 이해하면 쉽습니다.

- await를 만나면 현재 작업을 잠시 멈추고 제어권을 이벤트 루프에 넘깁니다.
- 비동기 작업이 완료되면 해당 결과값이 **Event Queue**에 들어가고, 자기 차례가 오면 다시 실행을 재개합니다.

## 요약

### 🌍 멀티 플랫폼 아키텍처

- **컴파일 유연성**: 모바일(ARM/x64), 웹(JS/Wasm), 데스크톱 네이티브 코드를 모두 생성 가능.
- **Flutter 엔진과의 결합**: 하드웨어 가속 렌더링을 위해 최적화된 실행 환경 제공.

### ⚡ JIT vs AOT 컴파일 전략

- **JIT (Just-In-Time)**: 개발용. 런타임 컴파일을 통해 **Hot Reload** 지원 (생산성 극대화).
- **AOT (Ahead-Of-Time)**: 배포용. 빌드 시 기계어로 완전 컴파일하여 **고성능 및 빠른 시작** 보장.

### 🔄 싱글 스레드 & 이벤트 루프

- **Isolate**: Dart의 최소 실행 단위. 메모리를 공유하지 않아 공유 자원 경쟁(Lock) 문제 방지.
- **Event Loop**: Microtask와 Event 큐를 관리하여 싱글 스레드 환경에서도 효율적인 비동기 처리 수행.
- **비동기 모델**: `Future`와 `Stream`을 통해 I/O 작업 중에도 UI가 멈추지 않는 Non-blocking 환경 제공.

# 2. 변수와 상수

Dart는 정적 타입 언어이지만 `var`를 통한 타입 추론을 지원합니다.

## 2-1. final vs const

| 특징 | final | const |
| --- | --- | --- |
| **성격** | 런타임 상수 | 컴파일 타임 상수 |
| **초기화 시점** | 실행 시 값이 결정됨 (예: API 응답값) | 빌드 시 값이 결정되어야 함 (예: '1.0.0') |
| **메모리** | 호출 시 할당 | 컴파일 시 할당되어 재사용 (Canonicalization) |

```dart
void main() {
  // final: 실행할 때마다 값이 달라질 수 있음
  final DateTime now = DateTime.now(); 
  
  // const: 빌드할 때 이미 값이 정해져 있어야 함
  const String version = '1.0.1';
  // const DateTime errorTime = DateTime.now(); // Error! (컴파일 시점엔 시간을 알 수 없음)

  // 활용 예시 (Flutter 최적화)
  // const 위젯은 메모리에 한 번만 올라가고 리빌드 시 재사용됨
}
```

## 2-2. 변수 선언 방식

### var (타입 추론)

var는 초기화되는 값에 따라 타입을 자동으로 결정합니다. 한 번 타입이 결정되면 다른 타입의 값을 넣을 수 없습니다.

```dart
void main() {
  var name = 'My Name'; // String으로 추론
  // name = 100; // Error: A value of type 'int' can't be assigned to a variable of type 'String'
  
  String explicitName = 'My Name'; // 명시적 선언 (협업 시 권장)
}
```

# 3. Null Safety

Dart 2.12부터 도입된 Sound Null Safety는 런타임 에러를 방지합니다. 모든 변수는 기본적으로 Non-nullable이며, 명시적으로 허용했을 때만 null을 가질 수 있습니다.

```dart
String name = "My Name"; // Non-nullable
String? nickname; // Nullable

// late: 초기화를 나중에 하겠다는 약속
late String description;

void main() {
  description = "Mobile Developer";
  print(description);
}
```

## 3-1. Nullable 변수 (?)

타입 뒤에 ?를 붙여 Null이 가능함을 명시합니다.

```dart
String name = 'My Name'; // 절대 null이 될 수 없음
String? nickname; // null 허용 (기본값 null)
```

## 3-2. Null 관련 연산자

### **1) ?. (Null-aware access)**

객체가 null이면 null을 반환하고, 아니면 해당 속성에 접근합니다.

```dart
print(nickname?.length); // nickname이 null이면 null 출력, 아니면 길이 출력
```

### **2) ?? (Null-coalescing operator)**

좌항이 null이면 우항의 값을 사용합니다. (기본값 설정 시 유용)

```dart
String displayName = nickname ?? 'Guest'; // nickname이 null이면 'Guest' 할당
```

### **3) ! (Null assertion operator)**

컴파일러에게 "이 변수는 절대 null이 아니니까 믿고 써!"라고 강제합니다. (런타임 에러 위험)

```dart
String confirmedName = nickname!; // nickname이 null이면 앱 크래시 발생
```

### **4) ??= (Null-aware assignment)**

변수가 null일 때만 값을 할당합니다.

```dart
nickname ??= 'Newbie'; // nickname이 null이었으므로 'Newbie'가 들어감
```

## 3-3. late 키워드 (지연 초기화)

선언 시점에는 값을 넣지 않지만, **사용하기 전에는 반드시 초기화하겠다**고 약속하는 키워드입니다. (Kotlin의 lateinit과 동일)

- **언제 쓰나?**: initState에서 초기화해야 하는 경우나, 초기화 비용이 커서 실제 사용 시점까지 미루고 싶을 때.

```dart
class UserProfile {
  late String _bio;

  void init() {
    _bio = 'I love Dart!'; // 사용 전 초기화
  }

  void printBio() {
    print(_bio); // 만약 초기화 없이 호출하면 LateInitializationError 발생
  }
}
```

## 💡 언제 무엇을 사용하면 좋을까?

1. **기본적으로 final을 사용하세요.**
    - 데이터가 변하지 않는다면 final로 선언하는 것이 가독성과 안정성 측면에서 가장 좋습니다.
2. **완전한 고정 값(코드 상수)은 const를 사용하세요.**
    - 에러 메시지, API 주소, 디자인 수치(Padding, Margin) 등은 const로 선언하여 메모리 효율을 높입니다.
3. **함수 내부의 짧은 로직에서는 var를 사용해도 좋습니다.**
    - 타입이 명확히 추론되는 경우 코드가 간결해집니다.
4. **!(Assertion) 보다는 ??(Default)를 사용하세요.**
    - 앱의 안정성을 위해 null일 경우의 대체 값을 제공하는 습관이 중요합니다.

## 요약

### 📌 변수 선언 전략

- `var`: 타입 추론. 로컬 변수에서 주로 사용.
- `final`: 런타임 상수. 한 번만 할당 가능.
- `const`: 컴파일 타임 상수. 메모리 정준화(Canonicalization)를 통해 성능 최적화.

### 🛡️ Null Safety 연산자

- `Type?`: Nullable 선언.
- `obj?.prop`: Null 안전 접근.
- `value ?? 'default'`: Null일 경우 기본값 제공.
- `value!`: Null이 아님을 강제 (주의해서 사용).
- `late`: 지연 초기화. 초기화 시점을 뒤로 미룸.

### 🚀 예제 코드

```dart
void main() {
  final name = 'Name';
  const pi = 3.14;

  String? message;
  print(message?.length ?? 0); // 안전한 설계
}
```

# 4. 매개변수(Parameters)와 특수 연산자

## 4-1. 매개변수(Parameters)의 종류

Dart는 매개변수를 크게 두 가지 방식으로 정의합니다.

### 1) 위치 지정 매개변수 (Positional Parameters)

가장 전통적인 방식입니다. 인자의 순서가 중요합니다.

```dart
// 필수 위치 인자
void printInfo(String name, int age) {
  print('$name is $age years old');
}

// 선택적 위치 인자: []를 사용하며 맨 뒤에 위치해야 함
void printLog(String message, [String? tag]) {
  print('[${tag ?? 'INFO'}] $message');
}

void main() {
  printInfo('Name', 29); // 순서 엄수
  printLog('Hello'); // 선택적 인자 생략 가능
}
```

### 2) 이름 지정 매개변수 (Named Parameters) ⭐

함수 호출 시 {}를 사용하여 매개변수의 이름을 명시합니다. **Flutter 위젯 생성자의 99%가 이 방식을 사용합니다.**

- **가독성**: 인자가 많아져도 각 값이 무엇을 의미하는지 명확합니다.
- **유연성**: 호출 시 인자의 순서를 바꿔도 상관없습니다.
- **필수 여부**: required 키워드로 필수 인자를 지정합니다.

```dart
void createUser({
  required String id,
  required String name,
  int? age, // 선택 사항
  String country = 'Korea', // 기본값 설정 가능
}) {
  print('ID: $id, Name: $name, Age: $age, Country: $country');
}

void main() {
  // 호출 시 이름을 명시하므로 순서가 바뀌어도 무관함
  createUser(
    name: 'My Name',
    id: 'admin_01',
  );
}
```

## 4-2. 왜 Flutter는 Named Parameter를 사랑할까?

Android XML이나 Java/Kotlin의 일반적인 생성자와 비교해 보면 명확합니다.

**Java/Kotlin 스타일 (위치 기반): (물론 최신 Java, Kotlin도 네임드 파라미터를 지원합니다.)**

```dart
// 인자가 많아지면 각 숫자가 padding인지, margin인지 알기 어려움
val widget = MyWidget("Title", 10.0, 20.0, true)
```

**Flutter 스타일 (이름 기반):**

```dart
Column(
  mainAxisAlignment: MainAxisAlignment.center,
  crossAxisAlignment: CrossAxisAlignment.start,
  children: [ ... ],
)
```

위젯은 수많은 스타일 속성을 가집니다. Named Parameter를 사용하면 **"설계도(Widget)"에 필요한 값만 골라서 이름을 적어주는 방식**이 되므로 코드가 훨씬 직관적인 문서처럼 읽히게 됩니다.

## 4-3. 특수 연산자

### 1) 캐스케이드 연산자 (.., ?..)

동일한 객체에 대해 연속적인 작업을 수행할 때 사용합니다. 객체를 반환하지 않는 메서드들을 체이닝할 수 있습니다. (**Kotlin의 apply 확장 함수와 매우 유사합니다.)**

```dart
class Player {
  String? name;
  int level = 0;
  void sayHello() => print('Hi, I am $name');
}

void main() {
  // 일반적인 방식
  var p1 = Player();
  p1.name = 'My Name';
  p1.level = 10;
  p1.sayHello();

  // 캐스케이드 방식
  var p2 = Player()
    ..name = 'Dart'
    ..level = 5
    ..sayHello();
    
  // Null-aware 캐스케이드 (?..)
  // 객체가 null이면 하위 작업을 아예 수행하지 않음
  Player? p3;
  p3?..name = 'Unknown'
    ..sayHello();
}
```

### 2) 스프레드 연산자 (..., ...?)

컬렉션(List, Map, Set)을 다른 컬렉션 안에 펼쳐서 넣을 때 사용합니다. UI 리스트를 조건부로 합칠 때 매우 유용합니다.

```dart
void main() {
  var baseColors = ['Red', 'Green'];
  var allColors = ['Blue', ...baseColors]; // ['Blue', 'Red', 'Green']

  List<String>? optionalItems;
  var finalItems = [
    'Home',
    ...?optionalItems, // optionalItems가 null이면 무시하고 진행
  ];
}
```

## 요약

### 📌 매개변수 시스템

- **Positional**: 순서가 중요한 일반적인 매개변수. `[]`로 선택적 위치 인자 선언 가능.
- **Named**: `{}`를 사용하며 호출 시 이름을 명시.
    - `required`: 필수 인자 표시.
    - 가독성이 뛰어나 Flutter 위젯 생성자에 필수적으로 사용됨.

### 🛠️ 주요 연산자

- **Cascade (`..`)**: 객체 생성 후 연속적으로 필드나 메서드에 접근할 때 사용 (Kotlin의 `apply`와 유사).
- **Null-aware Cascade (`?..`)**: 객체가 Null이 아닐 때만 캐스케이드 수행.
- **Spread (`...`)**: 리스트나 맵을 다른 컬렉션에 펼쳐서 삽입.
- **Null-aware Spread (`...?`)**: 컬렉션이 Null이 아닐 때만 펼치기 수행.

### 🚀 실무 팁

Flutter에서 `build()` 메서드 내의 위젯들은 대부분 **Named Parameters**로 구성됩니다. 인자가 3개 이상 넘어가는 함수를 만든다면 가독성을 위해 Named Parameter 사용을 적극 권장합니다.

```dart
// Good Example in Flutter
Text(
  'Hello Dart',
  style: TextStyle(
    fontSize: 20,
    fontWeight: FontWeight.bold,
  ),
)
```