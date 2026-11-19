# F411_LAB

STM32F411 + 초음파 센서(HC-SR04) + 온습도 센서(DHT11) 실습 프로젝트입니다.

## 1. 보드 구성

![NUCLEO-F411RE 보드](images/board_nucleo_f411re.jpg)

- 보드: NUCLEO-F411RE, USB(ST-LINK)로 PC에 연결
- 센서는 점퍼선으로 연결, 초음파 핀은 `TRIG=PA1`, `ECHO=PA0`
- 초록색 점퍼선은 오실로스코프 프로브로 연결

## 2. PuTTY · 오실로스코프

### PuTTY (초음파)

![PuTTY UART 로그](images/putty_uart_log.png)

- `D,시간(ms),거리(mm),valid,errors`: 5회 중앙값 거리, 1초마다 출력
- `C,ticks`: 마지막 Echo 폭 (1 tick = 1 µs)
- `C × 0.1715 ≈ D`로 두 값이 서로 맞음 (예: `C,10099` ≈ 1732 mm, `D` 1726 mm)
- `C`가 튀어도(`3365`) `D`는 유지되어 중앙값 필터가 동작함

### PuTTY (초음파 + 온습도)

![PuTTY 초음파 + 온습도 로그](images/putty_uart_ultrasonic_dht.png)

- `TH,시간(ms),221,300,1,0` / `ENV,+22.1C,30.0%RH`: 2초마다 온습도 출력 (22.1 °C, 30.0 %RH)
- `S,3449`: 온도로 보정한 음속(×10, 344.9 m/s)으로 보임
- 보정 음속으로 계산하면 `10491 × 3449 / 20000 ≈ 1809 mm`로 `D`와 일치 (기준 음속 3430이면 약 1799 mm)
- 거리 1809 mm, `C` 10489 ~ 10494로 안정, `errors` 0

> `TH`, `ENV`, `S` 출력은 아직 저장소 코드에 없어서 필드 의미는 로그 기준 추정입니다.

### 오실로스코프

![오실로스코프 Echo 펄스](images/oscilloscope_echo_pulse.png)

- Echo 펄스 폭 **2.1 ms**, 진폭 3.36 V → 약 360 mm

## 3. SWV 데이터 트레이스

### ① Echo 폭 · 엣지 카운트

![SWV 데이터 트레이스 타임라인](images/swv_trace_graph.png)

- 트리거 200 ms 주기, Echo High 약 10 ms (약 1.7 m)
- 엣지 카운트는 측정마다 +2로 누락 없음

### ② Trig / Echo 엣지

![Trig / Echo 엣지 트레이스](images/swv_trace_trig_echo.png)

- Trig → Echo Rising → 약 10 ms 후 Falling 순서가 매 주기 반복

### ③ 초음파 + 온습도(DHT11)

![초음파 + DHT 트레이스](images/swv_trace_ultrasonic_dht.png)

- DHT 라인이 4.00~4.02 s에 약 20 ms Low (DHT11 시작 신호로 보임)
- 직후에 초음파 Trig가 이어짐

> `g_trace_trig_edge`, `g_trace_dht_edge`는 아직 저장소 코드에 없어서 그래프 기준으로만 정리했습니다.

### SWV 설정 (①번 기준)

![Serial Wire Viewer 설정](images/swv_trace_settings.png)

- 코어 84 MHz, SWO 2000 kHz, 타임스탬프 활성화
- Comparator 0~3: `g_trace_echo_edge`, `g_trace_echo_ticks`, `g_trace_echo_state`, `g_trace_edge_count` (Write, Data Value + PC)
