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

![FSM 다이어그램](./image/FSM.png)

### 1-2. MOTOR STATE

| 이름 | 값 | PA0 | PA1 | 설명 |
|---|:---:|---|---|---|
| `MOTOR_STOP` | 0 | GPIO LOW | GPIO LOW | 모터 정지 |
| `MOTOR_CW` | 1 | GPIO LOW | TIM5_CH2 PWM | 소프트웨어 기준 정방향 회전 |
| `MOTOR_CCW` | 2 | TIM5_CH1 PWM | GPIO LOW | 소프트웨어 기준 역방향 회전 |


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

### 2-1. BUTTON/UART 이벤트
| 이름 | 값 | 설명 |
|---|:---:|---|
| `EVT_NONE` | 0 | 이벤트 없음 / 이벤트 소비 완료 |
| `EVT_BTN_SHORT` | 1 | 버튼 짧게 눌림 (버튼 눌린 시점부터 뗀 시간까지 측정. 눌린 시간 < 3000ms) |
| `EVT_BTN_LONG` | 2 | 버튼 길게 눌림 (버튼이 눌린 채로 경과 시간 >= 3000ms) |
| `EVT_UART` | 3 | UART로 문자 수신 (수신 문자는 ISR에서 Uart_Data_In에 저장) |

### 2-2. 상태별 이벤트 처리표

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

## 3. PWM 및 타이머

### 3-1. TIM5 모터 PWM

| 항목 | 설정값 |
|---|---|
| TIM5 기준 주파수 | 8 MHz |
| PWM 주파수 | 10 kHz |
| 출력 채널 | CH1, CH2 |
| 카운터 모드 | Down counter |
| 반복 모드 | Repeat |
| 초기 Duty | 75% (`range = 5`) |
| Duty 범위 | 50~95% |

### 3-2. TIM4 버튼 3초 인지 타이머

| 항목 | 설정값 |
|---|---|
| TIM4 Tick | 1000 μs |
| TIM4 기준 주파수 | 1 kHz |
| 길게 누름 설정값 | 3000 ms |
| 인터럽트 | Update interrupt, IRQ 30 |

## 4. VARIABLE

### 4-1. 열거형 및 구조체

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

### 4-2. GLOBAL VARIABLE

| 이름 | Type | 초기값 | 역할 | 변경 시점 |
|---|---|---|---|---|
| `evt` | `event_t` | `EVT_NONE` | 현재 이벤트 저장 | BUTTON, UART 인터럽트 발생시`check_event()` 에서 갱신|
| `motor_state` | `motor_state_t` | `MOTOR_STOP` | 현재 모터 상태 저장 | `drive_motor()`에서 모터 상태 변경 시 갱신 |
| `new_motor_state` | `motor_state_t` | `MOTOR_STOP` | 새로운 모터 회전 저장 | `update_motor_state()`에서 이벤트 처리 시 갱신 |
| `motor_speed` | uint8_t | `5` | 현재 PWM Duty 레벨 (0~9) | `drive_motor()`에서 모터 상태 변경 시 갱신 |
| `new_motor_speed` | uint8_t | `5` | 새로운 PWM Duty 레벨 (0~9) | `update_motor_state()`에서 이벤트 처리 시 갱신 |
| `btn_flag` | volatile uint8_t | `0` | 버튼 인터럽트 (falling edge/rising edge) 발생 여부 | Button_ISR에서 1로 세팅 → `check_event()`에서 읽은 직후 0으로 리셋 |
| `uart_flag` | volatile uint8_t | `0` | UART 인터럽트 발생 여부 | `USART2_IRQHandler()`에서 1로 세팅 → `check_event()`에서 읽은 직후 0으로 리셋 |
| `uart_data` | volatile uint8_t | `0` | UART로 수신된 문자 | `USART2_IRQHandler()`에서 수신 시 갱신 |
| `timer_flag` | volatile uint8_t | `0` | 타이머 인터럽트 발생 여부  | `TIM4_IRQHandler()`에서 갱신 |

## 5. METHOD

### 5-1. 주요 METHOD

