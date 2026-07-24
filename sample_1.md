# [3조] 모터 제어 상위설계서

신유지, 이호윤, 이양배, 한소영

## 1. 요구 사항

* 모터는 버튼과 UART 명령 두 가지 방식으로 제어
* 전원 인가 시 모터는 정지 상태
* 메인 루프는 UART 처리, 버튼 처리, 모터 상태 처리를 반복 수행
* 방향 전환 시 모터 부하 방지를 위해 전환 간 대기시간 필요
* UART로 PWM Duty Rate를 50%에서 95%까지 설정할 수 있어야 한다.
* 위 요구사항에 명시되지 않은 세부 구현 방식은 자유롭게 설계한다.

## 2. 상태 및 이벤트

### 2-1. 모터 상태

| 이름 | 값 | PA0 | PA1 | 설명 |
|---|:---:|---|---|---|
| `MOTOR_STOP` | 0 | GPIO LOW | GPIO LOW | 모터 정지 |
| `MOTOR_CW` | 1 | GPIO LOW | TIM5_CH2 PWM | 소프트웨어 기준 정방향 회전 |
| `MOTOR_CCW` | 2 | TIM5_CH1 PWM | GPIO LOW | 소프트웨어 기준 역방향 회전 |

### 3-2. 버튼/UART 이벤트

| 이름 | 값 | 발생 조건 | 처리 결과 |
|---|:---:|---|---|
| `BTN_NONE` | 0 | 초기 상태 또는 이벤트 처리 완료 | 모터 상태 변경 없음 |
| `BTN_SHORT_PRESSED` | 1 | 버튼을 누른 후 3초 이내에 해제 | STOP→CW, CW↔CCW |
| `BTN_LONG_PRESSED` | 2 | 버튼을 누른 상태로 약 3초 경과 | CW 또는 CCW 상태에서 STOP |
| `BTN_UART` | 3 | UART로 `'f'`, `'s'`, `'r'` 중 하나 수신 | 수신 문자에 따라 CW, STOP, CCW |


### 3-4. 상태별 이벤트 처리표

| 현재 상태 | 입력 이벤트/명령 | 다음 상태 | 수행 함수 | 전환 전 정지 지연 |
|---|---|---|---|:---:|
| `MOTOR_STOP` | 버튼 짧게 누름 | `MOTOR_CW` | `motor_cw()` | 없음 |
| `MOTOR_STOP` | 버튼 길게 누름 | `MOTOR_STOP` | 없음 | 없음 |
| `MOTOR_CW` | 버튼 짧게 누름 | `MOTOR_CCW` | `motor_stop_delay()` → `motor_ccw()` | 적용 |
| `MOTOR_CW` | 버튼 길게 누름 | `MOTOR_STOP` | `motor_stop()` | 없음 |
| `MOTOR_CCW` | 버튼 짧게 누름 | `MOTOR_CW` | `motor_stop_delay()` → `motor_cw()` | 적용 |
| `MOTOR_CCW` | 버튼 길게 누름 | `MOTOR_STOP` | `motor_stop()` | 없음 |
| 모든 상태 | UART `'f'` | `MOTOR_CW` | `motor_cw()` | 미적용 |
| 모든 상태 | UART `'s'` | `MOTOR_STOP` | `motor_stop()` | 없음 |
| 모든 상태 | UART `'r'` | `MOTOR_CCW` | `motor_ccw()` | 미적용 |
| 모든 상태 | UART `'0'`~`'9'` | 현재 상태 유지 | `TIM5_Out_PWM_Generation()` | 해당 없음 |

## 4. 버튼 입력 처리

### 4-1. 버튼 판정 방식

| PC13 입력 | 판정 함수 | 의미 |
|:---:|---|---|
| LOW | `Key_Get_Pressed()` | 버튼 눌림 |
| HIGH | `Key_Get_Released()` | 버튼 해제 |

Falling edge와 Rising edge를 모두 검출  
ISR에서는 긴 처리를 하지 않고 `Key_event_generated`만 1로 설정한다. 실제 눌림·해제 판정은 메인 루프의 `button_scan()`에서 수행한다.

### 4-2. 짧게/길게 누름 판정 흐름

