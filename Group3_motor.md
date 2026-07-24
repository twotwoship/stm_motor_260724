# [3조] 모터 제어 상위설계서

신유지, 이호윤, 이양배, 한소영

## 요구 사항
* 모터는 버튼과 UART 명령 두 가지 방식으로 제어할 수 있어야 한다.
* 시스템 시작 시 모터는 정지 상태여야 한다.
* 메인 루프는 UART 처리, 버튼 처리, 모터 상태 처리를 반복 수행한다.
* 방향 전환시 모터 부하 방지를 위해 전환 간 대기시간을 둔다.
* UART PWM duty rate 50 ~ 95% 이다.
* 위 요구사항에 명시되지 않은 세부 구현 방식은 자유롭게 설계한다.

## 1. STATE

### 1-1. FSM

**다이어그램 1. 메인 동작 흐름**

```mermaid
flowchart LR
    A[전원 인가] --> B[인터럽트 이벤트 감지]
    B --> C[이벤트 기반 상태 업데이트]
    C --> D[상태 기반 모터 제어]
    D --> B
```

**다이어그램 2. 모터 상태 전이**

### 1-2. MOTOR STATE

| 이름 | 값 | 설명 |
|---|---|---|
| `STOP` | `0` | 모터 정지 |
| `CW` | `1` | 모터 정회전 |
| `CCW` | `2` | 모터 역회전 |

### 1-3. MOTOR SPEED

| 이름 | 값 | 설명 |
|---|---|---|
| `0` | `50` | PWM Duty 50% 설정 |
| `1` | `55` | PWM Duty 55% 설정 |
| `2` | `60` | PWM Duty 60% 설정 |
| `3` | `65` | PWM Duty 65% 설정 |
| `4` | `70` | PWM Duty 70% 설정 |
| `5` | `75` | PWM Duty 75% 설정 |
| `6` | `80` | PWM Duty 80% 설정 |
| `7` | `85` | PWM Duty 85% 설정 |
| `8` | `90` | PWM Duty 90% 설정 |
| `9` | `95` | PWM Duty 95% 설정 |

## 2. EVENT (BUTTON/UART)

| 이름 | 값 | 설명 |
|---|:---:|---|
| `EVT_NONE` | 0 | 이벤트 없음 / 이벤트 소비 완료 |
| `EVT_BTN_SHORT` | 1 | 버튼 짧게 눌림 (버튼 눌린 시점부터 뗀 시간까지 측정. 눌린 시간 < 3000ms) |
| `EVT_BTN_LONG` | 2 | 버튼 길게 눌림 (버튼이 눌린 채로 경과 시간 >= 3000ms) |
| `EVT_UART` | 3 | UART로 문자 수신 (수신 문자는 ISR에서 Uart_Data_In에 저장) |

## 3. GLOBAL VARIABLE

| 이름 | Type | Scope | 초기값 | 역할 | 변경 시점 |
|---|---|---|---|---|---|
| `evt` | `event_t` | 전역 | `EVT_NONE` | 현재 이벤트 저장 | 인터럽트 발생시`check_event()` 에서 갱신|
| `motor_state` | `motor_state_t` | 전역 | `MOTOR_STOP` | 현재 모터 상태 저장 | `drive_motor()`에서 모터 상태 변경 시 갱신 |
| `new_motor_state` | `motor_state_t` | 전역 | `MOTOR_STOP` | 새로운 모터 회전 저장 | `update_motor_state()`에서 이벤트 처리 시 갱신 |
| `motor_speed` | uint8_t | 전역 | `5` | 현재 PWM Duty 레벨 (0~9) | `drive_motor()`에서 모터 상태 변경 시 갱신 |
| `new_motor_speed` | uint8_t | 전역 | `5` | 새로운 PWM Duty 레벨 (0~9) | `update_motor_state()`에서 이벤트 처리 시 갱신 |
| `btn_flag` | volatile uint8_t | 전역 | `0` | 버튼 인터럽트 (falling edge/rising edge) 발생 여부 | Button_ISR에서 1로 세팅 → `check_event()`에서 읽은 직후 0으로 리셋 |
| `uart_flag` | volatile uint8_t | 전역 | `0` | UART 인터럽트 발생 여부 | `UART_ISR`에서 1로 세팅 → `check_event()`에서 읽은 직후 0으로 리셋 |
| `uart_data` | volatile uint8_t | 전역 | `0` | UART로 수신된 문자 | `UART_ISR`에서 수신 시 갱신 |

## 4. METHOD

| 이름 | 입력 | 출력 | 설명 | 호출 시점 |
|---|---|---|---|---|
| `check_event()` | `btn_flag`, `uart_flag` (전역 참조) | `evt` 갱신 | `btn_flag`가 세워져 있으면 핀 상태로 falling/rising을 판별해 `EVT_BTN_SHORT` 여부를 계산하고, 눌린 채로 3초가 지나면 `EVT_BTN_LONG`을 생성. `uart_flag`가 세워져 있으면 `EVT_UART`를 생성. | main 루프 |
| `update_motor_state()` | `evt`, `motor_state`, `motor_speed` (전역 참조) | `new_motor_state`, `new_motor_speed` 갱신 | `evt`가 `EVT_BTN_SHORT`면 CW/CCW를 토글, `EVT_BTN_LONG`이면 `MOTOR_STOP`으로 전환. `EVT_UART`면 `uart_data`가 숫자인지 'f', 'r', 's'인지에 따라 `new_motor_speed` 또는 `new_motor_state`를 갱신. | main 루프, `evt != EVT_NONE`일 때만 호출 |
| `drive_motor()` | `new_motor_state`, `new_motor_speed` (전역 참조) | GPIO 방향 설정, PWM Duty 반영, `motor_state`, `motor_speed` 갱신 | `new_motor_state`와 `motor_state`, `new_motor_speed`와 `motor_speed`를 비교하여 다를 때만 GPIO 방향을 설정하고 PWM Duty를 반영 | main 루프 |
