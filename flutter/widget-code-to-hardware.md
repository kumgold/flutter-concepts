# Flutter Rendering Pipeline: 4단계 여정

Flutter가 화면을 그리는 과정은 크게 두 개의 스레드(Thread)가 주도합니다.

1. **UI Thread (CPU)**: Dart 코드가 실행되는 곳.
2. **Raster Thread (CPU -> GPU)**: GPU와 통신하며 명령을 전달하는 곳.

---

## 1단계: Build & Layout (UI Thread / CPU)

**"설계도 그리기"**

개발자가 작성한 Dart 코드가 실행되는 단계입니다.

1. **V-Sync 신호 수신**: 하드웨어(디스플레이)가 "나 그릴 준비 됐어!"라고 신호를 보내면, Flutter 엔진이 SchedulerBinding을 통해 Dart 코드를 깨웁니다.
2. **Build**: Widget Tree를 순회하며 변경사항을 파악하고, Element Tree를 갱신합니다.
3. **Layout**: RenderObject들이 서로의 크기(Size)와 위치(Offset)를 계산합니다. (Constraint 전달 -> Size 결정)
4. **Paint (Recording)**:
    - **중요**: 여기서 실제로 픽셀을 찍는 게 아닙니다.
    - Canvas.drawRect, Canvas.drawText 같은 메서드를 호출하면, 이는 "PictureLayer"라는 객체에 **명령어(Command)** 형태로 기록됩니다.
    - 결과물: **Layer Tree** (그리기 명령어가 담긴 여러 층의 레이어 뭉치)

> 💻 하드웨어 관점: 여기까지는 CPU가 연산을 수행하며 시스템 메모리(RAM)에 가상의 트리 구조(Layer Tree)를 만듭니다.
>

---

## 2단계: Compositing & Submit (UI Thread -> Raster Thread)

**"설계도 제출 및 합성"**

Dart 영역에서 만든 Layer Tree를 C++ 영역(Flutter Engine)으로 넘기는 과정입니다.

1. **SceneBuilder**: Dart의 Layer Tree를 엔진이 이해할 수 있는 Scene 객체로 변환합니다.
2. **Raster Thread로 전송**: 완성된 Scene을 Raster Thread로 보냅니다. 이 순간부터 UI Thread는 다음 프레임을 준비하러 떠납니다. (비동기 파이프라인)

---

## 3단계: Rasterization (Raster Thread / CPU -> GPU)

**"명령어 번역 (Skia / Impeller)"**

여기가 **가장 중요한 병목 지점**이자 하드웨어 가속의 핵심입니다. Raster Thread는 GPU가 이해할 수 있는 언어로 번역을 담당합니다.

### 3-1. Skia (구형 엔진) 또는 Impeller (신형 엔진)

이전에는 **Skia**, 최근(iOS 등)에는 **Impeller**라는 그래픽 엔진을 사용합니다.

1. **Layer 합성**: 여러 겹의 Layer를 순서대로 정리합니다.
2. **GPU 명령어 변환**:
    - Dart가 보낸 "원 그려줘"라는 고수준 명령을 GPU가 이해하는 저수준 그래픽 API (**OpenGL, Vulkan, Metal**) 호출로 바꿉니다.
    - 예: drawRect -> "삼각형 정점(Vertices) 2개를 배치하고, 셰이더(Shader)를 입혀라."

### 3-2. Shader (셰이더)

- **Shader란?**: GPU 코어 하나하나가 실행하는 **작은 프로그램**입니다. "이 픽셀은 무슨 색이어야 해?"를 계산합니다.
- **Jank의 원인 (Skia Shader Compilation)**:
    - Skia는 앱 실행 중에 새로운 그리기 명령이 들어오면, **실시간으로 셰이더를 컴파일**했습니다.
    - 이 컴파일 시간이 길어지면 프레임을 놓쳐서 버벅임(Jank)이 발생했습니다.
- **Impeller의 해결책**:
    - 앱 빌드 타임에 모든 셰이더를 **미리 컴파일(AOT)** 해둡니다. 그래서 런타임에 버벅임이 없습니다.

---

## 4단계: Hardware Execution (GPU -> Display)

**"물리적 발광"**

이제 데이터는 CPU를 떠나 그래픽 카드(GPU)와 디스플레이로 넘어갔습니다.

1. **GPU Processing (병렬 처리)**:
    - Raster Thread가 보낸 명령을 받아 수천 개의 GPU 코어가 동시에 계산합니다.
    - **Vertex Processing**: 도형의 꼭짓점 좌표를 계산.
    - **Fragment Shading**: 픽셀 하나하나의 색상을 채움.
