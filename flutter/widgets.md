# Flutter에서 위젯이란?

## 1. 위젯은 UI가 아니다 (Widget ≠ UI)
Android의 View나 iOS의 UIView는 화면에 픽셀을 그리고, 터치 이벤트를 처리하며, 자기 자신의 상태를 직접 들고 있는 **무거운 객체(Heavyweight Object)** 입니다.
   
하지만 Flutter의 Widget은 UI 그 자체가 아닙니다. Widget은 "UI를 어떻게 구성할 것인가?"에 대한 설정 정보를 담고 있는 **가벼운 설계도(Lightweight Blueprint)** 입니다.

### 설계도와 건물의 비유
- Widget (설계도): "여기에 파란색 배경의 100x100 크기 상자를 그려줘"라는 종이 한 장. (매우 가볍고 버리기 쉬움)
- RenderObject (건물): 실제로 메모리를 점유하고 픽셀을 칠하며, 레이아웃 계산과 페인팅을 담당하는 실제 객체. (매우 무겁고 재사용이 중요함)

UI가 아니다. Android의 View, iOS의 UIView는 화면에 픽셀을 그리고, 터치 이벤트를 받고, 자신의 상태를 갖고 있는 무거운 객체이다. 하지만 Flutter의 Widget은 UI 자체가 아니다.


## 2. 불변성: 왜 @immutable 인가?
Flutter의 소스 코드를 보면 Widget 클래스에는 @immutable 어노테이션이 붙어 있습니다. 이는 생성된 후 내부 상태를 절대 바꿀 수 없음을 의미합니다.
   
```dart
@immutable
abstract class Widget extends DiagnosticableTree {
   const Widget({ this.key });
   final Key? key; // 모든 필드는 final이어야 함
}
```
   
### 왜 수정(Modify)하지 않고 교체(Replace)하는가?
Native View 시스템처럼 특정 속성만 수정(view.color = blue)하면 빠를 것 같지만, 실제로는 그렇지 않습니다.

1. <b>연산의 단순화:</b> 속성 하나가 바뀌었을 때 이전 상태와 비교하여 무엇이 변했는지 추적하는 것보다, 전체 설계도를 새로 전달하고 엔진이 바뀐 부분만 찾아내는 것이 수학적으로 더 깔끔하고 빠릅니다.
2. <b>Dart의 세대별 가비지 컬렉션(Generational GC):</b> Dart는 수만 개의 짧은 수명을 가진 객체(위젯)를 생성하고 파괴하는 데 최적화되어 있습니다. 초당 120번 위젯을 새로 만들어도 성능 저하가 거의 없습니다.
3. <b>Thread Safety:</b> 객체가 변하지 않으므로 비동기 작업 중 데이터 오염 걱정이 없습니다.


## 3. 화면 갱신 메커니즘 (Replacement)
사용자가 setState()를 호출하면 Flutter 내부에서는 다음과 같은 일이 일어납니다.

1. <b>Rebuild:</b> 해당 위젯의 build() 메서드가 다시 실행되어 새로운 위젯 트리를 생성합니다.
2. <b>Diffing (엘리먼트 트리의 역할):</b> Flutter 엔진은 기존 설계도(Old Widget)와 새 설계도(New Widget)를 비교합니다.
3. <b>Update:</b> "어? 색상만 빨간색에서 파란색으로 바뀌었네?"라고 판단하면, 무거운 실제 객체인 RenderObject는 그대로 두고 속성값만 갱신합니다.

>💡 핵심 요약: 위젯 트리는 매 프레임 파괴되고 재생성되지만, 실제 화면을 그리는 RenderObject 트리는 위젯 설계도의 변경사항만 반영하며 끈질기게 살아남습니다.


## 4. const 키워드: 렌더링 파이프라인의 지름길
```dart
// Case A: 매번 새로운 설계도를 인쇄함
Container(child: Text('Hello'));

// Case B: 이미 인쇄된 설계도를 재사용함
const Container(child: const Text('Hello'));
```   

- <b>Canonicalization (정준화):</b> const 위젯은 컴파일 타임에 메모리의 단 한 곳에만 생성됩니다.
- <b>성능 이점:</b> build()가 아무리 많이 호출되어도 const가 붙은 위젯은 메모리 주소값이 동일합니다.
- <b>Short-circuiting:</b> Flutter 엔진은 주소값이 같으면 "이 아래 자식들은 볼 것도 없이 똑같겠군!"이라며 **하위 트리에 대한 모든 연산과 비교를 통째로 생략(Skip)**합니다. 이것이 const가 가져다주는 진정한 성능 최적화입니다.


## 5. StatelessWidget vs StatefulWidget
### 5-1. StatelessWidget (상태가 없는 설계도)
- 한 번 전달받은 데이터가 변하지 않는 UI를 정의합니다.
- build 메서드는 오직 부모로부터 전달받은 정보나 외부 의존성 변화에 의해서만 호출됩니다.
- Side Effect 금지: build 내부에서 HTTP 요청을 보내는 등의 행위는 절대 금물입니다. (매 프레임 호출될 수 있기 때문)
   
### 5-2. StatefulWidget (상태를 추적하는 설계도)
위젯 자체는 여전히 불변이지만, 별도의 State 객체를 두어 변화를 관리합니다.

- 구조적 분리: Widget 객체는 계속 교체되지만, State 객체는 메모리에 유지됩니다.
- 생명주기 유지: 화면에서 위젯 설계도가 갈아끼워지는 동안에도 State는 살아남아 사용자의 입력값이나 서버 데이터를 유지합니다.
- setState(): "설계도가 바뀔 데이터가 생겼으니 build를 다시 해달라"고 프레임워크에 요청하는 트리거입니다.

```dart
class CounterWidget extends StatefulWidget {
    @override
    State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
    int _counter = 0; // 이 값은 위젯이 재생성되어도 유지됨

    @override
    Widget build(BuildContext context) {
        return Text('$_counter');
    }
}
```


## Stateless Widget
- 상태변화가 없는 위젯. 주로 많은 위젯이 StatelessWidget으로 구현된다. (예: Text, Icon 등)
- build 메서드를 반드시 오버라이드 해야 한다. build 메서드는 Widget을 그리고 Widget Tree에 추가하는 역할을 하는 메서드이다.
- build 메서드는 Widget을 생성하거나 Widget의 Dependency(예를 들면, Widget에 전달된 State)가 변경될 때 호출된다.
- 모든 프레임에서 호출될 수 있기 때문에 어떤 Side Effect를 포함해서는 안된다.
```dart
class PaddedText extends StatelessWidget {
  const PaddedText({super.key});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(8.0),
      child: const Text('Hello, World!'),
    );
  }
}
```

## Stateful Widget
- 유저 Input에 따라 상태가 변하는 Widget은 Stateful Widget으로 구현한다.
- StatefulWidget은 build 메서드를 포함하지 않는다. State라는 Subclass에 변경 가능한 상태를 저장하여 표현한다.
- State Subclass에 build 메서드가 포함되어 있다. State 객체에 변경이 발생되었다면 반드시 setState를 호출해서 Framework에 변경되었다는 신호를 보내야 한다.
- Widget으로부터 State를 분리한다면 다른 Widget의 상태 손실 걱정 없이 StatelessWidget과 StatefulWidget을 같은 방식으로 처리할 수 있다.
```dart
 class CounterWidget extends StatefulWidget {
  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Text('$_counter');
  }
}
```

---

참고 자료 : <br>
https://docs.flutter.dev/get-started/fundamentals/state-management <br>
https://docs.flutter.dev/get-started/fundamentals/widgets
