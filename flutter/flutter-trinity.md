# The Trinity (Flutter’s Trees)

### 1. Widget Tree (개념적으로 트리)

- **역할** : 개발자가 코드로 작성한 설계도
- **특징** : 불변(Immutable). 아주 가볍다. 매 프레임마다 삭제되고 다시 만들어진다.
- **비유** : “벽지를 파란색으로 칠해라”라고 적힌 작업 지시서.

### 2. Element Tree (자료구조상 트리)

- **역할** : Widget과 RenderObject를 연결하는 중간 관리자.
- **특징** : 가변(Mutable). 화면에 계속 살아남아 상태(State)를 관리한다.
- **비유** : 현장 감독관. 작업 지시서(Widget Tree)가 바뀌면, 실제로 벽을 부술지 아니면 페인트만 칠할지 결정.
- **핵심** : BuildContext가 바로 Element이다.

### 3. RenderObject Tree (자료구조상 트리)

- **역할** : 실제 화면에 선을 긋고, 크기를 계산하고(Layout), 색칠하는(Paint) 객체.
- **특징** : 가변(Mutable). 아주 무겁고 비싸다. 가능한 한 재사용해야 한다.
- **비유** : 실제 시멘트 벽과 페인트공.

## 작동 원리 시나리오

개발자가 `setState`를 호출해서 텍스트 색상을 빨강 → 파랑으로 변경했다고 가정한다면,

### 1. Widget Tree

- `Text(color: Red)` 위젯이 버려진다.
- `Text(color: Blue)` 위젯이 새로 생성된다.

### 2. Element Tree

- Element는 자신의 짝이었던 Widget(빨강)이 사라지고 새 Widget(파랑)이 들어온 것을 감지한다.
- 이 때, Flutter의 핵심 알고리즘인 `canUpdate()`를 실행한다.
    - 질문 : newWidget이 oldWidget과 runtimeType이 같고 Key도 같은가?
    - 답변 : 둘 다 Text 타입이고 키도 없으니 같습니다.
- 오케이, 그럼 굳이 RenderObject를 제거할 필요는 없겠다. 속성 값만 업데이트.

### 3. RenderObject Tree

- 기존에 있던 RenderParagraph(텍스트를 그리는 객체)는 파괴되지 않는다.
- 대신 속성이 빨강 → 파랑으로 변경된다.
- 화면에 파란 글씨가 그려진다.

### 만약, Type이 달랐다면?

Text Widget이 Image Widget으로 변경되었다면, Element는 재사용을 못 한다고 판단하고 기존 RenderObject를 파괴하고 메모리에서 지운 뒤, 새로운 Image Widget용 RenderObject를 만든다. (이 작업은 비용이 많이 든다.)

## BuildContext

`build(BuildContext context)` 에서 이 context가 Element 자신이다. 소스 코드를 보면 Element 클래스는 BuildContext를 구현하고 있다.

```dart
abstract class Element extends DiagnosticableTree implements BuildContext { ... }

```