2. **Frame Buffer**:
    - GPU가 계산을 끝낸 픽셀 데이터들은 비디오 메모리(VRAM)의 **Frame Buffer**라는 공간에 저장됩니다. (완성된 그림 한 장)
3. **Display Controller**:
    - 디스플레이 장치에 붙어있는 칩셋입니다.
    - 주기적(60Hz, 120Hz)으로 Frame Buffer를 읽어서(Scan out), 패널의 소자(LED/OLED)에 전압을 가해 **빛을 냅니다.**

---

# 🔎 심화: 코드가 하드웨어에 미치는 영향 (성능 저하의 원인)

이제 왜 특정 코드가 성능을 떨어뜨리는지 하드웨어 관점에서 설명할 수 있습니다.

### 1. saveLayer()와 Offscreen Buffer (치명적)

Flutter에서 Opacity, ShaderMask, ClipPath 등을 남발하면 성능이 떨어지는 이유입니다.

- **상황**: saveLayer() 명령이 떨어짐.
- **하드웨어 동작**:
    1. GPU는 그리던 작업을 멈춤 (Context Switch).
    2. VRAM에 새로운 임시 도화지(Offscreen Buffer)를 할당함.
    3. 그곳에 그림을 그림.
    4. 다시 원래 도화지(Frame Buffer)로 돌아옴.
    5. 임시 도화지의 내용을 픽셀 단위로 복사해서 붙여넣음.
- **결과**: GPU 메모리 대역폭 낭비 & 렌더링 파이프라인 중단.
- **해결**: FadeTransition(GPU Alpha) 등을 사용하여 saveLayer 호출을 피함.

### 2. Image Decoding (IO -> CPU -> GPU)

고해상도 이미지를 로딩할 때 렉이 걸리는 이유입니다.

- **과정**:
    1. 디스크에서 압축된 이미지(JPG/PNG)를 읽음 (IO Thread).
    2. CPU가 압축을 풀어서 비트맵(Bitmap)으로 만듦.
    3. 이 거대한 비트맵 데이터를 CPU 메모리에서 **GPU 메모리로 전송(Upload)** 해야 함.
- **병목**: CPU와 GPU 사이의 **버스(Bus) 대역폭**은 한계가 있습니다. 너무 큰 이미지를 보내면 전송하느라 16.6ms를 넘겨버립니다.
- **해결**: cacheWidth, cacheHeight를 사용하여 딱 필요한 크기로 리사이징 후 GPU에 전송해야 합니다.

---

# Code to Hardware 흐름도

```
[1. Dart Code (CPU)]
   ↓ (setState)
   Widget Tree 생성
   ↓ (Layout)
   RenderObject Tree (크기/위치 계산)
   ↓ (Paint)
   DisplayList (그리기 명령어 기록) - "여기 빨간 원 그려"

[2. Flutter Engine (CPU - Raster Thread)]
   ↓ (Layer Tree 합성)
   SceneBuilder
   ↓ (Skia/Impeller 변환)
   GPU Command (OpenGL/Metal) - "삼각형 정점 좌표 3개, 셰이더 ID 5번 실행"

[3. GPU (Hardware)]
   ↓ (Vertex Shader -> Fragment Shader)
   픽셀 색상 계산 (병렬 처리)
   ↓
   Frame Buffer (VRAM에 완성된 그림 저장)

[4. Display (Hardware)]
   ↓ (V-Sync)
   Frame Buffer 읽기 (Scan out)
   ↓
   LCD/OLED 패널 발광 (사용자 눈에 보임)

```

# 요약: Flutter 최적화 3계명

안드로이드 View 시스템과 원리는 똑같습니다.

1. **CPU에게 일을 덜 시켜라 (Rebuild 최소화)**
    - Compose: derivedStateOf, Modifier 람다 사용.
    - Flutter: AnimatedBuilder의 child 파라미터 활용, const 생성자 활용.
2. **GPU에게 일을 넘겨라 (Paint 최소화)**
    - Compose: graphicsLayer.
    - Flutter: RepaintBoundary.
3. **레이어 합성을 조심해라**
    - Compose: 과도한 clip, alpha 주의.
    - Flutter: Opacity 대신 FadeTransition, ClipRRect 남발 자제.

<details>
    <summary><b>Flutter의 하드웨어 가속</b></summary>

- 안드로이드 View의 **RenderNode (DisplayList 캐싱)** 개념이 Flutter에서는 **RepaintBoundary (Layer)** 로 구현됩니다.
- **일반적인 경우**: 부모 위젯을 다시 그리면 자식 위젯도 페인트 명령(draw...)을 다시 실행합니다.
- **RepaintBoundary 사용**:
  - 이 위젯 아래의 그리기 결과는 별도의 **텍스처(이미지)** 처럼 GPU 메모리에 저장됩니다.
  - 위치가 이동하거나 회전할 때, 다시 그리기(Paint)를 수행하지 않고 **"저장된 이미지를 이동/회전만 시켜"**라고 GPU에게 명령합니다.
  - Compose의 graphicsLayer와 완벽하게 동일한 역할입니다.
 