```mermaid
stateDiagram-v2
    [*] --> RELEASED
    RELEASED --> PRESSED_WAIT: Falling edge / TIM4 3000 ms 시작
    PRESSED_WAIT --> RELEASED: 3초 전 Rising edge / SHORT 이벤트
    PRESSED_WAIT --> LONG_REPORTED: TIM4 만료 / LONG 이벤트
    LONG_REPORTED --> RELEASED: Rising edge / 추가 이벤트 없음
```

1. 버튼을 누르면 EXTI 인터럽트가 발생한다.
2. `button_scan()`은 PC13이 LOW임을 확인하고 TIM4를 약 3000 ms로 시작한다.
3. 3000 ms 전에 버튼을 해제하면 TIM4를 정지하고 `BTN_SHORT_PRESSED`를 발생시킨다.
4. 버튼을 계속 누르면 TIM4 ISR이 `TIM4_Expired`를 1로 설정한다.
5. `button_check_long_press()`는 버튼이 여전히 눌려 있으면 `BTN_LONG_PRESSED`를 발생시키고 TIM4를 정지한다.
6. 길게 누름이 이미 발생한 뒤 버튼을 해제할 때는 짧게 누름 이벤트를 추가로 만들지 않는다.

## 5. UART 입력 처리

### 5-1. 수신 처리 구조

```mermaid
flowchart LR
    A["USART2 RX 인터럽트"] --> B["DR 1바이트 읽기"]
    B --> C["Uart_Data 저장"]
    C --> D["Uart_Data_In = 1"]
    D --> E["uart_processing()"]
    E --> F{"수신 문자"}
    F -->|0~9| G["PWM Duty 즉시 변경"]
    F -->|f / s / r| H["BTN_UART 이벤트 생성"]
    F -->|그 외| I["무시"]
```
### 5-2. UART 명령

| 명령 | 기능 | 결과 |
|:---:|---|---|
| `'f'` | 정방향 명령 | `motor_cw()` 호출 |
| `'s'` | 정지 명령 | `motor_stop()` 호출 |
| `'r'` | 역방향 명령 | `motor_ccw()` 호출 |
| `'0'` | 속도 단계 0 | PWM Duty 50% |
| `'1'` | 속도 단계 1 | PWM Duty 55% |
| `'2'` | 속도 단계 2 | PWM Duty 60% |
| `'3'` | 속도 단계 3 | PWM Duty 65% |
| `'4'` | 속도 단계 4 | PWM Duty 70% |
| `'5'` | 속도 단계 5 | PWM Duty 75% |
| `'6'` | 속도 단계 6 | PWM Duty 80% |
| `'7'` | 속도 단계 7 | PWM Duty 85% |
| `'8'` | 속도 단계 8 | PWM Duty 90% |
| `'9'` | 속도 단계 9 | PWM Duty 95% |

## 6. PWM 및 타이머

### 6-1. TIM5 모터 PWM

| 항목 | 설정값 |
|---|---|
| TIM5 기준 주파수 | 8 MHz |
| PWM 주파수 | 10 kHz |
| 출력 채널 | CH1, CH2 |
| 카운터 모드 | Down counter |
| 반복 모드 | Repeat |
| 초기 Duty | 75% (`range = 5`) |
| Duty 범위 | 50~95% |

### 6-2. TIM4 길게 누름 타이머

| 항목 | 설정값 |
|---|---|
| TIM4 Tick | 1000 μs |
| TIM4 기준 주파수 | 1 kHz |
| 길게 누름 설정값 | 3000 ms |
| 인터럽트 | Update interrupt, IRQ 30 |

## 7. 변수

### 7-1. 열거형 및 구조체

```c
typedef enum
{
    MOTOR_STOP,
    MOTOR_CW,
    MOTOR_CCW
} MOTOR_STATE;

typedef enum
{
    BTN_NONE,
    BTN_SHORT_PRESSED,
    BTN_LONG_PRESSED,
    BTN_UART
} BUTTON_EVENT;

typedef struct Button
{
    BUTTON_EVENT event;
    int pressed;
    int long_pressed;
} BUTTON;
```

### 7-2. 전역 변수

