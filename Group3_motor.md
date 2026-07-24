<<<<<<< HEAD
# [3조] 모터 제어 상위설계서

신유지, 이호윤, 이양배, 한소영

## 1. STATE

### 1-1. FSM

**다이어그램 1. 메인 동작 흐름**

```mermaid
flowchart LR
    A[전원 인가] --> B[이벤트 감지 함수]
    B --> C[상태 처리 함수]
    C --> D[모터 제어 함수]
    D --> B
```

**다이어그램 2. 모터 상태 전이**

*** 추후 추가 ***

### 1-2. MOTOR STATE

| 이름 | 값 | 설명 |
|---|---|---|
| `STOP` | `0` | 모터 정지 |
| `CW` | `1` | 모터 정회전 |
| `CCW` | `2` | 모터 역회전 |


### 1-3. EVENT STATE

| 이름 | 값 | 설명 |
|---|:---:|---|
| `EVT_NONE` | 0 | 이벤트 없음 / 이벤트 소비 완료 |
| `EVT_BTN_SHORT` | 1 | 버튼 짧게 눌림 (3초 미만) |
| `EVT_BTN_LONG` | 2 | 버튼 길게 눌림 (3초 이상) |
| `EVT_UART_RX` | 3 | UART로 1바이트 수신 (payload에 수신 문자 포함) |


### 1-4. SPEED STATE

| 이름 | 값 | 설명 |
|---|---|---|
| `0` | `20` | PWM Duty 20% 설정 |
| `1` | `50` | PWM Duty 50% 설정 |
| `2` | `55` | PWM Duty 55% 설정 |
| `3` | `60` | PWM Duty 60% 설정 |
| `4` | `65` | PWM Duty 65% 설정 |
| `5` | `70` | PWM Duty 70% 설정 |
| `6` | `75` | PWM Duty 75% 설정 |
| `7` | `80` | PWM Duty 80% 설정 |
| `8` | `85` | PWM Duty 85% 설정 |
| `9` | `90` | PWM Duty 90% 설정 |

## 2. EVENT

### 2-1. BUTTON EVENT

| 이름 | 값 | 설명 | 발생 조건 |
|---|---|---|---|
| `BUTTON_NONE` | 0 | 이벤트 없음 | 아래 세 조건 모두 미충족 |  
| `BUTTON_SHORT` | 1 | 버튼 짧게 눌림 | 버튼 눌린 시점, 눌린 시간 < 3000ms | 
| `BUTTON_LONG` | 2 | 버튼 길게 눌림 | 버튼이 눌린 채로 경과 시간 >= 3000ms |  

### 2-2. UART EVENT

| 이름 | 값 | 발생 조건 | 
|---|---|---|
| UART_SPEED | `'0'` ~ `'9'` | UART로 해당 문자를 수신 | 
| UART_FORWARD | `'F'` | UART로 해당 문자를 수신 | 
| UART_REVERSE | `'R'` | UART로 해당 문자를 수신 | 
| UART_STOP | `'S'` | UART로 해당 문자를 수신 | 

## 3. GLOBAL VARIABLE

| 이름 | Type | Scope | 초기값 | 역할 | 변경 시점 |
|---|---|---|---|---|---|
| `event` | `event_t` | 전역 | `BUTTON_NONE` | 현재 이벤트 저장 | |
| `motor_state` | `motor_state_t` | 전역 | `MOTOR_STOP` | 현재 모터 회전 방향 / 정지 상태 | `update_motor_state()`에서 `EVT_BTN_SHORT` / `EVT_BTN_LONG` / `EVT_UART_RX` 처리 시 직접 갱신 |
| `new_motor_state` | int | | `STOP` | | |
| `uart_flag` | int | | `0` | | |
| `motor_speed` | uint8_t | 전역 | `5` | 현재 PWM Duty 레벨 (0~9) | `update_motor_state()`에서 `EVT_UART_RX`의 숫자 payload 처리 시 갱신 |
| `btn_flag` | volatile uint8_t | 전역 | 0 | 버튼 edge(falling/rising) 발생 여부 | EXTI_Button_ISR에서 1로 세팅 → check_event()에서 읽은 직후 0으로 리셋 |
| `uart_flag` | volatile uint8_t | 전역 | `0` | UART 1바이트 수신 여부 | `UART_ISR`에서 1로 세팅 → `check_event()`에서 읽은 직후 0으로 리셋 |
| `uart_Data_In` | volatile uint8_t | 전역 | `0` | UART로 수신된 문자 | `UART_ISR`에서 수신 시 갱신 |

| `new_speed` | int | | `50` | | |
| `new_speed` | int | | `50` | | |









## 4. METHOD

| 이름 | 입력 | 출력 | 설명 | 호출 시점 |
|---|---|---|---|---|
| | | | | |

