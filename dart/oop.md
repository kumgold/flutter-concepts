# 1. Factory Constructor

가장 독특하고 중요한 개념입니다. 일반 생성자와 달리 **새로운 인스턴스를 생성하지 않고도 호출**할 수 있습니다.

- **특징**: 생성자 내부에서 return 키워드를 사용할 수 있습니다.
- **주요 용도**:
    1. **싱글톤(Singleton)** 패턴 구현
    2. 서브클래스의 인스턴스 반환
    3. 캐싱 로직 구현 (이미 생성된 객체 재사용)

## 1-1. 핵심 개념

일반 생성자는 호출될 때 **무조건 새로운 인스턴스**를 생성하여 반환합니다. 하지만 factory 키워드가 붙은 생성자는 다음과 같은 유연함을 가집니다.

- **인스턴스 재사용 가능**: 매번 새로 만드는 대신 기존에 생성된 인스턴스를 반환할 수 있습니다.
- **서브클래스 반환 가능**: 해당 클래스가 아닌 자식 클래스의 인스턴스를 반환할 수 있습니다.
- **비즈니스 로직 포함 가능**: 생성 단계에서 조건문 등을 통해 어떤 객체를 반환할지 결정할 수 있습니다.

> **⚠️ 주의사항**: 팩토리 생성자 내부에서는 this 키워드에 접근할 수 없습니다. (인스턴스가 아직 생성되기 전이거나 이미 생성된 것을 반환하기 때문)
>

## 1-2. 주요 용도 및 실전 예제

### 1) 싱글톤(Singleton) 패턴 구현

가장 많이 쓰이는 방식입니다. 앱 전체에서 딱 하나의 인스턴스만 유지해야 하는 Database나 AuthService 등에 적합합니다.

```dart
class AuthService {
  // 1. 내부에서 사용할 정적 인스턴스 변수
  static final AuthService _instance = AuthService._internal();

  // 2. factory 생성자: 항상 동일한 _instance를 반환
  factory AuthService() {
    return _instance;
  }

  // 3. 내부에서만 호출 가능한 '이름 지정 생성자' (Private)
  AuthService._internal() {
    print("서비스 로직 초기화...");
  }
}

void main() {
  var s1 = AuthService();
  var s2 = AuthService();

  print(identical(s1, s2)); // true (두 객체는 완벽하게 동일함)
}
```

### 2) JSON 데이터를 객체로 변환 (Data Model)

Flutter 개발 시 API 응답(Map)을 모델 객체로 바꿀 때 전형적으로 사용하는 방식입니다.

```dart
class User {
  final int id;
  final String name;

  User({required this.id, required this.name});

  // 팩토리 생성자를 통해 JSON 데이터를 파싱하여 객체 반환
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as int,
      name: json['name'] as String,
    );
  }
}
```

### 3) 조건에 따른 서브클래스 반환 (다형성 활용)

특정 조건에 따라 서로 다른 기능을 하는 자식 클래스를 반환할 수 있습니다.

```dart
abstract class Logger {
  factory Logger(String type) {
    if (type == 'cloud') return CloudLogger();
    return FileLogger();
  }
  
  void log(String message);
}

class FileLogger implements Logger {
  @override
  void log(String message) => print("파일에 저장: $message");
}

class CloudLogger implements Logger {
  @override
  void log(String message) => print("클라우드 전송: $message");
}

void main() {
  // 호출하는 쪽에서는 어떤 구체 클래스가 올지 알 필요가 없음
  final logger = Logger('cloud');
  logger.log("에러 발생!"); // "클라우드 전송: 에러 발생!" 출력
}
```

## 3. 일반 생성자 vs 팩토리 생성자 비교

| **구분** | **일반 생성자 (Generative)** | **팩토리 생성자 (factory)** |
| --- | --- | --- |
| **반환값** | 새로운 인스턴스를 즉시 생성 | 기존 인스턴스 혹은 자식 인스턴스 선택 가능 |
| **return 키워드** | 사용 불가 | **사용 필수 (인스턴스를 반환해야 함)** |
| **사용 목적** | 필드 초기화 및 객체 생성 | 싱글톤, 캐싱, 타입 분기 로직 |
| **this 접근** | 가능 | 불가능 |