- Widget은 자신의 위치를 모른다. (그냥 설정 정보(종이쪼가리)이기 때문)

  결론 : 효율성과 재사용성(const) 때문에 의도적으로 Widget이 부모/위치 정보를 모르게 설계

  ## 1. 하나의 위젯이 여러 곳에 동시에 존재할 수 있다.

  Flutter에서는 동일한 Widget 인스턴스를 여러 화면에서 동시에 사용할 수 있다.

    ```dart
    // 메모리에 딱 1개만 생성된 const 위젯
    const myIcon = Icon(Icons.star, color: Colors.yellow);
    
    Column(
      children: [
        Container(child: myIcon), // 1번 위치: Container의 자식
        Padding(child: myIcon),   // 2번 위치: Padding의 자식
      ],
    )
    
    ```

  만약, myIcon Widget 클래스 안에 parent라는 변수가 있다고 가정한다면,

    - 1번 위치에 놓일 때 myIcon.parent = Container가 되어야 한다.
    - 2번 위치에 놓일 때 myIcon.parent = Padding이 되어야 한다.

  myIcon 객체는 하나인데, 부모가 여러명일 수는 없다. 따라서 Widget 자체는 Widget 속성에 대한 정보만 갖는다. **위치 정보를 갖게 되면 재사용이 불가능**해진다.

  ## 2. 불변성(Immutablility) 유지와 성능

  Widget은 불변(Immutable)이다. 생성되자마자 모든 값이 고정(final)되어야 한다. 만약 Widget이 부모를 알고 있다면?

    - 부모가 변경되면 자식도 새로 만들어야 함 : 트리 구조상 부모가 바뀌면, 자식 Widget 내부의 parent 변수를 수정해야 한다. 하지만 Widget은 불변이라 수정이 불가능하기 때문에, 결국 자식 Widget을 통째로 새로 복사해서 만들어야 한다.
    - 연쇄 반응 : 자식이 새로 만들어지면, 그 자식들도 새로 만들어야 한다. **결과적으로 트리의 아주 작은 부분만 변경되어도 트리 전체를 새로 만들어야 하는 비효율이 발생**한다.

  Flutter는 Widget은 그냥 설정 정보일 뿐, 위치는 몰라야 하는 원칙 덕분에 부모의 변경 여부와 관계 없이 const Widget을 그대로 재활용 할 수 있다.

  ## 3. 진짜 위치는 Element가 알고 있다.

  위와 같은 이유로 Flutter는 부모/위치 정보를 전담하는 객체를 따로 두었는데, 이게 Element다.

    - Widget : UI를 그려달라고 요청 (부모/위치 모름)
    - Element : 부모 위젯/위치 정보를 알고 있음.

    ```dart
    // 1. Widget (위치 정보 없음)
    abstract class Widget {
      // parent 필드가 아예 없음!
      final Key? key;
      // ...
    }
    
    // 2. Element (위치 정보 있음)
    abstract class Element implements BuildContext {
      Element? _parent; // ★ 진짜 위치는 얘가 안다!
      Widget? _widget;
      // ...
    }
    
    ```

  ## 결론

  Widget Tree는 논리적인 개념으로, 자료구조로 서로 연결된 LinkedNode가 아니다. 자료구조상 진짜 트리는 Element Tree이며, 우리가 BuildContext로 `context.findAncestorWidgetOfExactType` 를 찾을 수 있는 이유 또한 `context`가 곧 `Element`이기 때문이다.

- Element는 Tree에서 자신의 위치를 알고 있다. (부모 자식에 대한 정보를 모두 알고 있다.)

코드를 예로 들면, 그렇기 때문에 `Theme.of(context)`를 호출한다는 것은 Element에게 거슬러 올라가서 가장 가까운 Theme Widget을 찾을 것을 부탁하는 것과 같다.

## 복잡하게 만든 이유

Widget을 바로 업데이트 하면 되지 않을까? → 성능 이슈

- 스마트폰 화면은 초당 60 ~ 120번 갱신된다.
- 매번 실제 UI 객체(RenderObject)를 만들고 부수는 것은 CPU와 메모리에 엄청난 부담이다.
- 단순한 데이터 객체(Widget)을 만들고 부수는 건 매우 빠르다.

Flutter의 전략은 가벼운 Widget을 막 업데이트하고, 무거운 RenderObject를 최대한 재활용하는 것이다. 그 사이에서 재활용 여부를 판한다는 것은 Element가 담당한다.

## 실제 코드와 연결

```dart
Container(
  color: Colors.blue,
  child: Text('Hello'),
)

```

### 1. Widget Tree

- `Container` → `ColoredBox`(내부적으로 생성) → `Text`

### 2. Element Tree

- `StatelessElement`(Container용) → `RenderObjectElement`(ColorBox용) → `StatelessElement`(Text용)

### 3. RenderObject Tree

- `Container` 는 렌더링을 하지 않기 때문에 RenderObject가 없다.
- `RenderColoredBox` (실제로 파란색을 칠함)
- `RenderParagraph` (실제로 글자를 그림)

`Container` 나 `StatelessWidget` 같은 객체는 화면에 그림을 그리지 않는다. 이들은 그저 다른 Widget을 묶어 주는 역할만 하고 실제 그림은 RenderObject를 가진 Widget(RenderObjectWidget)이 그린다.

<details>
    <summary><b>Text Widget이 StatelessWidget이지만 화면에 그릴 수 있는 이유</b></summary>

위에서 Text Widget은 StatelessWidget인데 렌더링이 가능한 RenderParagraph가 되는 이유가 뭘까?

### Text Widget 내부

```dart
    class Text extends StatelessWidget {
      final String data;
      final TextStyle? style;
    
      @override
      Widget build(BuildContext context) {
        // 1. 부모로부터 기본 스타일(DefaultTextStyle)을 가져옵니다.
        final DefaultTextStyle defaultTextStyle = DefaultTextStyle.of(context);
    
        // 2. 내 스타일과 합칩니다.
        TextStyle? effectiveTextStyle = style;
        if (style == null || style!.inherit) {
          effectiveTextStyle = defaultTextStyle.style.merge(style);
        }
    
        // 3. 여기서 진짜 RenderObject인 RichText를 리턴합니다!
        return RichText(
          text: TextSpan(
            text: data,
            style: effectiveTextStyle,
          ),
          // ... 기타 설정들
        );
      }
    }
```