| 이름 | 입력 | 출력 | 설명 | 호출 시점 |
|---|---|---|---|---|
| `check_event()` | `btn_flag`, `uart_flag` (전역 참조) | `evt` 갱신 | `btn_flag`가 세워져 있으면 핀 상태로 falling/rising을 판별해 `EVT_BTN_SHORT` 여부를 계산하고, 눌린 채로 3초가 지나면 `EVT_BTN_LONG`을 생성. `uart_flag`가 세워져 있으면 `EVT_UART`를 생성. | main 루프 |
| `update_motor_state()` | `evt`, `motor_state`, `motor_speed` (전역 참조) | `new_motor_state`, `new_motor_speed` 갱신 | `evt`가 `EVT_BTN_SHORT`면 CW/CCW를 토글, `EVT_BTN_LONG`이면 `MOTOR_STOP`으로 전환. `EVT_UART`면 `uart_data`가 숫자인지 'f', 'r', 's'인지에 따라 `new_motor_speed` 또는 `new_motor_state`를 갱신. | main 루프, `evt != EVT_NONE`일 때만 호출 |
| `drive_motor()` | `new_motor_state`, `new_motor_speed` (전역 참조) | GPIO 방향 설정, PWM Duty 반영, `motor_state`, `motor_speed` 갱신 | `new_motor_state`와 `motor_state`, `new_motor_speed`와 `motor_speed`를 비교하여 다를 때만 GPIO 방향을 설정하고 PWM Duty를 반영 | main 루프 |

### 5-2. ISR 

| 이름 | 출력 | 설명 | 호출 시점 |
|---|---|---|---|
| `EXTI15_10_IRQHandler()` | `btn_flag` 갱신 | `btn_flag`를 1로 세팅 | PC13 USER KEY (falling edge, rising edge) 인터럽트 발생 |
| `USART2_IRQHandler()` | `uart_flag` 갱신 | 입력 받은 문자를 `uart_data`에 저장하고, `uart_flag`를 1로 세팅 | USART2 인터럽트 발생  |
| `TIM4_IRQHandler()` | `timer_flag` 갱신 | `timer_flag`를 1로 세팅 | 타이머 인터럽트(3000ms) 발생 |

### 5-3. TIMER 제어

| 함수 | 입력 값 | 출력 값 | 변경하는 변수 | 역할 |
|---|---|---|---|---|
| `TIM5_Out_Init()` | 없음 | 없음 (`void`) | `RCC->APB1ENR`, `TIM5->CCMR1`, `TIM5->CCER` | TIM5_CH1·CH2 PWM 출력 기능 초기화 |
| `TIM5_Out_PWM_Generation(int range)` | `range` 0~9 | 없음 (`void`) | `TIM5->PSC`, `ARR`, `CCR1`, `CCR2`, `EGR`, `CR1` | 10 kHz PWM과 50~95% Duty 설정 |
| `TIM5_Out_Stop()` | 없음 | 없음 (`void`) | `TIM5->CR1`의 CEN 비트 | TIM5 정지. 현재 메인 흐름에서는 호출하지 않음 |
| `TIM4_Repeat_Interrupt_Enable(int en, int time)` | `en`, `time`(ms) | 없음 (`void`) | RCC·TIM4 관련 레지스터와 NVIC IRQ 30 설정 | 지정 시간으로 TIM4 Update 인터럽트 시작 또는 비활성화 |
| `TIM4_Stop()` | 없음 | 없음 (`void`) | `TIM4->CR1`, `TIM4->DIER`, `TIM4->SR`, NVIC IRQ 30 Pending 상태 | TIM4와 Update 인터럽트를 정지하고 Pending 상태 정리 |

### 5-4. 모터 설정

| 함수 | 입력 값 | 출력 값 | 변경하는 변수 | 역할 |
|---|---|---|---|---|
| `motor_init()` | 없음 | 없음 (`void`) | `RCC->AHB1ENR`, `GPIOA->MODER`, `OTYPER`, `ODR`, `motor_state` | 모터 출력 LOW 및 정지 상태 초기화 |
| `PAx_PWM_set(int pin_num)` | `pin_num` 0 또는 1 | 없음 (`void`) | `GPIOA->MODER`, `GPIOA->AFR[0]` | 지정 핀을 AF2 PWM 출력으로 변경 |
| `PAx_GPIO_set(int pin_num)` | `pin_num` 0 또는 1 | 없음 (`void`) | `GPIOA->MODER`, `GPIOA->OTYPER`, `GPIOA->ODR` | 지정 핀을 Push-pull GPIO LOW로 변경 |
| `motor_stop()` | 없음 | 없음 (`void`) | `GPIOA->MODER`, `GPIOA->ODR`, `motor_state` | PA0·PA1을 LOW로 설정하고 모터 정지 |
| `motor_cw()` | 없음 | 없음 (`void`) | `GPIOA->MODER`, `OTYPER`, `ODR`, `AFR[0]`, `motor_state` | PA0은 LOW, PA1은 PWM으로 설정하여 정방향 출력 |
| `motor_ccw()` | 없음 | 없음 (`void`) | `GPIOA->MODER`, `OTYPER`, `ODR`, `AFR[0]`, `motor_state` | PA1은 LOW, PA0은 PWM으로 설정하여 역방향 출력 |
| `motor_stop_delay()` | 없음 | 없음 (`void`) | `GPIOA->MODER`, `GPIOA->ODR`, `motor_state` | 모터를 정지하고 Busy-wait하여 버튼 방향 전환 전 정지 구간 제공 |
| `motor_handle()` | `motor_state`, `btn.event`, `Uart_Data` | 없음 (`void`) | `motor_state`, `btn.event`, 모터 출력 관련 GPIOA 레지스터 | 현재 상태와 입력 이벤트에 따라 모터를 제어하고 처리한 이벤트 소거 |

