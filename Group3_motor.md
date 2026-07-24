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
|---|---|---|
| `EVT_BUTTON_NONE` | `0` | 안 눌림 |
| `BUTTON_SHORT` | `1` | 짧게 눌림 |
| `BUTTON_LONG` | `2` | 길게 눌림 |
| `BUTTON_LONG` | `2` | 길게 눌림 |


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

| 이름 | 값 | 발생 조건 | 감지 방식 | 감지 위치 |
|---|---|---|---|---|
| `BUTTON_NONE` | 0 | 초기 상태 / 이벤트 소비 완료 |  | | 
| `BUTTON_SHORT` | 1 | 누른 후 3초 미만에 뗌 | 타이머 인터럽트 | |
| `BUTTON_LONG` | 2 | 누른 채로 3초 이상 유지 | 읽은 직후 즉시 리셋 | | 

### 2-2. UART EVENT

| 이름 | 값 | 발생 조건 | 
|---|---:|---|---|
| UART_SPEED | `'0'` ~ `'9'` | UART로 해당 문자를 수신 | 
| UART_FORWARD | `'F'` | UART로 해당 문자를 수신 | 
| UART_REVERSE | `'R'` | UART로 해당 문자를 수신 | 
| UART_STOP | `'S'` | UART로 해당 문자를 수신 | 

## 3. VARIABLE

| 이름 | Type | Scope | 초기값 | 역할 | 변경 시점 |
|---|---|---|---|---|---|
| `btn_event` | enum | | `BUTTON_NONE` |  | |
| `motor_state` | enum | | `STOP` | | |
| `new_motor_state` | int | | `STOP` | | |
| `UART_flag` | int | | `0` | | |
| `speed` | int | | `50` | | |
| `new_speed` | int | | `50` | | |

## 4. METHOD

| 이름 | 입력 | 출력 | 설명 | 호출 시점 |
|---|---|---|---|---|
| | | | | |