- `Text` 자체는 화면에 픽셀을 그리는 능력이 없는 것이 맞다.
- `Text`의 역할은 스타일을 계산하고 정리해서 `RichText`에 넘겨주는 것.

### Text Widget의 진짜 주인공 RichText & RenderParagraph

`Text` 가 리턴하는 `RichText` 는 `MultiChildRenderObjectWidget`을 상속받는다. 즉, `RichText`가 실제로 `RenderObject`를 갖고 있다. RichText 내부에는 `createRenderObject`라는 메서드가 있어서 여기서 `*RenderObject*`를 생성한다.

```dart
    class RichText extends MultiChildRenderObjectWidget {
      @override
      RenderParagraph createRenderObject(BuildContext context) {
        return RenderParagraph(text, ...); // ★ 진짜 RenderObject 등장!
      }
    
      @override
      void updateRenderObject(BuildContext context, RenderParagraph renderObject) {
        renderObject.text = text; // 내용이 바뀌면 RenderObject에게 알려줌
      }
    }
    
```

- RenderParagraph : 실제 RenderObject
- 이 객체가 캔버스에 글자를 그리고(Paint) 크기를 계산(Layout)한다.

### 계층 구조 정리

`Text(’Hello’)` 를 썼을 때 내부적으로 일어나는 일을 트리로 보면,

1. Widget Tree
   1. `Text` (StatelessWidget)
   2. `RichText` (RenderObjectWidget)
2. Element Tree
   1. `StatelessElement` (Text용)
   2. `MultiChildRenderObjectElement` (RichText용, 여기서 RenderObject 생성.)
3. RenderObject Tree
   1. Text는 RenderObject가 없음
   2. RenderParagraph (RichText가 만든 진짜 RenderObject로 화면에 ‘Hello’를 그림)

### Text라는 포장지가 필요한 이유

- 편의성이다. `RichText`는 스타일 상속을 자동으로 처리하지 않는다. 그래서 `RichText`만 사용하는 경우 기본 글씨가 빨간색에 밑줄이 그어진다.
- Text Widget은 `MaterialApp`이나 `Scaffold`가 내려주는 테마를 찾아서(`DefaultTextStyle.of(context)`) 보기 좋은 스타일을 적용해 준다.

### Text Widget 요약

- Text는 스타일만 계산하고 RichText를 생성하고 리턴한다.
- RichText가 실제 RenderObjectWidget으로 RenderParagraph를 생성한다.
- RenderParagraph가 실제로 화면에 글자 픽셀을 그린다.

> **Text Widget은 대표적인 Flutter의 조합(Composition) 패턴**이다. 복잡한 기능을 가진 RenderObjectWidget을 사용하기 편하게 StatelessWidget으로 감싸서 제공하는 방법이다. Container, Padding, Center 등의 Widget이 이런 패턴을 사용한다.

</details>

<details>
<summary><b>RenderObject와 RenderParagraph의 차이점</b></summary>

- RenderObject : 추상적인 조상 (Base Class)
- RenderParagraph : 구체적인 구현체 (Concrete Implementation)

## 계층 구조

```dart
    RenderObject (최상위 추상 클래스)
        │
        └─ RenderBox (2D 좌표계를 사용하는 렌더링 객체)
              │
              └─ RenderParagraph (글자를 그리는 객체)
    
```

- RenderObject : 모든 렌더링 객체의 뿌리. 기본 규칙만 정의.
- RenderParagraph : 규칙에 따라 픽셀을 그리는 법을 구현. (특히 글자.)

## 비교

| 특징 | RenderObject | RenderParagraph |
      | --- | --- | --- |
| 정체 | 추상 클래스(Abstract class) | 구체 클래스(Concrete class) |
| 역할 | 렌더링 시스템의 뼈대와 규칙 정의 | 텍스트 레이아웃 계산 및 글자 그리기 |
| 하는 일 | 부모가 준 제약(Constraint)를 받고, 실제 자신의 크기를 정하는 프로토콜 정의 | 폰트 크기, 높이, 줄 바꿈 등을 계산 |
| 사용처 | 상속 | RichText Widget 내부에 생성됨 |

## 역할 상세

### RenderObject

What을 정의한다.

- Layout Protocol : `layout()` 함수를 갖고 있다. 부모가 크기 제한(Constraint)를 주면 자기 자신의 사이즈를 계산하는 규칙이다.
- Paint Protocol : `paint()` 함수를 갖고 있다. Canvas에 그리는 규칙을 담당한다.
- Hit Test : 사용자 액션에 반응하는 규칙이다.