---

# 2. Mixin (믹스인) - with 키워드

다중 상속의 단점(다이아몬드 문제)을 해결하면서 **여러 클래스의 기능을 한 곳에 주입**할 수 있는 방법입니다.

- **개념**: 클래스 계층 구조에 포함되지 않고 코드를 재사용하는 방법입니다.
- **제약**: 생성자를 가질 수 없습니다.
- **Android 개발자와의 비교**: Kotlin의 Interface with Default Implementation과 유사하지만, Mixin은 상태(변수)를 가질 수 있고 이를 그대로 주입한다는 점이 더 강력합니다.

## 2-1. 핵심 개념

Mixin은 상속하지 않고 클래스에 기능을 추가하는 방법입니다.

- **다중 상속의 대안**: Dart는 클래스 간의 다중 상속을 허용하지 않지만, Mixin을 사용하면 여러 개의 기능을 한 클래스에 섞어 넣을 수 있습니다.
- **독립적 존재**: Mixin 자체는 인스턴스화할 수 없습니다 (new 불가).
- **생성자 없음**: 생성자를 가질 수 없습니다. 오직 기능(메서드)과 상태(변수)만 전달합니다.

## 2-2. 왜 Mixin인가? (Android와의 비교)

안드로이드(Kotlin)에서 여러 클래스에 공통 기능을 넣으려면 보통 다음 두 가지 방식을 씁니다.

1. **Interface with Default Method**: 기능을 정의할 순 있지만, 상태(필드 변수)를 가질 수 없습니다.
2. **Composition (위임)**: 별도의 객체를 만들어 내부 변수로 들고 있어야 합니다.

**하지만 Dart의 Mixin은 상태(변수)까지 통째로 주입할 수 있습니다.**

## 2-3. 실전 예제

### 1) 기본 사용법 (mixin & with)

```dart
// 1. mixin 정의
mixin Flyer {
  void fly() => print("하늘을 날아갑니다!");
}

mixin Swimmer {
  void swim() => print("물속을 헤엄칩니다!");
}

// 2. 상속(extends)과 믹스인(with) 결합
class Animal {
  void eat() => print("음식을 먹습니다.");
}

// 오리는 동물이면서 날 수도 있고 헤엄칠 수도 있음
class Duck extends Animal with Flyer, Swimmer {}

void main() {
  var duck = Duck();
  duck.eat();  // Animal에서 상속
  duck.fly();  // Flyer에서 주입됨
  duck.swim(); // Swimmer에서 주입됨
}
```

### 2) on 키워드: 사용 제한하기

특정 클래스를 상속받은 클래스에서만 이 Mixin을 쓸 수 있도록 제한할 수 있습니다. (안정성 확보)

```dart
abstract class Programmer {
  void coding();
}

// 이 Mixin은 Programmer의 자식 클래스에서만 사용 가능함
mixin FlutterSkill on Programmer {
  void buildApp() {
    coding(); // Programmer의 메서드에 접근 가능!
    print("Flutter 앱을 빌드합니다.");
  }
}

class SeniorDeveloper extends Programmer with FlutterSkill {
  @override
  void coding() => print("다트 코드를 작성합니다.");
}

// Error! Animal은 Programmer가 아니므로 FlutterSkill을 섞을 수 없음
// class Dog extends Animal with FlutterSkill {}
```

## 2-4. Flutter 실무에서의 Mixin 활용

Flutter 프레임워크를 사용하다 보면 자신도 모르게 Mixin을 자주 접하게 됩니다.

- **WidgetsBindingObserver**: 앱의 생명주기(포그라운드/백그라운드 전환)를 감지할 때 with 키워드로 추가합니다.
- **SingleTickerProviderStateMixin**: 애니메이션을 구현할 때 Ticker(박자)를 공급받기 위해 사용합니다.

```dart
class _MyAnimationState extends State<MyWidget> with SingleTickerProviderStateMixin {
  // SingleTickerProviderStateMixin이 제공하는 기능을 통해
  // AnimationController를 초기화할 수 있음
  late AnimationController _controller = AnimationController(vsync: this); 
}
```