| 이름 | 형식 | 초기값 | 쓰는 위치 | 읽고 처리하는 위치 | 역할 |
|---|---|:---:|---|---|---|
| `Key_event_generated` | `volatile int` | 0 | `EXTI15_10_IRQHandler()` | `button_scan()` | PC13 edge 발생 알림 |
| `Uart_Data_In` | `volatile int` | 0 | `USART2_IRQHandler()` | `uart_processing()` | UART 새 데이터 수신 알림 |
| `Uart_Data` | `volatile unsigned char` | 0 | `USART2_IRQHandler()` | `uart_processing()`, `motor_handle()` | 최근 수신 문자 1바이트 |
| `TIM4_Expired` | `volatile int` | 0 | `TIM4_IRQHandler()` | `button_check_long_press()` | TIM4 만료 알림 |
| `btn` | `BUTTON` | `button_init()`에서 초기화 | 버튼/UART 처리 함수 | `motor_handle()` | 현재 입력 이벤트 및 버튼 상태 |
| `motor_state` | `MOTOR_STATE` | `MOTOR_STOP` | 모터 제어 함수 | `motor_handle()` | 현재 모터 상태 |

## 8. 인터럽트

| ISR | 발생 원인 | ISR 처리 | 후속 처리 |
|---|---|---|---|
| `EXTI15_10_IRQHandler()` | PC13 Falling/Rising edge | `Key_event_generated = 1`, EXTI/NVIC Pending clear | `button_scan()`이 현재 핀 상태를 판독 |
| `USART2_IRQHandler()` | USART2 RXNE | `USART2->DR`을 `Uart_Data`에 저장, `Uart_Data_In = 1` | `uart_processing()`이 명령 해석 |
| `TIM4_IRQHandler()` | TIM4 Update | TIM4/NVIC Pending clear, `TIM4_Expired = 1` | `button_check_long_press()`가 길게 누름 판정 |

인터럽트에서는 플래그 설정과 필수 Pending clear만 수행 ／ 상태 변경과 모터 구동은 메인 루프에서 수행

## 9. 주요 함수

### 9-1. main

| 함수 | 입력 | 출력/변경 | 역할 |
|---|---|---|---|
| `Sys_Init(int baud)` | Baud rate | 시스템 주변장치 설정 | 클록, USART2, 표준 출력, LED 초기화 |
| `TIM5_Out_Init()` | 없음 | TIM5·CCMR1·CCER 설정 | TIM5_CH1·CH2 PWM 출력 기능 초기화 |
| `TIM5_Out_PWM_Generation(int range)` | 0~9 | PSC, ARR, CCR1, CCR2, CR1 | 10 kHz PWM과 50~95% Duty 설정 |
| `TIM5_Out_Stop()` | 없음 | TIM5 CR1.CEN=0 | TIM5 정지. 현재 메인 흐름에서는 호출하지 않음 |
| `motor_init()` | 없음 | GPIO, `motor_state` | 모터 출력 LOW 및 정지 상태 초기화 |
| `motor_stop()` | 없음 | PA0·PA1 LOW, `MOTOR_STOP` | 모터 정지 |
| `PAx_PWM_set(int pin_num)` | 0 또는 1 | GPIOA MODER·AFR | 지정 핀을 AF2 PWM 출력으로 변경 |
| `PAx_GPIO_set(int pin_num)` | 0 또는 1 | GPIOA MODER·OTYPER·ODR | 지정 핀을 Push-pull GPIO LOW로 변경 |
| `motor_cw()` | 없음 | PA0 LOW, PA1 PWM, `MOTOR_CW` | 정방향 출력 설정 |
| `motor_ccw()` | 없음 | PA1 LOW, PA0 PWM, `MOTOR_CCW` | 역방향 출력 설정 |
| `motor_stop_delay()` | 없음 | 모터 정지 및 Busy-wait | 버튼 방향 전환 전 정지 구간 제공 |
| `motor_handle()` | `btn.event`, `Uart_Data` | 모터 상태·출력 변경 | 현재 상태와 입력 이벤트에 따라 모터 제어 후 이벤트 소거 |
| `button_init()` | 없음 | PC13, `btn` | 버튼 GPIO와 버튼 상태 초기화 |
| `button_scan()` | EXTI 플래그, PC13 | TIM4, `btn` | 누름·해제 판정 및 짧게 누름 이벤트 생성 |
| `button_check_long_press()` | TIM4 만료 플래그 | `btn` | 길게 누름 이벤트 생성 |
| `button_processing()` | 없음 | 버튼 관련 상태 | 버튼 스캔과 길게 누름 판정을 순서대로 호출 |
| `uart_processing()` | UART 플래그·문자 | PWM 또는 `BTN_UART` | 숫자·방향 명령 해석 |
| `Main()` | 없음 | 전체 시스템 | 초기화 후 세 처리 함수를 무한 반복 |