</details>

<details>
    <summary><b>개념을 적용한 예제 코드</b></summary>

### 1. RepaintBoundary 사용하기

(Compose의 graphicsLayer와 대응)

복잡한 이미지를 계속 회전시키는 애니메이션입니다.

### ❌ [Bad] 매 프레임 다시 페인트 (CPU 과부하)

    ```dart
    class BadRotation extends StatelessWidget {
      @override
      Widget build(BuildContext context) {
        // AnimationController에 의해 build가 매번 호출됨
        return Transform.rotate(
          angle: animationValue,
          child: ComplexWidget(), // 엄청 복잡한 그림 (그리는 데 5ms 소요)
        );
      }
    }
    
    ```

**문제점:**

    - 회전 각도가 바뀔 때마다 ComplexWidget의 paint() 메서드가 매번 실행됩니다.
    - CPU는 매번 "복잡한 그림 그리기 명령어"를 새로 생성해서 GPU에 보냅니다.

### ✅ [Good] 레이어 분리 (GPU 가속)

    ```dart
    class GoodRotation extends StatelessWidget {
      @override
      Widget build(BuildContext context) {
        return Transform.rotate(
          angle: animationValue,
          child: RepaintBoundary( // ★ 핵심: 여기서 레이어를 끊어줌
            child: ComplexWidget(),
          ),
        );
      }
    }
    
    ```

**이유:**

    - RepaintBoundary를 쓰면 ComplexWidget은 딱 한 번만 그려져서 GPU 메모리에 저장됩니다.
    - 회전할 때 CPU는 ComplexWidget의 paint()를 호출하지 않습니다.
    - 대신 GPU에게 "아까 저장한 그 텍스처를 1도만 돌려"라고 명령합니다. 성능이 비약적으로 상승합니다.

    ---

### 2. AnimatedBuilder & child 캐싱

(Compose의 Modifier.offset {} 람다와 유사한 효과)

애니메이션이 돌 때 위젯 트리 전체를 재생성(Rebuild)하지 않도록 하는 기법입니다.

### ❌ [Bad] 전체 Rebuild

    ```dart
    class BadAnimation extends StatefulWidget {
      @override
      Widget build(BuildContext context) {
        // _controller가 변할 때마다 setState() 호출 -> BadAnimation 전체가 다시 빌드됨
        return Container(
          color: Colors.blue,
          transform: Matrix4.translationValues(0, _controller.value, 0),
          child: const VeryExpensiveWidget(), // 얘도 불필요하게 다시 검토됨
        );
      }
    }
    
    ```

### ✅ [Good] 변하는 부분만 쏙 빼내기

    ```dart
    class GoodAnimation extends StatelessWidget {
      @override
      Widget build(BuildContext context) {
        return AnimatedBuilder(
          animation: _controller,
          // 1. 변하지 않는 부분(ExpensiveWidget)은 한 번만 생성해서 child로 전달
          child: const VeryExpensiveWidget(),
          builder: (context, child) {
            // 2. 여기만 매 프레임 실행됨
            return Transform.translate(
              offset: Offset(0, _controller.value),
              child: child, // 3. 캐싱된 child 재사용 (Rebuild 안 함)
            );
          },
        );
      }
    }
    
    ```

**이유:**

    - Compose에서 layout 단계에서만 오프셋을 계산하듯, Flutter에서는 AnimatedBuilder의 builder 내부만 실행시킵니다.
    - 무거운 VeryExpensiveWidget은 다시 빌드되지 않고(Reused), 위치만 이동합니다. CPU 사용량을 최소화합니다.

    ---

### 3. Opacity vs FadeTransition

(투명도 처리의 차이)

### ❌ [Bad] Opacity 위젯

    ```dart
    Opacity(
      opacity: animationValue,
      child: MyWidget(),
    )
    
    ```

**문제점:**

    - Opacity 위젯은 내부적으로 saveLayer()라는 비용이 큰 명령을 사용할 수 있습니다. (Off-screen 버퍼를 만들고 합성하는 과정이 CPU/GPU에 부담)

### ✅ [Good] FadeTransition (GPU Alpha)

    ```dart
    FadeTransition(
      opacity: animationController,
      child: MyWidget(),
    )
    
    ```

**이유:**

    - FadeTransition은 내부적으로 GPU의 **Alpha Blending** 기능을 최적화해서 사용합니다.
    - Compose의 graphicsLayer { alpha = ... }와 같이 GPU 레벨에서 처리되어 훨씬 부드럽습니다.
</details>