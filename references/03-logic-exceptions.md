## Dimension 3: Logic Flow & Exception Reset (Ghost Bug Zone)

AI writes sequential code, but embedded systems are **interrupt-parallel**. These are the silent killers.

---

### Item 10: Independent Watchdog (IWDG)

| Aspect | Detail |
|--------|--------|
| **Consequence** | Program stuck in ISR or infinite loop → car runs wild at full speed |
| **AI Common Mistake** | Almost never enables watchdog proactively |

**What to check:**

```c
// ❌ CRITICAL FAIL — no watchdog anywhere in the code
// If PID calculation overflows or I2C hangs, the car will keep going.

// ✅ CORRECT — IWDG enabled, refreshed in main loop
// In main() initialization:
void IWDG_Init(void) {
    hiwdg.Instance = IWDG;
    hiwdg.Init.Prescaler = IWDG_PRESCALER_64;    // LSI(32kHz) / 64 = 500 Hz
    hiwdg.Init.Reload = 2000;                      // 2000 / 500 = 4 seconds timeout
    HAL_IWDG_Init(&hiwdg);
}

// In main loop — refresh ONLY in the main loop, NEVER inside an ISR
while (1) {
    Sensor_Update();
    PID_Compute();
    Motor_Control();
    HAL_IWDG_Refresh(&hiwdg);   // <-- Must be here, at end of each complete loop cycle
}

// Motor enable tied to IWDG reset:
// EN pin is HIGH only when MCU is actively refreshing.
// On IWDG reset → MCU resets → EN defaults LOW → motors STOP.
```

**Verification rules:**
- IWDG must be **explicitly enabled** in code (not just configured, actually started).
- `HAL_IWDG_Refresh()` must appear in the **main loop**, NOT inside any ISR (refreshing in ISR defeats the purpose — the ISR may keep firing even when main loop is stuck).
- Timeout must be reasonable: typically **100 ms to 4 seconds** for a motor control system. Too short → false resets on brief I2C glitches. Too long → car travels far before reset.
- **Safety integration**: Motor enable (EN) pin must be hardware-default LOW (motor off). On IWDG reset, MCU GPIOs tri-state or go to reset state → motors stop. Verify this is the case.
- If watchdog is already present check that it's not accidentally disabled in debug configurations (some AI code disables IWDG in `HAL_DBGMCU_EnableDBGStopMode`).

**Corrective action if FAIL:**
```c
// Mandatory IWDG setup for motor control
// Place in main() BEFORE any peripheral init

// STM32 IWDG:
void IWDG_Setup(void) {
    // IWDG uses LSI (~32 kHz, may vary 17-47 kHz across chips)
    // Choose prescaler so reload fits in 12-bit (max 4095)
    // Prescaler 64 → ~500 Hz → 500 Hz × 4s = 2000 (OK)
    IWDG->KR = 0x5555;          // Unlock
    IWDG->PR = 0x04;            // Prescaler 64 (500 Hz @ 32kHz LSI)
    IWDG->RLR = 2000;           // Reload = 2000 → ~4 second timeout
    IWDG->KR = 0xCCCC;          // Start watchdog
    // After this point, IWDG is RUNNING and must be refreshed
}

// In main loop, refresh at the END of each iteration:
while (1) {
    // ... all processing ...
    IWDG->KR = 0xAAAA;           // Refresh (equivalent to HAL_IWDG_Refresh)
}

// CRITICAL: test that IWDG actually works:
// 1. Add an infinite loop before the IWDG_Refresh call
// 2. Verify the MCU resets within the timeout period
// 3. Remove the test loop
```

---

### Item 11: Power-On Default State (Anti-Ghost-Kick)

| Aspect | Detail |
|--------|--------|
| **Consequence** | Motor violently kicks on power-up ("ghost kick") |
| **AI Common Mistake** | Enables motor before PWM is configured to 0% duty |

**What to check:**

The power-on sequence MUST follow this exact order. Any deviation is a FAIL:

```c
// ❌ CRITICAL FAIL — motor enabled during peripheral init
void System_Init(void) {
    MX_GPIO_Init();        // GPIO init — EN pin might float HIGH!
    MX_TIM1_Init();        // PWM timer starts — might output random duty!
    Motor_Enable();        // EN HIGH → motor KICKS with whatever PWM state exists
}

// ✅ CORRECT — strict power-on sequence
void System_Init(void) {
    // Step 1: Initialize GPIOs with EN pin LOW (motor disabled)
    MX_GPIO_Init();
    // EN pin must be pulled LOW by hardware default or explicit init
    HAL_GPIO_WritePin(MOTOR_EN_PORT, MOTOR_EN_PIN, GPIO_PIN_RESET);

    // Step 2: Initialize PWM timer with 0% duty cycle
    MX_TIM1_PWM_Init();
    // Explicitly set all PWM channels to 0
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 0);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, 0);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, 0);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, 0);

    // Step 3: Start PWM output
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
    // ... all channels ...

    // Step 4: Wait 100 ms for everything to stabilize
    HAL_Delay(100);

    // Step 5: ONLY NOW enable motor driver
    HAL_GPIO_WritePin(MOTOR_EN_PORT, MOTOR_EN_PIN, GPIO_PIN_SET);
}
```