### RenderParagraph

RenderObject의 규칙을 따르면서 텍스트 렌더링을 전문적으로 수행한다.

- Layout : 들어온 텍스트(TextSpan)의 길이를 계산하고, 화면 너비에 맞춰 줄바꿈 등 최종적으로 Widget 자기 자신의 width, height를 결정한다.
- Paint : Canvas.drawTest 같은 저수준 API를 사용하여 실제 화면에 글자를 그린다.

</details>

<details>
    <summary><b>RenderObject Tree</b></summary>

RenderObject Tree는 실제로 자료구조상 트리 형태이다. Widget Tree와 반대로 자신의 부모, 자식, 형제들과 서로 연결되어 있다.

## 1. 코드 증거 : parent 포인터

RenderObject의 소스 코드를 따라 올라가면 AbstractNode라는 클래스를 상속받는데, 여기에 명확하게 `parent`변수가 존재한다.

```dart
    // Flutter Framework 소스 코드 (요약)
    abstract class AbstractNode {
      Object? _parent; // ★ 부모를 가리키는 포인터가 존재함!
    
      // 부모를 가져오는 getter
      Object? get parent => _parent;
    
      // 자식이 붙을 때 부모를 설정하는 로직
      void adoptChild(AbstractNode child) {
        child._parent = this;
        // ...
      }
    }
```

- Widget : `parent` 변수 없음. (어디든 쓰일 수 있어야 함)
- RenderObject : `parent` 변수 있음. (화면의 특정 위치에 고정된 실체이기 때문)

## 2. RenderObject가 서로 연결되어야 하는 이유

Flutter의 레이아웃 알고리즘이 작동하려면, 부모 자식이 실시간으로 대화를 해야 하기 때문이다.

- Constraint 전달 (Parent → Child)
  - 부모 RenderObject는 자식에게 크기 제약을 전달한다. (예 : 너는 100px까지 늘어날 수 있다.)
  - Constraint 전달을 위해 부모는 자식을 참조할 수 있어야 한다.
- 크기 보고 (Child → Parent)
  - 자식 RenderObject는 계산을 마치고 자신의 사이즈를 부모에게 보고한다. (예 : 50px만 쓰겠습니다.)
  - 또는, 자식의 내용물이 변경되어서 크기가 변경되면, 부모에게 다시 계산해야 한다고 요청을 보낸다. (`markNeedsLayout`)
  - 이 때 포인터를 타고 올라가서 부모를 꺠운다.
- 만약 서로 참조가 불가능하다면 UI를 그리거나 갱신이 불가능하다.

## 3. 자료구조의 특징 : 이중 연결 리스트

RenderObject의 자식 관리 방식은 일반적인 `List<Child>` 형태가 아닌 Linked List를 사용한다.

```dart
    abstract class RenderObject ... {
      RenderObject? parentData; // 부모가 자식을 관리하기 위한 데이터
    
      // 부모뿐만 아니라 형제끼리도 연결됨
      RenderObject? nextSibling;     // 내 다음 형제
      RenderObject? previousSibling; // 내 이전 형제
    }
```

- 부모 RenderObject는 자식 전체 리스트를 들고 있는게 아니라 First Child만 알고 있는 경우가 많다.
- 둘째 자식을 찾으려면 nextSibling을 타고 이동해야 한다.
- 이렇게 하면 자식을 중간에 추가하거나 삭제할 때 O(1)이라 매우 효율적이다.

</details>

<details>
    <summary><b>WIdget vs Element vs RenderObject</b></summary>

| 트리 종류 | 자료구조상 트리인가? | 부모를 아는가? | 이유 |
| --- | --- | --- | --- |
| Widget | 아니요 | 모름 | 재사용(const)를 위해 위치 정보가 없어야 함. |
| Element | 예 | 예 | 위젯의 위치를 관리하고 트리를 갱신해야 함. (관리자) |
| RenderObject | 예 | 예 | 레이아웃 계산을 위해 부모/자식 간 통신이 필수임. |

## Widget Tree 요약

- **Widget Tree** : 개발자가 만드는 설계도 (빠르고 가볍고 불변)
- **Element Tree** : Widget과 RenderObject를 조율하는 관리자(BuildContext). 변경 사항을 비교해서 재사용을 결정한다.
- **RenderObject Tree** : 실제 픽셀을 그리는 객체. (무겁고 비쌈)
- **Flutter의 전략** : Widget(가볍고 싼)은 자주 변경하되, RenderObject(무겁고 비쌈)는 최대한 재활용 한다.

</details>
