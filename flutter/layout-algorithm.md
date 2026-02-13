# Flutter Layout Algorithm System

Constraints go down. Sizes go up. Parent sets position

## 프로토콜 상세

### 1단계: 제약 조건 전달(Constraints go down. Parent → Child)

부모 위젯이 자식에게 BoxConstraints를 내려보냅니다.

- 부모 위젯 예시 : “자식 위젯은 최소 0부터 최대 300px까지만 쓸 수 있고, 세로는 무조건 50px만 써야 한다.”
- 데이터 : `minWidth: 0, maxWidth: 300, minHeight: 50, maxHeight: 50`

### 2단계: 크기 결정 및 보고(Sizes go up. Child → Parent)

- 자식 위젯 예시 : “폰트 사이즈를 계산하면 가로 50px, 세로 40px을 얻지만, 부모 위젯에서 무조건 50px을 사용하라고 했으니 높이는 무조건 50px로 지정.”
- 데이터 : `Size(40, 50)`

### 3단계: 위치 결정(Parent sets position.)

부모는 자식이 제출한 크기를 받고 자식을 배치한다. 자식은 자신의 크기만 결정할 수 있다. 위치를 결정할 권한은 부모에게 있다.

- 부모 위젯이 자식으로부터 데이터를 받고 위치를 결정한다.
- 데이터 : `child.parentData.offset = Offset(0, 0)`

## 제약 조건(Constraints)의 종류

어떤 제약 조건이냐에 따라 위젯의 성격이 달라진다.

### Tight

- 특징 : `min`과 `max`  값이 같다.
- 의미 : 무조건 이 크기로 지정된다.
- 예시 : `SizedBox(width: 100, height: 100)`

### Loose

- 특징 : `min` 은 0, `max` 가 특정 값.
- 의미 : 이 안에서 자유롭게 크기를 정할 수 있다.
- 예시 : `Center`, `Align` 등
    - 예를 들어, Center 위젯 안에서 Text는 화면을 꽉 채우지 않고 자기 글자 크기만큼 작아질 수 있다.

### Unbounded

- 특징 : `max` 가 `double.infinity`
- 의미 : 크기 제한 없음
- 예시 : `ListView`, `SingleChildScrollView`, `Column` 등
    - Unbounded 제약을 받은 자식 위젯이 똑같이 무한히 커지려고(예: `Expanded`) 할 때 → `RenderFlex overflowed` 에러 발생

## 실전 예제

### 예제 1)

```dart
Container(
	width: 100, 
	height: 100, 
	child: Container(color: red)
)
```

- 바깥 `Container` 는 자식에게 Tight(100x100) 제약을 전달한다.
- 안쪽 `Container` 는 강제로 100x100 사이즈를 갖는다.
- 결과 : 화면에 100x100 빨간 박스가 그려진다.

### 예제 2)

```dart
Center(
	child: Container(
		width: 100, 
		height: 100, 
		color: red,
	)
)
```

- `Center` 는 화면 전체 크기를 부모로부터 받는다.
- `Center`는 자식에게 Loose 제약을 전달한다.
- 자식 `Container` 는 원하는 사이즈를 가질 수 있다.
- 결과 : `Center` 에 의해 정중앙에 위치한 100x100 빨간 박스가 그려진다.

## 렌더링 파이프라인의 핵심: `layout()` 메서드

`RenderObject`의 `layout()` 메서드

```dart
void layout(Constraints constraints, {bool parentUsesSize = false}) {
  // 1. 부모가 준 제약(constraints)을 저장
  this.constraints = constraints;

  // 2. 내 자식들이 있다면 자식들의 layout()을 재귀적으로 호출
  //    (Constraints go down)
  if (child != null) {
    child.layout(childConstraints, parentUsesSize: true);
  }

  // 3. 자식들의 크기를 바탕으로 내 크기(Size) 결정
  //    (Sizes go up)
  this.size = calculateMySize();
  
  // 4. (옵션) 부모가 내 크기를 궁금해한다면 보고함
}
```

- Flutter layout 성능 최적화 : `RelayoutBoundary`
    - `parentUsesSize = false` 로 `layout()` 을 호출하는 경우 자식 위젯은 크기를 자유롭게 정할 수 있다.
    - 이런 경우 자식의 크기가 변하더라도 부모 위젯은 레이아웃을 다시 계산할 필요가 없다.

## 요약

- Constraints down : 부모가 자식 위젯에게 제약 조건을 전달한다.
- Sized up : 자식 위젯은 제약 조건 안에서 크기를 정하고 부모에게 전달한다.
- Parent sets position : 부모가 자식 위젯의 위치를 지정한다. 자식은 크기만 정할뿐이다.