**Verification rules:**
- **GPIO default**: EN pin must be configured to output LOW in `MX_GPIO_Init()`. Check `GPIO_InitStruct.Pull` — if it's `GPIO_NOPULL` and the pin floats high during reset, the motor will kick.
- **PWM zero**: Before `HAL_TIM_PWM_Start()`, all compare registers must be explicitly set to 0.
- **Delay**: There must be at least a **100 ms delay** between PWM start and EN assertion. This allows the timer, gate driver bootstrap capacitors, and power rails to stabilize.
- **Check the physical pull resistor**: The EN pin should have an **external pulldown resistor** (10kΩ to GND) so that even during MCU reset (GPIOs tri-state), the motor driver stays disabled.
- If there's a bootloader, check that EN stays LOW during the bootloader delay period.

**Corrective action if FAIL:**
```c
// Safe power-on sequence — use this EXACT pattern
void SafeMotorInit(void) {
    // 1. EN pin LOW first — before ANY PWM config
    HAL_GPIO_WritePin(MOTOR_EN_GPIO_Port, MOTOR_EN_Pin, GPIO_PIN_RESET);

    // 2. Initialize PWM timer
    MX_TIM1_Init();

    // 3. Force all PWM channels to 0% duty
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 0);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, 0);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, 0);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, 0);

    // 4. Start PWM output at 0% duty
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_2);
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_3);
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_4);

    // 5. Wait for stabilization
    HAL_Delay(100);

    // 6. Enable motor driver
    HAL_GPIO_WritePin(MOTOR_EN_GPIO_Port, MOTOR_EN_Pin, GPIO_PIN_SET);

    // HARDWARE CHECK: measure EN pin voltage with oscilloscope during power-up.
    // It must stay at 0V until after the 100ms delay. Any pulse above 0.5V → add external pulldown.
}
```

---

### Item 12: Interrupt Priority Grouping

| Aspect | Detail |
|--------|--------|
| **Consequence** | Attitude update preempted by UART ISR → late computation → PWM blanking → motor loss of control |
| **AI Common Mistake** | No priority grouping configured at all |

**What to check:**

```c
// ❌ FAIL — default priorities: all interrupts at same level, no preemption
// UART RX ISR can block the sensor timer ISR → stale attitude data

// ✅ CORRECT — explicit priority grouping
// In main(), before any peripheral init:
HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);  // 4 bits preemption, 0 bits subpriority

// Timer interrupts (attitude update, motor control): HIGHEST priority
HAL_NVIC_SetPriority(TIM7_IRQn,   0, 0);  // 1 kHz sensor update → Preempt priority 0 (highest)
HAL_NVIC_SetPriority(TIM1_UP_TIM10_IRQn, 1, 0);  // Motor PWM update → Priority 1

// Communication: LOWEST priority
HAL_NVIC_SetPriority(USART1_IRQn, 14, 0);  // Bluetooth/Debug UART → Priority 14
HAL_NVIC_SetPriority(USART2_IRQn, 15, 0);  // Wireless module → Priority 15 (lowest)

// I2C (sensor): medium priority — needs timely service but below motor control
HAL_NVIC_SetPriority(I2C1_EV_IRQn,  2, 0);
HAL_NVIC_SetPriority(I2C1_ER_IRQn,  2, 0);
```

**Verification rules:**
- Priority grouping must be explicitly set (prefer `NVIC_PRIORITYGROUP_4` for maximum preemption control).
- **Hard real-time ISRs** (sensor timer, motor control timer, encoder overflow) → preempt priority **0–1**.
- **Medium ISRs** (I2C DMA complete, ADC complete) → preempt priority **2–5**.
- **Soft real-time / best-effort ISRs** (UART RX/TX, SPI, SysTick) → preempt priority **10–15**.
- No two ISRs that can preempt each other should share the same priority level unless their execution time is guaranteed trivial (< 1 µs).
- Check for nested interrupt issues: if UART ISR uses `HAL_Delay()` (which relies on SysTick), and SysTick priority is lower than UART, the system hangs. This pattern must be flagged as FAIL.

**Corrective action if FAIL:**
```c
// Interrupt priority hierarchy for motor control system
// Lower number = higher priority (Cortex-M convention)

// Priority 0 (highest): Motion-critical timer ISRs
//   - Sensor reading timer (1 kHz MPU6050)
//   - PID compute + PWM update timer
// Priority 1: Encoder timer overflow
// Priority 2-3: I2C/SPI DMA interrupts (sensor data transfer)
// Priority 4-5: ADC completion (battery monitoring)
// Priority 10-11: SysTick
// Priority 14: Debug UART
// Priority 15 (lowest): Wireless/bluetooth UART

// Example setup:
HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);  // MUST be called first

HAL_NVIC_SetPriority(TIM7_IRQn,           0, 0);  // Motion timer
HAL_NVIC_SetPriority(TIM1_UP_TIM10_IRQn,  0, 0);  // PWM update
HAL_NVIC_SetPriority(TIM2_IRQn,           1, 0);  // Encoder timer
HAL_NVIC_SetPriority(I2C1_EV_IRQn,        2, 0);  // MPU6050 I2C event
HAL_NVIC_SetPriority(I2C1_ER_IRQn,        2, 0);  // MPU6050 I2C error
HAL_NVIC_SetPriority(ADC_IRQn,            4, 0);  // Battery ADC
HAL_NVIC_SetPriority(SysTick_IRQn,       10, 0);  // System tick
HAL_NVIC_SetPriority(USART1_IRQn,        14, 0);  // Debug UART
HAL_NVIC_SetPriority(USART2_IRQn,        15, 0);  // Bluetooth UART
```

