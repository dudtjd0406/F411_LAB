# F411_LAB

STM32F411 + 초음파 센서(HC-SR04 계열) 실습 프로젝트입니다.

## SWV 데이터 트레이스 관찰 (초음파 Echo)

`Core/Src/ultrasonic.c`의 동작을 **SWV(Serial Wire Viewer) 데이터 트레이스**로 관찰한 기록입니다.
CPU를 멈추지 않고 전역 변수 4개의 쓰기(Write) 이벤트를 타임스탬프와 함께 캡처했습니다.

### 1. 관찰 결과

![SWV 데이터 트레이스 타임라인](images/swv_trace_graph.png)

관찰 구간은 약 83.35 s ~ 84.3 s (약 1 s)입니다.

| 관찰 항목 | 그래프에서 확인한 내용 | 코드와의 대응 |
|---|---|---|
| `g_trace_echo_state` (파란선) | 83.4, 83.6, 83.8, 84.0, 84.2 s 부근에서 주기적으로 튀어 오름 | `US_SAMPLE_PERIOD_MS = 200` → 200 ms마다 트리거 |
| `g_trace_echo_edge` (주황선) | 트리거 직후 올라갔다가 약 10 ms 뒤 0으로 내려옴 | Echo 핀 Rising 때 1, Falling 때 0 |
| `g_trace_echo_ticks` (빨간선) | 10,000 tick 근처에서 거의 일정 | Echo 펄스 폭(1 tick = 1 µs) → 약 10 ms |
| `g_trace_edge_count` (초록선) | 측정할 때마다 계단식으로 증가 | Rising/Falling 각각 +1 → 측정 1회당 +2 |

정리하면 다음과 같습니다.

- 트리거 간격이 200 ms로 일정하게 찍혀서, 폴링 방식 `Ultrasonic_Task()`의 샘플 주기가 설계대로 동작함을 확인했습니다.
- Echo 폭이 약 10,000 µs로 일정하면 거리는 대략 1.7 m 수준입니다 (`10000 × 3430 / 20000 ≈ 1715 mm`). 눈금을 읽은 추정값이라 정확한 값은 `Ultrasonic_Result`로 확인해야 합니다.
- `g_trace_edge_count`는 Echo 엣지가 빠짐없이 들어오면 측정마다 정확히 2씩 늘어납니다. 그래프에서 계단 간격이 일정해서 엣지 누락은 보이지 않습니다.
- 파란선(state)은 평소 바닥(0 상태)에 있다가 트리거 시점에 2로 올라가는 것으로 보입니다. 축 범위가 1~2라서 0은 바닥에 붙어 보입니다.

### 2. 관찰한 변수

| Comparator | 변수 | 타입 | 값의 의미 | 갱신 위치 |
|---|---|---|---|---|
| 0 | `g_trace_echo_edge` | `uint8_t` | 1 = Echo High 구간, 0 = Low / 타임아웃 | `Ultrasonic_OnEchoEdge()`, 타임아웃 처리 |
| 1 | `g_trace_echo_ticks` | `uint16_t` | 마지막 Echo 펄스 폭 (tick) | Falling 엣지 |
| 2 | `g_trace_echo_state` | `uint8_t` | 0 = 대기, 1 = Echo High, 2 = 트리거 후 Echo 대기 | `Ultrasonic_Task()`, `Ultrasonic_OnEchoEdge()` |
| 3 | `g_trace_edge_count` | `uint32_t` | Echo 엣지 누적 횟수 | Rising / Falling 엣지 |

`state` 변화 순서는 `0 → 2 (트리거 송출) → 1 (Rising 엣지) → 0 (Falling 엣지)`입니다.
Echo가 35 ms(`US_ECHO_TIMEOUT_MS`) 안에 돌아오지 않으면 `edge`와 `state`가 0으로 초기화되고 에러 카운트가 올라갑니다.

### 3. SWV 설정

![Serial Wire Viewer 설정](images/swv_trace_settings.png)

| 항목 | 값 |
|---|---|
| 코어 클럭 | 84 MHz |
| SWO 클럭 | 2000 kHz (클럭 프리스케일러 42) |
| 타임스탬프 | 활성화, 프리스케일러 1 |
| PC 샘플링 | 비활성화 |
| 트레이스 이벤트 (CPI / SLEEP / FOLD / EXC / LSU / EXETRC) | 모두 비활성화 |
| ITM 포트 | 모두 비활성화 |
| Comparator 0~3 | 모두 활성화, 액세스 `Write`, 생성 `Data Value + PC` |

DWT 비교기 4개를 모두 사용했습니다. 변수를 더 관찰하려면 기존 항목 중 하나를 비워야 합니다.

### 4. 재현 방법

1. 프로젝트를 Debug 모드로 빌드하고 디버그 세션을 시작합니다.
2. Debug 설정의 **Serial Wire Viewer** 옵션에서 위 표대로 Comparator 0~3 변수를 등록합니다.
3. SWV 데이터 트레이스 타임라인 그래프 뷰를 열고 변수 4개를 추가합니다.
4. Resume 후 SWV 트레이스를 시작합니다 (Start Trace).
5. 센서를 향해 물체를 움직이면 `g_trace_echo_ticks`가 거리에 비례해 변하는 것을 볼 수 있습니다.

### 5. 참고

- 관련 코드: `Core/Src/ultrasonic.c`, `Core/Inc/ultrasonic.h`
- 타이머는 TIM1을 1 tick = 1 µs로 사용합니다 (`US_TIMER_TICKS_PER_US = 1`).
- 거리 계산은 `width_us × 음속(×10) / 20000` (mm), 5회 중앙값 필터를 거쳐 1초마다 게시됩니다.