## 2-5. Mixin의 우선순위 (Linearization)

여러 개의 Mixin을 섞었을 때 동일한 이름의 메서드가 있다면, **가장 마지막에 작성된 Mixin이 우선순위**를 가집니다.

```dart
mixin A { void test() => print("A"); }
mixin B { void test() => print("B"); }

class Test with A, B {}

void main() {
  Test().test(); // "B" 출력 (오른쪽에 가까울수록 덮어씀)
}
```

---

# 3. Implicit Interfaces (암시적 인터페이스)

Dart에는 interface라는 키워드가 따로 없습니다. **모든 클래스는 암시적으로 인터페이스 역할을 수행**할 수 있습니다.

- **extends**: 상속 (구현을 물려받음, 하나만 가능)
- **implements**: 인터페이스 구현 (모든 추상 멤버를 재정의해야 함, 여러 개 가능)

```dart
class Person {
  final String name;
  Person(this.name);
  void greet() => print('Hi, I am $name');
}

// Person 클래스의 '설계'만 가져와서 완전히 새로 구현
class Imposter implements Person {
  @override
  String get name => "I'm fake";

  @override
  void greet() => print("I'm not real");
}
```

## 3-1. 개념 및 사용법

어떤 클래스를 선언하면, Dart는 해당 클래스의 멤버들을 포함하는 인터페이스를 자동으로 생성합니다. 이 클래스를 implements 하면, 부모의 **구현(Body)은 무시하고 설계(Signature)만** 가져옵니다.

```dart
class Person {
  final String _name;
  Person(this._name);

  void greet() => print("안녕하세요, 저는 $_name입니다.");
}

// implements를 사용하면 Person의 '설계도'만 가져옴
class MockPerson implements Person {
  @override
  String get _name => "가짜"; // 추상 멤버처럼 반드시 재정의 필요

  @override
  void greet() => print("저는 가짜 객체라 인사를 못 해요.");
}
```

## 3-2. 왜 암시적 인터페이스를 사용하는가?

1. **Boilerplate(상용구) 감소**: Java처럼 interface Service와 class ServiceImpl을 따로 만들 필요가 없습니다. 그냥 Service 클래스 하나만 만들고, 테스트할 때만 implements Service 하여 Mock 객체를 만들면 됩니다.
2. **유연한 API 설계**: 외부 라이브러리의 구체 클래스도 내가 필요하면 인터페이스로 취급하여 완전히 새로운 구현체를 만들 수 있습니다.
3. **다중 구현 지원**: extends는 하나만 가능하지만, implements는 여러 클래스를 나열할 수 있어 다중 상속의 효과를 낼 수 있습니다.

## 3-3. Mixin vs Interface (무엇이 다른가?)

안드로이드 개발자로서 가장 헷갈릴 수 있는 부분입니다. 핵심은 코드를 재사용하느냐, 껍데기만 가져오느냐입니다.

| **구분** | **Interface (implements)** | **Mixin (with)** |
| --- | --- | --- |
| **목적** | **계약(Contract)** 정의 | **코드 재사용(Reuse)** 및 주입 |
| **구현 포함** | 불가능 (모든 메서드 재정의 필수) | **가능** (구현된 메서드를 그대로 사용) |
| **상태(변수)** | 설계만 가져옴 (필드 재정의 필수) | **변수(State)까지 통째로 주입됨** |
| **관계** | "~처럼 동작한다" (Capability) | "~의 기능을 가진다" (Component) |

### 💡 정리하자면:

- **Interface**는 "너 이 메서드들 반드시 만들어야 해!"라고 **강제**할 때 씁니다.
- **Mixin**은 "이 기능 미리 만들어놨으니까 가져다 써!"라고 **제공**할 때 씁니다.

---

# 4. Extension Methods (확장 메서드)

기존에 정의된 클래스나 외부 라이브러리의 클래스에 **새로운 기능을 추가**할 때 사용합니다. Kotlin의 Extension과 매우 흡사합니다.

```dart
// String 클래스에 기능을 확장
extension StringExtensions on String {
  bool get isEmail {
    return contains('@');
  }
}

void main() {
  String email = "good@example.com";
  print(email.isEmail); // true
}
```