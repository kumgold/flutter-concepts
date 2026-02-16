## 1. Reified Generics (실체화된 제네릭)

Dart의 제네릭은 Reified(실체화)되어 있습니다. 즉, 제네릭 타입 정보가 컴파일 타임뿐만 아니라 **런타임에도 유지**됩니다.

### Java vs Dart 비교

- **Java (Type Erasure)**: List<String>은 런타임에 그저 List가 됩니다. 따라서 list instanceof List<String> 같은 코드는 불가능합니다.
- **Dart (Reification)**: List<String>은 런타임에도 자신이 String의 리스트임을 알고 있습니다.

```dart
void main() {
  var names = <String>['name1', 'name2'];
  
  // ✅ Dart에서는 런타임에 제네릭 타입을 정확히 체크할 수 있습니다.
  print(names is List<String>); // true
  print(names is List<int>);    // false
  
  // 타입 정보가 그대로 유지되므로 타입을 출력할 수도 있습니다.
  print(names.runtimeType);     // List<String>
}
```

### Kotlin 개발자를 위한 팁

Kotlin에서는 런타임에 제네릭 타입을 확인하려면 inline 함수와 reified 키워드를 명시적으로 사용해야 하지만, **Dart는 모든 제네릭이 기본적으로 reified 상태**입니다.

## 2. Generic Methods & Classes

클래스나 함수를 선언할 때 타입을 파라미터화하여 재사용성을 극대화합니다.

```dart
// 제네릭 클래스
class DataHolder<T> {
  final T data;
  DataHolder(this.data);

  void printType() {
    // 런타임에도 T가 무엇인지 정확히 압니다.
    print("Type of T is: $T");
  }
}

// 제네릭 메서드
T firstElement<T>(List<T> list) {
  return list[0];
}

void main() {
  var holder = DataHolder<int>(100);
  holder.printType(); // "Type of T is: int"
}
```

## 3. 타입 시스템의 독특한 점: Covariance (공변성)

Dart의 타입 시스템은 다른 언어에 비해 다소 유연(Loose)한 면이 있는데, 그 대표적인 예가 **리스트의 공변성**입니다.

- **개념**: Dog가 Animal의 자식이라면, List<Dog>는 List<Animal>의 하위 타입으로 간주됩니다.
- **위험성**: 이로 인해 컴파일 타임에는 에러가 없지만, 런타임에 에러가 발생할 수 있습니다.

```dart
class Animal {}
class Dog extends Animal {}
class Cat extends Animal {}

void main() {
  // 1. Dog 리스트 생성
  List<Dog> myDogs = [Dog()];

  // 2. List<Dog>를 List<Animal>에 할당 (Dart에서는 허용됨 - Covariance)
  List<Animal> myAnimals = myDogs;

  // 3. Animal 리스트니까 Cat을 넣을 수 있을까?
  // 컴파일 타임 OK, 하지만 런타임에 TypeError 발생!
  // 왜냐하면 실제 알맹이는 List<Dog>이기 때문입니다.
  try {
    myAnimals.add(Cat()); 
  } catch (e) {
    print(e); // type 'Cat' is not a subtype of type 'Dog' of 'value'
  }
}
```

## 4. 제네릭 제약 (Generic Constraints)

extends 키워드를 사용하여 제네릭에 올 수 있는 타입을 제한할 수 있습니다.

```dart
// num(int, double) 혹은 그 하위 타입만 가능하도록 제한
class Calculator<T extends num> {
  T add(T a, T b) {
    // num의 기능을 사용할 수 있음
    return (a + b) as T;
  }
}
```

## **🔍 분석**

### 1. 왜 Reification을 선택했는가?

Reified 제네릭은 개발자에게 훨씬 직관적인 경험을 제공합니다. 런타임에 is T나 T.toString()을 자유롭게 쓸 수 있어, JSON 직렬화 라이브러리(JsonSerializable 등)나 DI 프레임워크(GetIt)를 만들 때 Java보다 훨씬 깔끔한 구현이 가능합니다.

### 2. 강력한 타입 체크 vs 유연성

Dart는 **Sound Type System**을 지향합니다. 하지만 앞서 본 '리스트 공변성' 같은 부분은 편의를 위해 열어둔 구멍이기도 합니다. 대규모 프로젝트에서는 List<Animal> list = <Dog>[]와 같은 코드를 지양하고, 타입을 명확히 일치시키는 컨벤션을 가지는 것이 중요합니다.

## 요약

### 💎 Reified Generics

- Dart의 제네릭은 **런타임에도 타입 정보가 유지**됩니다.
- Java의 Type Erasure와 달리, 실행 중에 `is List<String>`과 같은 타입 체크가 가능합니다.
- Kotlin의 `reified` 키워드 기능을 언어 차원에서 기본으로 제공합니다.

### 🔄 Covariance (공변성)

- `List<Dog>`를 `List<Animal>` 타입 변수에 할당할 수 있습니다.
- 이는 편리하지만, 런타임에 잘못된 타입을 삽입하려 할 때 `TypeError`를 유발할 수 있으므로 주의가 필요합니다.

### 🛠️ Generic Constraints

- `class Box<T extends Shape>`와 같이 사용하여 특정 클래스 계층으로 타입을 제한할 수 있습니다.

### 🚀 실무 예제 (Type Check)

```dart
void checkType<T>(dynamic data) {
  if (data is T) {
    print("Data matches the generic type $T");
  } else {
    print("Data is NOT a $T");
  }
}
```