---

### Item 13: Math Exception Handling (NaN / Inf)

| Aspect | Detail |
|--------|--------|
| **Consequence** | `sqrt()`/`asin()` overflow → NaN in PID output → 100% duty cycle → car crashes at full speed |
| **AI Common Mistake** | No math validity checking at all |

**What to check:**

```c
// ❌ CRITICAL FAIL — unbounded math input, unchecked PID output
float angle = asin(ay / sqrt(ax*ax + ay*ay + az*az));  // If sqrt=0 or ratio>1 → NaN!
float motor_pwm = PID_Compute(&pid, target, angle);
SetMotorPWM(motor_pwm);   // NaN propagates to PWM register → undefined behavior

// ✅ CORRECT — input clamping + output NaN guard
float Safe_ASin(float x) {
    if (x > 1.0f)  x = 1.0f;
    if (x < -1.0f) x = -1.0f;
    if (isnan(x))  return 0.0f;   // Safety fallback
    return asinf(x);
}

float Safe_Sqrt(float x) {
    if (x < 0.0f)  x = 0.0f;      // Negative → 0 (shouldn't happen, but guard)
    if (isnan(x))  return 0.0f;
    return sqrtf(x);
}

float PID_Compute_Safe(PID_TypeDef *pid, float setpoint, float measurement) {
    if (isnan(setpoint) || isnan(measurement)) {
        Error_Handler();  // Sensor failure → stop everything
    }
    // ... PID computation (with item 7 anti-windup) ...
    return output;
}

// In motor control:
float pid_out = PID_Compute_Safe(&pid, target, current);
if (isnan(pid_out) || isinf(pid_out)) {
    Error_Handler();  // Halt all motors immediately
}
SetMotorPWM(pid_out);
```

**Verification rules:**
- **Every** call to `asin()`, `acos()`, `atan2()` must have its input argument clamped to `[-1.0, 1.0]` (with a small epsilon margin: `[-0.9999, 0.9999]`).
- **Every** call to `sqrt()` (and `log()`, `pow()` if used) must have its input clamped to `≥ 0`.
- **Division operations**: denominator must be checked for `|denom| < 1e-9` before division.
- **PID output**: before writing to any PWM register, the output must be checked with `isnan()` and `isinf()`. If true → call `Error_Handler()` (disable all motors, enter infinite loop with LED error pattern).
- **`arm_math.h` CMSIS-DSP functions**: if using CMSIS-DSP (`arm_sqrt_f32`, etc.), these also need input guarding — they can also produce NaN.
- Search for **any floating-point operation** in the motor control path and verify guard code on the same line or immediately before.

**Corrective action if FAIL:**
```c
// Math safety wrappers — use these INSTEAD of raw math.h functions
// everywhere in motor control and attitude code

#include <math.h>

// Clamp and guard asin/acos input
static inline float safe_asin(float x) {
    if (isnan(x)) return 0.0f;
    if (x >  0.9999f) x =  0.9999f;
    if (x < -0.9999f) x = -0.9999f;
    return asinf(x);
}

static inline float safe_acos(float x) {
    if (isnan(x)) return 0.0f;
    if (x >  0.9999f) x =  0.9999f;
    if (x < -0.9999f) x = -0.9999f;
    return acosf(x);
}

// Guard sqrt input
static inline float safe_sqrt(float x) {
    if (isnan(x) || x < 0.0f) return 0.0f;
    return sqrtf(x);
}

// Guard division
static inline float safe_div(float num, float den) {
    if (fabsf(den) < 1e-9f) return 0.0f;
    float result = num / den;
    if (isnan(result) || isinf(result)) return 0.0f;
    return result;
}

// NaN guard for motor output — call this on the final PWM value
static inline void guard_motor_output(float pwm_value) {
    if (isnan(pwm_value) || isinf(pwm_value)) {
        // CRITICAL: disable all motors immediately
        HAL_GPIO_WritePin(MOTOR_EN_GPIO_Port, MOTOR_EN_Pin, GPIO_PIN_RESET);
        // Stop all PWM channels
        HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_1);
        HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_2);
        // Signal error
        Error_Handler();  // Infinite loop with LED SOS pattern
    }
}

// Use in motor control path:
float output = PID_Compute(&pid, target, measurement);
guard_motor_output(output);
```

---

---