### 5-5. 버튼 입력 및 처리

| 함수 | 입력 값 | 출력 값 | 변경하는 변수 | 역할 |
|---|---|---|---|---|
| `button_init()` | 없음 | 없음 (`void`) | `RCC->AHB1ENR`, `GPIOC->MODER`, `GPIOC->PUPDR`, `btn.event`, `btn.pressed`, `btn.long_pressed` | PC13 입력·Pull-up과 버튼 상태 초기화 |
| `button_scan()` | `Key_event_generated`, PC13 입력, `btn.long_pressed` | 없음 (`void`) | `Key_event_generated`, `btn.event`, `btn.pressed`, `btn.long_pressed`, TIM4 관련 레지스터 | 버튼 누름·해제를 판정하고 짧게 누름 이벤트 생성 |
| `button_check_long_press()` | `TIM4_Expired`, `btn.pressed` | 없음 (`void`) | `TIM4_Expired`, `btn.event`, `btn.long_pressed`, TIM4 관련 레지스터 | TIM4 만료 시 버튼이 계속 눌렸는지 확인하고 길게 누름 이벤트 생성 |
| `button_processing()` | 버튼·TIM4 전역 상태 | 없음 (`void`) | 하위 함수를 통해 버튼 상태, 이벤트 플래그, TIM4 관련 레지스터 변경 | 버튼 스캔과 길게 누름 판정을 순서대로 호출 |
| `Key_Get_Pressed()` | `GPIOC->IDR`의 PC13 값 | `int`: 눌림이면 1, 아니면 0 | 없음 | PC13이 LOW인지 확인 |
| `Key_Get_Released()` | `GPIOC->IDR`의 PC13 값 | `int`: 해제이면 1, 아니면 0 | 없음 | PC13이 HIGH인지 확인 |
| `Key_ISR_Enable(int en)` | `en` | 없음 (`void`) | RCC·GPIOC·SYSCFG·EXTI 관련 레지스터와 NVIC IRQ 40 설정 | EXTI13 양쪽 edge 인터럽트 활성화 또는 NVIC 인터럽트 비활성화 |

### 5-6. UART

| 함수 | 입력 값 | 출력 값 | 변경하는 변수 | 역할 |
|---|---|---|---|---|
| `uart_processing()` | `Uart_Data_In`, `Uart_Data` | 없음 (`void`) | `Uart_Data_In`, `btn.event`, TIM5 PWM 관련 레지스터 | 숫자 명령은 PWM Duty로, `f/s/r` 명령은 모터 이벤트로 변환 |
| `Uart2_Init(int baud)` | `baud` | 없음 (`void`) | RCC·GPIOA·USART2 관련 레지스터 | PA2·PA3를 AF7로 설정하고 USART2 초기화 |
| `Uart2_RX_Interrupt_Enable(int en)` | `en` | 없음 (`void`) | `USART2->CR1`의 RXNEIE 비트, NVIC IRQ 38 설정 | USART2 수신 인터럽트 활성화 또는 비활성화 |

### 5-7. IRQ

| ISR | 발생 원인 | ISR 처리 | 후속 처리 |
|---|---|---|---|
| `EXTI15_10_IRQHandler()` | PC13 Falling/Rising edge | `Key_event_generated = 1`, EXTI/NVIC Pending clear | `button_scan()`이 현재 핀 상태를 판독 |
| `USART2_IRQHandler()` | USART2 RXNE | `USART2->DR`을 `Uart_Data`에 저장, `Uart_Data_In = 1` | `uart_processing()`이 명령 해석 |
| `TIM4_IRQHandler()` | TIM4 Update | TIM4/NVIC Pending clear, `TIM4_Expired = 1` | `button_check_long_press()`가 길게 누름 판정 |
