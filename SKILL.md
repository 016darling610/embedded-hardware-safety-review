---
name: embedded-hardware-safety-review
description: >-
  60-point hardware safety review for AI-generated embedded firmware code (STM32, DC/BLDC motor control, MPU6050 attitude sensing, PID control, servo, balancing robot, encoder, mecanum, Ackermann, I2C/SPI/UART/DMA comms, wireless remote, state machine, camera/vision fusion, competition logic, pre-race lockdown, defense-in-depth, emergency stop, post-mortem). Use this whenever the user submits embedded code for safety review, asks for code review of motor driver / STM32 / MPU6050 / PWM / PID / encoder / servo / I2C / balancing / mecanum / Ackermann / state-machine / competition code, mentions hardware safety concerns (烧毁、冒烟、失控、炸驱动、卡死、飞车、变砖、HardFault、推头、甩尾、冲线误判、虚电猝死、同频干扰、重心偏移、无日志、无法回退), or says keywords like 嵌入式安全审查、电机驱动审查、STM32安全、PWM安全、MPU6050安全、姿态解算审查、舵机安全、平衡车安全、麦轮安全、竞赛安全、封车安全. Also trigger proactively when you generate embedded motor control, sensor fusion, state-machine, or competition-strategy code — review your own output before presenting it to the user. Covers 20 dimensions across 60 items.
---

# Embedded Hardware Safety Review — 13-Point Checklist

## Purpose

AI-generated embedded code is syntactically correct but **physically dangerous**. LLMs do not understand voltage, current, or thermal dissipation — they treat GPIO configuration, PWM timers, and PID loops as abstract text with no physical consequence. This skill provides a systematic 60-point safety review across twenty dimensions: hardware physics, sensor algorithms, logic flow, power integrity, bus communication, control coupling, data persistence, code structure, kinematics, calibration, CPU/DMA, wireless/modes, state machine, system integrity, battery-aware control, multi-sensor fusion, and competition logic.

When you apply this skill, you are acting as a **hardware safety auditor**, not a code-style reviewer. Every item must be checked. If an item cannot be verified from the code alone, flag it as **NEEDS MANUAL VERIFICATION** and explain what physical measurement the developer must perform.

---

## Review Process

### Step 1: Identify the Code's Scope

Before reviewing, determine which subsystems are present:
- **Motor driver** (PWM, H-bridge, MOS driver, encoder) → Check items 1, 2, 3, 4, 9, 11, 18, 30, 43
- **Battery-powered** (LiPo, 2S/3S/4S) → Check items 5, 19, 43
- **MPU6050 / IMU / attitude** (DMP, quaternion, Euler angle, PID) → Check items 6, 7, 8, 13, 31, 44
- **Full system / main loop** (scheduler, interrupts, error handling) → Check items 10, 11, 12, 13
- **I2C/SPI sensors** (MPU6050, OLED, external ADC) → Check items 19, 21
- **Servo** (steering or pan/tilt) → Check item 20 **[conditional]**
- **Two-wheel balancing robot** → Check item 23 **[conditional]**
- **UART/Bluetooth/WiFi** (wireless comms) → Check items 22, 26, 33
- **Line-following / visual tracking** → Check items 24, 35
- **Mecanum / omni / swerve** → Check items 29, 46 **[conditional]**
- **Ackermann (rack-and-pinion steering)** → Check item 46 **[conditional]**
- **Encoder odometry** → Check items 9, 45 **[conditional]**
- **DMA-based ADC/sensor reading** → Check items 32, 39
- **Camera + IMU fusion** → Check item 47 **[conditional]**
- **Competition start/finish line** → Check items 48, 49 **[conditional]**

### Step 2: Check Each Applicable Item

Go through every item in the three dimensions below. For each item, you must:
1. Read the relevant code
2. Compare against the **Correct Approach** and the **AI Common Mistake**
3. Report: ✅ PASS / ❌ FAIL / ⚠️ NEEDS MANUAL VERIFICATION
4. For FAIL and WARNING: provide the exact fix with code snippet

### Step 3: Rate Overall Severity

After completing all checks, assign an overall severity:
- **CRITICAL** — one or more items can cause hardware destruction (fire, smoke, permanent damage) on first power-up
- **HIGH** — one or more items can cause loss of control, crash, or damage over time
- **MEDIUM** — items found that degrade performance or reliability but won't destroy hardware
- **LOW** — only minor concerns or all items pass

---

## Dimension 1: Hardware Physical Safety (Destruction / Smoke Risk)

AI does not understand current and voltage. You must **manually intercept** these 5 items.

---

### Item 1: PWM Frequency

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Motor whining + overheating, MOS transistor burnout |
| **AI Common Mistake** | Uses 50 Hz (servo frequency) or 1 kHz |

**What to check:**

```c
// ❌ CRITICAL FAIL — AI-generated garbage values
#define PWM_FREQUENCY  50       // This is servo frequency! Will destroy DC motor driver.
#define PWM_FREQ        1000    // Still wrong — audible whine, poor efficiency.

// ✅ CORRECT — DC motor forced into 10 kHz ~ 20 kHz range
#define PWM_FREQUENCY  15000    // Beyond human hearing, peak driver efficiency
```

**Verification rules:**
- For **DC brushed motors** with H-bridge: PWM carrier must be **10 kHz to 20 kHz**. Lock it to 15000 Hz.
- For **BLDC / FOC**: confirm the modulation frequency matches the driver IC datasheet.
- For **servos**: 50 Hz is actually correct (50 Hz = 20 ms frame). This is the ONLY exception.
- Check that the timer prescaler + ARR combination actually produces the declared frequency. AI often writes mismatched values (e.g., declares 15000 but prescaler math gives 450 Hz).

**Corrective action if FAIL:**
```c
// Force-lock the PWM frequency in the timer init
#define PWM_FREQUENCY_HZ   15000
// Recalculate ARR and PSC based on timer clock:
// ARR = (TIM_CLK / (PSC + 1) / PWM_FREQUENCY_HZ) - 1
```

---

### Item 2: Duty Cycle Limiting

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Motor stall → current surge → driver IC explosion |
| **AI Common Mistake** | PID output directly written to `__HAL_TIM_SET_COMPARE` with no limit |

**What to check:**

```c
// ❌ CRITICAL FAIL — unbounded PID output goes straight to hardware
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, (int)PID_Output);

// ✅ CORRECT — constrain before writing to hardware register
#define MAX_DUTY_CYCLE   85    // 85% maximum, leave 15% for battery sag
#define MIN_DUTY_CYCLE   -85   // For bidirectional drive

int32_t PWM_Out = (int32_t)PID_Output;
PWM_Out = constrain(PWM_Out, -MAX_DUTY_CYCLE, MAX_DUTY_CYCLE);
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, abs(PWM_Out));
```

**Verification rules:**
- `MAX_DUTY` must **never exceed 85%** (leave 15% margin for battery voltage fluctuation).
- The `constrain()` or equivalent must be applied to **every PWM write path** — not just the PID path. Check for direct register writes elsewhere.
- For bidirectional motor control, check both positive and negative bounds.
- Verify that the constraint is in terms of the timer's **ARR value**, not an arbitrary integer. `MAX_DUTY_CYCLE` should translate to `(ARR * 85 / 100)`.

**Corrective action if FAIL:**
```c
// Safe duty cycle limiting pattern
static inline void SetMotorPWM(int32_t pwm_value) {
    int32_t limited = (pwm_value > MAX_DUTY) ? MAX_DUTY :
                      (pwm_value < -MAX_DUTY) ? -MAX_DUTY : pwm_value;
    uint32_t compare_value = (limited >= 0) ? limited : -limited;
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, compare_value);
    // Set direction pin based on sign of 'limited'
    HAL_GPIO_WritePin(DIR_PORT, DIR_PIN, (limited >= 0) ? GPIO_PIN_SET : GPIO_PIN_RESET);
}
```

---

### Item 3: Pin Mode Conflicts

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | IO pin burnout / MCU latch-up |
| **AI Common Mistake** | Configures encoder AB-phase input pins as `GPIO_MODE_OUTPUT_PP` |

**What to check:**

Every GPIO pin connected to an **input signal** must be explicitly configured as input:

```c
// ❌ CRITICAL FAIL — input pin configured as push-pull output
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;  // On encoder A pin!
GPIO_InitStruct.Pull = GPIO_NOPULL;

// ✅ CORRECT — input pin explicitly configured
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;       // For plain digital input
// or
GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;       // For timer encoder mode (open-drain)
GPIO_InitStruct.Pull = GPIO_PULLUP;           // Pull-up recommended for encoder signals
```

**Verification rules:**
- **Encoder AB-phase pins**: Must be `GPIO_MODE_AF_PP` (for timer encoder mode with internal pull) or `GPIO_MODE_INPUT`. Never `GPIO_MODE_OUTPUT_PP`.
- **MPU6050 I2C pins (SCL/SDA)**: Must be `GPIO_MODE_AF_OD` with pull-up enabled. Never `GPIO_MODE_OUTPUT_PP`.
- **Battery voltage ADC pin**: Must be `GPIO_MODE_ANALOG`. Never `GPIO_MODE_OUTPUT_PP`.
- **Emergency stop / limit switch pins**: Must be `GPIO_MODE_INPUT` with appropriate pull-up/down.
- **Motor enable pin**: This IS an output — `GPIO_MODE_OUTPUT_PP` is correct here. But check that it initializes LOW (motor disabled).

**Corrective action if FAIL:**
```c
// Explicit pin mode audit pattern
// For each GPIO pin in the design, verify:
// INPUT pins  → GPIO_MODE_INPUT, GPIO_MODE_AF_OD, or GPIO_MODE_ANALOG
// OUTPUT pins → GPIO_MODE_OUTPUT_PP or GPIO_MODE_AF_PP

// After power-on init, use a multimeter to verify:
// - Input pins must NOT be forcibly pulled HIGH or LOW (voltage should float or match pull resistor)
// - Report: "Measure pin voltage at [PIN_NAME] — if stuck at 0V or 3.3V, pin mode is wrong"
```

---

### Item 4: Motor Dead Zone Compensation

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Violent low-speed vibration, mechanical looseness |
| **AI Common Mistake** | Guesses dead zone value (e.g., `PWM_Deadzone = 50`) with no basis |

**What to check:**

```c
// ❌ FAIL — arbitrary value, AI guessed this number
#define PWM_DEADZONE   50    // Where did 50 come from? The AI doesn't know.

// ✅ CORRECT — value determined by physical measurement
#define PWM_DEADZONE   37    // Measured: motor just starts rotating at duty=37 @ 12V
```

**Verification rules:**
- If the code contains a `DEADZONE` or `DEADBAND` constant, check whether the comments explain how it was measured.
- AI-generated dead zone values are **never trustworthy** — they are invariably guessed.
- The correct method: gradually increase PWM duty cycle from 0, record the value at which the motor just barely starts turning smoothly. This is the dead zone. Repeat at different battery voltages.

**Corrective action if FAIL:**
```
⚠️ NEEDS MANUAL VERIFICATION — Dead zone measurement required

  The value PWM_DEADZONE = [current_value] in the code was likely guessed by AI.

  Physical measurement procedure:
  1. Set PWM frequency to the target value (e.g., 15000 Hz)
  2. Start at duty cycle = 0, increment by 1 every 200 ms
  3. Record the duty cycle at which the motor JUST begins to rotate smoothly
  4. This recorded value is the dead zone threshold
  5. Replace the macro with the measured value AND add a comment:
     // Measured 2024-XX-XX: motor starts at duty=[N] @ 12.0V supply
  
  Important: different motors have different dead zones — even the same model.
  Do NOT copy dead zone values between motors.
```

---

### Item 5: Battery Voltage Monitoring

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Low battery → unstable servo/motor supply → MCU brownout reset → runaway |
| **AI Common Mistake** | Completely ignores voltage monitoring |

**What to check:**

```c
// ❌ CRITICAL FAIL — no battery voltage monitoring at all
// The code assumes infinite power supply.

// ✅ CORRECT — ADC monitors battery, disables motors on low voltage
#define BATTERY_LOW_THRESHOLD_3S  11.1f   // 3S LiPo: 3.7V/cell × 3 = 11.1V
#define BATTERY_LOW_THRESHOLD_2S  7.0f    // 2S LiPo: 3.5V/cell × 2 = 7.0V
#define BATTERY_CRITICAL_3S       10.5f   // 3S: emergency stop

void CheckBatteryVoltage(void) {
    uint32_t adc_value = 0;
    for (int i = 0; i < 8; i++) {  // Oversample 8x for noise reduction
        adc_value += ADC_Read(BATTERY_ADC_CHANNEL);
    }
    adc_value /= 8;
    float battery_voltage = (adc_value / 4095.0f) * VREF * VOLTAGE_DIVIDER_RATIO;

    if (battery_voltage < BATTERY_LOW_THRESHOLD) {
        MotorDisable();           // CUT motor enable immediately
        LED_Alarm_Blink();        // Blink warning LED
        // Enter safe state: motors OFF, only LED blinking
    }
}
```

**Verification rules:**
- Must have an **ADC channel** reading battery voltage (through a voltage divider).
- Must have a **low-voltage threshold** appropriate for the battery chemistry and cell count:
  - 2S LiPo: threshold at **7.0V** (3.5V/cell), critical at 6.4V
  - 3S LiPo: threshold at **11.1V** (3.7V/cell), critical at 10.5V
  - 4S LiPo: threshold at **14.8V** (3.7V/cell), critical at 14.0V
- Must **disable motor enable** (set EN pin LOW) when voltage drops below threshold.
- Must **leave indicator/warning LED active** so the user knows why the motors stopped.
- Must **NOT auto-recover** — require explicit user action (e.g., button press) to re-enable motors after low-voltage shutdown.
- Check that the voltage divider ratio in code matches the actual resistor values.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Battery voltage monitoring is missing

  Add the following hardware and software:
  Hardware:
  - Voltage divider: R1 (top) = 10kΩ, R2 (bottom) = 3.3kΩ → ratio ≈ 4.03
    (for 3S: 12.6V max / 4.03 ≈ 3.13V → safe for 3.3V ADC)
  - Connect divider output to an ADC-capable pin
  - Add 100nF capacitor at ADC pin for noise filtering

  Software:
  - Configure ADC channel in continuous or triggered mode
  - Read and filter every 100ms in main loop or a low-priority task
  - When voltage < threshold: MotorDisable() IMMEDIATELY
  - Blink LED pattern to indicate low battery (e.g., 3 fast blinks, pause, repeat)
```

---

## Dimension 2: Sensors & Attitude Algorithms (Runaway / Drift Risk)

Continuing from MPU6050 issues — AI's mathematical reasoning often has subtle flaws.

---

### Item 6: Yaw Reset (ResetYaw)

| Aspect | Detail |
|--------|--------|
| **Consequence** | Wrong heading reference → all tracking/turning is off |
| **AI Common Mistake** | Only clears the flag variable, doesn't zero internal static variables |

**What to check:**

```c
// ❌ FAIL — flag-only reset, internal state survives
void ResetYaw(void) {
    reset_yaw_flag = 1;   // Sets a flag, but does nothing to the accumulated value
}

// ✅ CORRECT — comprehensive yaw reset
void ResetYaw(void) {
    g_fYaw = 0.0f;              // Zero the yaw accumulator
    static_yaw_offset = 0.0f;   // Zero any static offset
    reset_yaw_flag = 1;
    // Clear DMP FIFO buffer to discard stale data
    mpu_dmp_fifo_reset();       // or: while(mpu_dmp_get_fifo_count()) { mpu_dmp_read_fifo(); }
}
```

**Verification rules:**
- Search for all `static` variables related to yaw/heading and verify each is explicitly set to zero.
- Search for DMP FIFO flush — stale quaternion data in the buffer will immediately overwrite the zeroed yaw.
- If using software complementary filter or Mahony/Madgwick filter: also reset the filter's internal state (quaternion to identity, gyro bias estimates to zero).
- Verify that `ResetYaw()` is called BEFORE the attitude update loop resumes, not after.

**Corrective action if FAIL:**
```c
// Complete yaw reset pattern
void ResetYaw(void) {
    // 1. Zero all accumulators
    g_fYaw = 0.0f;
    yaw_bias = 0.0f;
    last_yaw_angle = 0.0f;

    // 2. Flush DMP FIFO
    uint8_t dummy[28];
    short fifo_count;
    while ((fifo_count = mpu_dmp_get_fifo_count()) > 0) {
        if (fifo_count > 28) fifo_count = 28;
        mpu_dmp_read_fifo(NULL, NULL, NULL, dummy, NULL);
    }

    // 3. Reset flag AFTER clearing
    reset_yaw_flag = 0;
}
```

---

### Item 7: Integral Windup (PID Windup)

| Aspect | Detail |
|--------|--------|
| **Consequence** | Overshoot → violent reverse correction → fishtailing → crash |
| **AI Common Mistake** | PID integral term accumulates without bound |

**What to check:**

```c
// ❌ CRITICAL FAIL — unbounded integral accumulation
I_Term += Ki * Error;   // Accumulates forever during saturation!
Output = Kp * Error + I_Term + Kd * dError;

// ✅ CORRECT — integral separation + output clamping before accumulation
#define INTEGRAL_SEPARATION_THRESHOLD  100   // Stop I accumulation when error > this

float PID_Compute(PID_TypeDef *pid, float setpoint, float measurement) {
    float error = setpoint - measurement;

    // Proportional term
    pid->P_Term = pid->Kp * error;

    // Integral term with ANTI-WINDUP
    if (fabsf(error) < INTEGRAL_SEPARATION_THRESHOLD) {
        pid->I_Term += pid->Ki * error * pid->dt;  // Only accumulate when error is small
    }
    // Clamp integral term BEFORE using in output
    pid->I_Term = constrain(pid->I_Term, -pid->I_Max, pid->I_Max);

    // Derivative term
    pid->D_Term = pid->Kd * (error - pid->PrevError) / pid->dt;

    // Sum and clamp final output
    float output = pid->P_Term + pid->I_Term + pid->D_Term;
    output = constrain(output, -pid->OutMax, pid->OutMax);

    pid->PrevError = error;
    return output;
}
```

**Verification rules:**
- **Integral separation**: I term must stop accumulating when `|error| > threshold`.
- **I-term clamping**: I term itself must be clamped (`I_Term = constrain(I_Term, -I_Max, I_Max)`) before being added to the output sum.
- **Output clamping**: Final output must be clamped after summing P + I + D (this is item 2's domain, but verify here that clamping happens AFTER the I-term is added, not before).
- **Back-calculation anti-windup** (advanced): if using this method instead, verify that `(clamped_output - raw_output) * Kc` is subtracted from the integrator.
- **Reset on mode change**: if the controller switches between modes (e.g., manual → auto), I-term must be zeroed.

**Corrective action if FAIL:**
```c
// Robust PID with full anti-windup
typedef struct {
    float Kp, Ki, Kd;
    float I_Max;       // Integral saturation limit
    float OutMax;      // Output saturation limit
    float dt;          // Sample time

    float PrevError;
    float I_Term;
    float P_Term, D_Term;  // For debugging/monitoring
} PID_TypeDef;

#define INTEGRAL_SEP_THRESH  100.0f

float PID_Update(PID_TypeDef *pid, float setpoint, float measurement) {
    float error = setpoint - measurement;

    // P term
    pid->P_Term = pid->Kp * error;

    // I term — integral separation
    if (fabsf(error) < INTEGRAL_SEP_THRESH) {
        pid->I_Term += pid->Ki * error * pid->dt;
    }
    // Clamp I before output
    if (pid->I_Term > pid->I_Max)  pid->I_Term = pid->I_Max;
    if (pid->I_Term < -pid->I_Max) pid->I_Term = -pid->I_Max;

    // D term
    pid->D_Term = pid->Kd * (error - pid->PrevError) / pid->dt;
    pid->PrevError = error;

    // Sum and clamp
    float out = pid->P_Term + pid->I_Term + pid->D_Term;
    if (out > pid->OutMax)  out = pid->OutMax;
    if (out < -pid->OutMax) out = -pid->OutMax;

    return out;
}
```

---

### Item 8: Sensor Timestamp / Update Rate

| Aspect | Detail |
|--------|--------|
| **Consequence** | Stale attitude data → lag → high-speed rollover / crash |
| **AI Common Mistake** | Uses `delay(10)` or `HAL_Delay()` to control update period |

**What to check:**

```c
// ❌ CRITICAL FAIL — delay-based timing in attitude update loop
while (1) {
    MPU6050_Read_Data();
    Attitude_Update();
    Motor_Control();
    HAL_Delay(10);   // Blocks everything! Sensor data is stale. PWM flow interrupted.
}

// ✅ CORRECT — hardware timer interrupt drives the update rate
// In main.c or sensor.c:
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM7) {   // Dedicated sensor timer at 1 kHz
        MPU6050_Read_Data_IT();     // Read sensor in ISR
        Attitude_Update_IT();       // Update attitude filter
        sensor_timestamp++;         // Timestamp from hardware counter
    }
}

// Sensor read function: use a timestamp, never delay()
void MPU6050_Read_Data_IT(void) {
    // Read I2C data via DMA or fast register read
    // NO delay() or HAL_Delay() anywhere in this path
    // The hardware timer guarantees the 1 kHz cadence
}
```

**Verification rules:**
- Search for `delay()`, `HAL_Delay()`, `osDelay()`, `vTaskDelay()` inside attitude update functions — NONE allowed.
- Search for `while(xxx)` spin loops waiting for I2C — these are also blocking delays. Replace with DMA + interrupt completion.
- Verify that attitude update runs from a **hardware timer ISR** at a fixed rate (typically 200 Hz to 1 kHz for MPU6050).
- Check that the timer's actual period matches the declared rate (ARR + PSC calculation).
- For RTOS-based designs: sensor task must be highest priority and use timer-triggered notification, not `vTaskDelay()` for timing.

**Corrective action if FAIL:**
```c
// Hardware timer-based sensor read pattern

// 1. Configure a timer for the desired rate (e.g., TIM7 at 1 kHz)
//    TIM7 base clock = 84 MHz (APB1 on STM32F4)
//    PSC = 84 - 1 = 83  →  84 MHz / 84 = 1 MHz
//    ARR = 1000 - 1 = 999  →  1 MHz / 1000 = 1 kHz
#define SENSOR_TIMER           TIM7
#define SENSOR_UPDATE_RATE_HZ  1000

// 2. In the timer ISR, read sensor and update filter
void TIM7_IRQHandler(void) {
    if (__HAL_TIM_GET_FLAG(&htim7, TIM_FLAG_UPDATE)) {
        __HAL_TIM_CLEAR_FLAG(&htim7, TIM_FLAG_UPDATE);
        // Minimal work in ISR: read data, set flag for main loop
        MPU6050_Read_DMP_FIFO();  // Fast, non-blocking
        sensor_data_ready = 1;
    }
}

// 3. In main loop, process when data is ready
while (1) {
    if (sensor_data_ready) {
        sensor_data_ready = 0;
        Attitude_Compute();     // Heavy computation in main loop, not ISR
        Motor_Control();
    }
    HAL_IWDG_Refresh();         // Pet watchdog
}
```

---

### Item 9: Encoder Edge Miscounting

| Aspect | Detail |
|--------|--------|
| **Consequence** | Sudden speed measurement spikes → PID oscillation → driver destruction |
| **AI Common Mistake** | Single-edge triggering (only rising edge), no software filtering |

**What to check:**

```c
// ❌ FAIL — single-edge counting, no filtering
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == ENCODER_A_PIN) {
        encoder_count++;   // Only counts rising edges → half resolution
    }
}

// ✅ CORRECT — timer encoder mode with software moving average filter
// In MX_TIMx_Init():
static void MX_TIM2_Encoder_Init(void) {
    TIM_Encoder_InitTypeDef sEncoderConfig = {0};
    htim2.Instance = TIM2;
    htim2.Init.Prescaler = 0;
    htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
    htim2.Init.Period = 0xFFFF;  // 16-bit max
    htim2.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;

    sEncoderConfig.EncoderMode = TIM_ENCODERMODE_TI12;  // BOTH edges on BOTH channels = 4x resolution
    sEncoderConfig.IC1Polarity = TIM_ICPOLARITY_RISING;
    sEncoderConfig.IC1Selection = TIM_ICSELECTION_DIRECTTI;
    sEncoderConfig.IC1Prescaler = TIM_ICPSC_DIV1;
    sEncoderConfig.IC1Filter = 0x0F;  // Digital filter: 8 samples for debounce
    // ... same for IC2
    HAL_TIM_Encoder_Init(&htim2, &sEncoderConfig);
}

// Software moving average filter for speed calculation
#define SPEED_FILTER_WINDOW  8   // 5-10 values
static float speed_buffer[SPEED_FILTER_WINDOW] = {0};
static uint8_t speed_buffer_idx = 0;

float GetFilteredSpeed(void) {
    int16_t raw_count = (int16_t)__HAL_TIM_GET_COUNTER(&htim2);
    __HAL_TIM_SET_COUNTER(&htim2, 0);  // Reset counter after reading

    float raw_speed = (float)raw_count * SPEED_SCALE_FACTOR;

    // Moving average
    speed_buffer[speed_buffer_idx] = raw_speed;
    speed_buffer_idx = (speed_buffer_idx + 1) % SPEED_FILTER_WINDOW;

    float sum = 0;
    for (int i = 0; i < SPEED_FILTER_WINDOW; i++) {
        sum += speed_buffer[i];
    }
    return sum / SPEED_FILTER_WINDOW;
}
```

**Verification rules:**
- Encoder mode must be `TIM_ENCODERMODE_TI12` (both channels, both edges → **4× resolution**). Never `TI1` only.
- Timer input filter (IC1Filter / IC2Filter) must be set to at least `0x03` for basic debouncing. Values `0x08`–`0x0F` are better for noisy motor environments.
- Software: a **moving average filter** with window size **5 to 10** samples must be applied to the speed calculation.
- Check for counter overflow handling — the encoder timer wraps around at 0xFFFF (16-bit). The speed calculation must handle this wrap correctly (use signed 16-bit arithmetic: `int16_t delta = (int16_t)(current - previous)`).
- If using GPIO EXTI for encoder (not recommended but sometimes seen): must use `GPIO_MODE_IT_RISING_FALLING` (both edges), not just `GPIO_MODE_IT_RISING`.

**Corrective action if FAIL:**
```c
// Recommended: use STM32 hardware timer encoder mode
// Pin mapping for TIM2 encoder mode (common):
//   TIM2_CH1 → PA0  (AF1)
//   TIM2_CH2 → PA1  (AF1)

// If encoder EXTI is unavoidable (NOT recommended):
// Configure both A and B pins as GPIO_MODE_IT_RISING_FALLING
// Read B state in A's ISR (and vice versa) to determine direction:
void Encoder_ISR(void) {
    static int8_t enc_table[16] = {0,1,-1,0, -1,0,0,1, 1,0,0,-1, 0,-1,1,0};
    static uint8_t enc_state = 0;
    enc_state = ((enc_state << 2) | ((GPIOA->IDR >> 0) & 0x03)) & 0x0F;
    encoder_count += enc_table[enc_state];
}
```

---

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

## Dimension 4: [RESERVED — Items 14–17]

> **These 4 items are intentionally left as placeholders for future expansion.**
> Skip items 14–17 during review. Do not fabricate checks for them.
> When the user provides content for Dimension 4, update this section.

| Item | Status | Planned Topic Area |
|------|--------|-------------------|
| 14 | ⏳ RESERVED | TBD |
| 15 | ⏳ RESERVED | TBD |
| 16 | ⏳ RESERVED | TBD |
| 17 | ⏳ RESERVED | TBD |

---

## Dimension 5: Power & Electrical Integrity (AI's Physics Blind Spot)

AI treats motor stop as a software command — it doesn't know that a spinning motor is a generator.

---

### Item 18: Back-EMF Protection on Emergency Stop

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Hard stop from full speed → motor becomes generator → reverse current surges back into driver IC → driver MOSFET destroyed |
| **AI Common Mistake** | `Motor_Stop()` instantly sets PWM duty to 0%, then disables EN pin in the same cycle |

**What to check:**

```c
// ❌ CRITICAL FAIL — instant hard stop from 100% to 0%
void Motor_Stop(void) {
    Motor_Left_Set(0);     // PWM → 0% instantly
    Motor_Right_Set(0);
    HAL_GPIO_WritePin(EN_PORT, EN_PIN, GPIO_PIN_RESET);  // Driver disabled
    // Back-EMF current has nowhere to go except through the driver body diodes → BOOM
}

// ✅ CORRECT — soft braking ramp + coast stop
void Motor_Stop_Safe(void) {
    // Step 1: Ramp PWM down to 0 over 50 ms
    int16_t current_left = Motor_GetCurrentDutyLeft();
    int16_t current_right = Motor_GetCurrentDutyRight();
    int16_t steps = 10;  // 10 steps over 50ms = 5ms per step

    for (int i = steps; i >= 0; i--) {
        Motor_Left_Set(current_left * i / steps);
        Motor_Right_Set(current_right * i / steps);
        Delay_ms(5);  // 50ms total ramp-down (acceptable ONLY during emergency stop)
    }

    // Step 2: Coast stop — set both IN pins LOW (not brake!)
    // TB6612: IN1=LOW, IN2=LOW → coast (motor free-spins, EMF dissipates in windings)
    GPIO_ResetBits(DIR_PORT, IN1_PIN_LEFT | IN2_PIN_LEFT | IN1_PIN_RIGHT | IN2_PIN_RIGHT);

    // Step 3: Disable driver only after EMF has dissipated
    Delay_ms(10);
    HAL_GPIO_WritePin(EN_PORT, EN_PIN, GPIO_PIN_RESET);
}
```

**Verification rules:**
- Search for any instant transition from high duty to 0%. In the motor control path, the stop function must have a **ramp-down loop** (not a single register write).
- **Coast stop** (both IN pins LOW for TB6612/DRV8833) is preferred over **brake stop** (both IN pins HIGH) for emergency stops. Brake mode shorts the motor windings, converting kinetic energy to heat in the driver — still safer than back-EMF destruction, but coast stop is gentlest.
- Ramp-down duration: **30–100 ms** for small N20 motors, **100–300 ms** for larger 775/R550 motors.
- If the driver IC has a **BRAKE pin** (e.g., DRV8833's nSLEEP), verify that brake is NOT asserted during the ramp-down phase.
- **Normal speed changes** (not emergency stop): change PWM gradually (max 10% change per control cycle), not instantly from 50% to 20%.

**Corrective action if FAIL:**
```c
// Safe motor stop with soft ramp
#define EMERGENCY_RAMP_STEPS   10
#define EMERGENCY_RAMP_MS      50

static void Motor_CoastStop(void) {
    int16_t l = Motor_GetCurrentDutyLeft();
    int16_t r = Motor_GetCurrentDutyRight();
    float step_l = (float)l / EMERGENCY_RAMP_STEPS;
    float step_r = (float)r / EMERGENCY_RAMP_STEPS;

    for (int i = EMERGENCY_RAMP_STEPS - 1; i >= 0; i--) {
        Motor_Left_Set((int16_t)(step_l * i));
        Motor_Right_Set((int16_t)(step_r * i));
        Delay_ms(EMERGENCY_RAMP_MS / EMERGENCY_RAMP_STEPS);
    }

    // Coast stop: all direction pins LOW
    GPIO_ResetBits(MOTOR_DIR_PORT,
        MOTOR_IN1_LEFT | MOTOR_IN2_LEFT |
        MOTOR_IN1_RIGHT | MOTOR_IN2_RIGHT);

    // Wait for EMF dissipation, then disable driver
    Delay_ms(10);
    GPIO_ResetBits(MOTOR_EN_PORT, MOTOR_EN_PIN);
}
```

---

### Item 19: Ground Bounce & I2C Communication Integrity

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Motor PWM noise couples into sensor I2C bus → MPU6050 readings jump to garbage or I2C hangs → attitude NaN → car loses control |
| **AI Common Mistake** | When I2C fails, AI adds `Delay_ms(100)` and retries — blocking the main loop |

**What to check:**

```c
// ❌ FAIL — blocking retry on I2C failure, no hardware noise mitigation
uint8_t MPU6050_Read(uint8_t reg) {
    while (HAL_I2C_Master_Transmit(&hi2c1, MPU_ADDR, &reg, 1, 1000) != HAL_OK) {
        Delay_ms(50);  // Blocks main loop! Motors keep running with stale data.
    }
    // ...
}

// ✅ CORRECT — timeout-based I2C with hardware reset capability
#define I2C_TIMEOUT_MS      10     // 10ms timeout, NOT 1000ms
#define I2C_MAX_RETRIES      3

uint8_t MPU6050_Read_Safe(uint8_t reg, uint8_t *data, uint8_t len) {
    uint8_t retry;
    for (retry = 0; retry < I2C_MAX_RETRIES; retry++) {
        // Software I2C with explicit timeout (your existing code pattern)
        I2C_Start();
        I2C_WriteByte(MPU6050_ADDR);
        if (I2C_WaitAck_Timeout(100) == 0) {   // ACK OK
            I2C_WriteByte(reg);
            I2C_WaitAck_Timeout(100);
            // Read data...
            I2C_Stop();
            return 1;  // Success
        }
        I2C_Stop();
        // Bus recovery: 9 clock pulses to release stuck SDA
        I2C_Bus_Recovery();
    }
    // All retries exhausted → hardware reset MPU6050 power
    MPU6050_PowerCycle();  // Toggle MPU6050 VCC via GPIO-controlled P-MOSFET
    return 0;  // Failure — caller must handle gracefully
}
```

**Verification rules:**

**Hardware (cannot verify from code alone — flag as NEEDS MANUAL VERIFICATION):**
- Motor power and MCU/sensor power must be on separate power traces/planes, joining only at the battery connector.
- MPU6050 VCC pin must have **100 nF ceramic + 10 µF electrolytic** capacitors placed within 5 mm of the pin.
- I2C SCL/SDA lines: 4.7kΩ pull-up resistors to 3.3V (not to 5V). If bus capacitance is high (>100 pF), drop to 2.2kΩ.
- Motor driver IC must have a **bulk capacitor** (100–470 µF electrolytic) across its power pins, close to the IC.

**Software:**
- I2C read/write functions must have a **hard cap on total blocking time**: max **10 ms** per transaction (not HAL's default 1000 ms).
- After N consecutive I2C failures (N ≤ 5), the code must either: (a) power-cycle the sensor VCC via a GPIO-controlled MOSFET, or (b) enter a safe state (motors off, LED alarm).
- I2C bus recovery routine (9 SCL pulses to unstick a held-low SDA) must be present — your existing code in `mpu6050.c:164-170` already does this correctly.
- During I2C retries, the **motor control loop must continue running** with the last good sensor data — never block the main loop on I2C.

**Corrective action if FAIL:**
```
⚠️ HARDWARE CHECK — Ground bounce mitigation

  Verify with oscilloscope:
  1. Probe MPU6050 VCC pin during full-speed motor PWM. Ripple must be < 50mV p-p.
     If > 50mV: add 100nF + 10µF at MPU6050 VCC, check ground return path.
  2. Probe I2C SCL line during motor PWM. No missing pulses or glitches allowed.
     If glitches present: reduce pull-up resistor (2.2kΩ), separate motor/sensor ground returns.
  3. Probe motor driver VM pin: voltage sag during PWM on-cycle must be < 500mV.
     If > 500mV: add bulk capacitor (220µF+) at driver VM pin.
```

---

### Item 20: Servo Stall Current Protection

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Steering servo held at mechanical limit → current climbs to stall current (1A+) → servo motor windings overheat and burn out in seconds |
| **AI Common Mistake** | Continuously outputs the extreme PWM value (500/2500 µs) without time limit |
| **Trigger Condition** | **Check ONLY if the system includes one or more servo motors (steering servo, pan/tilt servo, gripper servo). Skip this item if no servos are present.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — servo held at extreme position indefinitely
void Servo_SetAngle(uint8_t angle_deg) {
    uint16_t pulse = SERVO_MIN_PULSE + (uint32_t)angle_deg * (SERVO_MAX_PULSE - SERVO_MIN_PULSE) / 180;
    TIM_SetCompare1(SERVO_TIM, pulse);  // No time limit, no position guard
}

// ✅ CORRECT — position limit + time guard
#define SERVO_MIN_ANGLE      10    // Mechanical limit, not 0°
#define SERVO_MAX_ANGLE     170    // Mechanical limit, not 180°
#define SERVO_EXTREME_TIMEOUT_MS  2000  // 2 seconds max at any extreme

static uint32_t servo_extreme_timer = 0;
static uint8_t  servo_extreme_active = 0;

void Servo_SetAngle_Safe(uint8_t target_angle) {
    uint8_t clamped = target_angle;
    if (clamped < SERVO_MIN_ANGLE) clamped = SERVO_MIN_ANGLE;
    if (clamped > SERVO_MAX_ANGLE) clamped = SERVO_MAX_ANGLE;

    // Check if at extreme position
    if (clamped <= SERVO_MIN_ANGLE + 5 || clamped >= SERVO_MAX_ANGLE - 5) {
        if (!servo_extreme_active) {
            servo_extreme_active = 1;
            servo_extreme_timer = GetTick();
        } else if (GetTick() - servo_extreme_timer > SERVO_EXTREME_TIMEOUT_MS) {
            // Held at extreme too long → force to center + alarm
            clamped = 90;  // Center position
            servo_extreme_active = 0;
            LED_Alarm_Blink();  // Alert user: servo was forced back
        }
    } else {
        servo_extreme_active = 0;
    }

    uint16_t pulse = SERVO_MIN_PULSE + (uint32_t)clamped * (SERVO_MAX_PULSE - SERVO_MIN_PULSE) / 180;
    TIM_SetCompare1(SERVO_TIM, pulse);
}
```

**Verification rules:**
- Servo angle output must be limited to a **safe mechanical range** (typically 10°–170°, not 0°–180°). The extreme ends are where the mechanical stop engages and current skyrockets.
- Must have a **time guard**: if the servo is commanded to stay at an extreme position (>±85% of range) for more than **2 seconds continuously**, force it back to center position and raise an alarm.
- Servo power should be supplied by a **separate regulator**, not the MCU's 3.3V LDO. The MCU LDO typically cannot supply 1A+ servo stall current.
- **Power-on initialization**: servo must start at center (90°) and move to target gradually, not snap to an arbitrary angle at boot.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Servo stall protection missing

  1. Add SERVO_MIN_ANGLE (10°) and SERVO_MAX_ANGLE (170°) limits in config.
  2. Add a 2-second timeout: if servo_angle is within 5° of either limit
     for > 2000ms continuously, override servo to 90° center and set alarm flag.
  3. Hardware: verify servo power rail uses a dedicated regulator (e.g., 5V/3A),
     NOT the 3.3V MCU regulator. MCU VCC from servo regulator ≈ instant MCU death on stall.
```

---

## Dimension 6: Communication Bus Deadlock & Timeout (The Silent Freeze)

These bugs don't crash the MCU — they freeze it silently. Without a watchdog, the car keeps running on the last valid PWM command.

---

### Item 21: I2C/SPI Bus Hang Recovery

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | I2C slave (MPU6050) holds SDA low → `HAL_I2C_Master_Transmit()` waits forever (default timeout = HAL_MAX_DELAY = 0xFFFFFFFF) → main loop dead → motors keep running with no sensor updates → car crashes with no obstacle avoidance |
| **AI Common Mistake** | Uses HAL default infinite timeout, or soft-resets the entire MCU on I2C hang |

**What to check:**

```c
// ❌ CRITICAL FAIL — blocking I2C with no or huge timeout
HAL_I2C_Master_Transmit(&hi2c1, MPU_ADDR, buf, len, 1000);  // 1000ms! Blocked for 1 second.
// or worse:
HAL_I2C_Master_Transmit(&hi2c1, MPU_ADDR, buf, len, HAL_MAX_DELAY);  // INFINITE wait

// ❌ WRONG RECOVERY — reset entire MCU on I2C hang
NVIC_SystemReset();  // MCU reboots → GPIOs float → motors may kick!

// ✅ CORRECT — 10ms timeout + hardware power-cycle the slave device
#define I2C_TIMEOUT_MS      10    // Maximum 10ms per I2C transaction
#define I2C_HANG_RECOVERY_COUNT  5  // After 5 consecutive hangs, power-cycle slave

// Software I2C timeout pattern (your existing code should extend this):
static uint8_t I2C_WaitAck_Timeout(uint32_t timeout_us) {
    uint32_t start = TIM_GetCounter(I2C_TIMEOUT_TIMER);
    I2C_SDA_H(); I2C_Delay();
    I2C_SCL_H(); I2C_Delay();
    while (I2C_SDA_READ()) {
        if ((uint32_t)(TIM_GetCounter(I2C_TIMEOUT_TIMER) - start) > timeout_us) {
            I2C_SCL_L();
            return 1;  // ACK timeout — slave not responding
        }
    }
    I2C_SCL_L();
    return 0;  // ACK received
}

// Power-cycle slave via GPIO-controlled P-MOSFET:
static void MPU6050_PowerCycle(void) {
    GPIO_ResetBits(MPU_PWR_PORT, MPU_PWR_PIN);  // P-MOSFET gate LOW = OFF
    Delay_ms(50);  // Let capacitors discharge
    GPIO_SetBits(MPU_PWR_PORT, MPU_PWR_PIN);    // P-MOSFET gate HIGH = ON
    Delay_ms(100);  // Wait for MPU6050 startup (including PLL lock)
    MPU6050_Init();  // Re-initialize registers
}
```

**Verification rules:**
- Every I2C transaction must have a timeout ≤ **10 ms**. This applies to both HAL and software I2C.
- After N consecutive I2C failures (N ≤ 5), the slave device must be **power-cycled** via GPIO. Soft reset (writing 0x80 to PWR_MGMT_1) is insufficient — a stuck-low SDA can't be cleared by I2C commands.
- During I2C failure and recovery, the **motor control must NOT stall**. Use last-known-good sensor values (marked with a `stale` flag) to decide whether to continue or stop.
- Slave power control requires **hardware support**: the MPU6050 VCC must be switchable via a GPIO-controlled P-MOSFET or load switch. If the PCB does not have this, flag it as ⚠️ NEEDS HARDWARE MODIFICATION.
- SP: same principle — use `HAL_SPI_TransmitReceive()` with a finite timeout, and provide a GPIO-based CS (chip select) reset sequence on timeout.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — I2C timeout + hardware slave reset

  Your existing code (mpu6050.c) already has software I2C + 3 retries + bus recovery
  (9 SCL pulses). This is a good baseline. Three things to add:

  1. Hardware: Add a P-MOSFET on MPU6050 VCC, gate controlled by a free GPIO.
     This lets you power-cycle the MPU6050 without rebooting the MCU.

  2. Software: On 3 consecutive I2C failures, call MPU6050_PowerCycle()
     instead of just retrying.

  3. During I2C recovery window (~150ms), set a flag 'sensors_stale = 1'.
     Main loop reads this flag and either slows down or stops motors.
```

---

### Item 22: UART Receive Overflow Protection

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Host/Bluetooth floods UART RX → Overrun Error (ORE) flag set → UART stops receiving → no more remote commands → car keeps executing last command forever |
| **AI Common Mistake** | Parses commands with `strcmp()` inside the UART RX ISR, or uses a single-byte receive buffer with no overflow handling |

**What to check:**

```c
// ❌ CRITICAL FAIL — UART ISR does string parsing, no overflow protection
void USART1_IRQHandler(void) {
    uint8_t ch = USART_ReceiveData(USART1);
    rx_buf[rx_idx++] = ch;
    if (ch == '\n') {
        rx_buf[rx_idx] = '\0';
        if (strcmp(rx_buf, "FORWARD\n") == 0) { ... }  // strcmp in ISR!
        rx_idx = 0;
    }
    // No ORE check — if overrun occurs, USART stops receiving silently
}

// ✅ CORRECT — DMA circular buffer + IDLE line detection
#define UART_RX_BUF_SIZE  64
static volatile uint8_t uart_rx_dma_buf[UART_RX_BUF_SIZE];
static volatile uint8_t uart_rx_data_ready = 0;

// Init: DMA circular mode
void UART_DMA_Init(void) {
    DMA_InitTypeDef DMA_InitStructure;
    DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&USART1->DR;
    DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)uart_rx_dma_buf;
    DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;
    DMA_InitStructure.DMA_BufferSize = UART_RX_BUF_SIZE;
    DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;  // Circular buffer
    // ... other DMA config ...
    DMA_Init(DMA1_Channel5, &DMA_InitStructure);

    USART_DMACmd(USART1, USART_DMAReq_Rx, ENABLE);
    DMA_Cmd(DMA1_Channel5, ENABLE);

    // Enable IDLE line interrupt
    USART_ITConfig(USART1, USART_IT_IDLE, ENABLE);
}

// IDLE ISR: fires when RX line goes idle after a frame
void USART1_IRQHandler(void) {
    if (USART_GetITStatus(USART1, USART_IT_IDLE) != RESET) {
        USART_ReceiveData(USART1);  // Clear IDLE flag (must read DR + SR)
        DMA_Cmd(DMA1_Channel5, DISABLE);
        uint16_t received_len = UART_RX_BUF_SIZE - DMA_GetCurrDataCounter(DMA1_Channel5);
        DMA_SetCurrDataCounter(DMA1_Channel5, UART_RX_BUF_SIZE);
        DMA_Cmd(DMA1_Channel5, ENABLE);
        uart_rx_data_ready = 1;    // Signal main loop to process
    }
    // Handle Overrun Error
    if (USART_GetFlagStatus(USART1, USART_FLAG_ORE) != RESET) {
        USART_ReceiveData(USART1);  // Read DR to clear ORE
    }
}
```

**Verification rules:**
- Search for `USART_IT_ORE` or `USART_FLAG_ORE` — overrun error must be explicitly handled. Reading the DR register clears ORE.
- UART ISR must NOT call `strcmp()`, `strstr()`, `sscanf()`, `printf()`, or any blocking function. String parsing belongs in the main loop.
- Must use either: (a) DMA circular buffer + IDLE line interrupt, or (b) a ring buffer with head/tail pointers and explicit overflow detection.
- Single-byte interrupt receive is acceptable for low-bandwidth remote control (e.g., 9600 baud joystick), but must still check for ORE and protect against buffer overflow.
- If using an RTOS, the UART ISR should post to a queue/semaphore, not do the parsing itself.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — UART overflow protection

  If UART is currently unused in your project: skip this item.
  If UART is used for Bluetooth/WiFi remote control:

  1. Replace single-byte RX ISR with DMA circular buffer + IDLE interrupt.
  2. Move all command parsing out of ISR into main loop.
  3. Add ORE flag handling in the ISR.
  4. If no DMA channels are free, implement a software ring buffer with:
     - rx_head (ISR writes), rx_tail (main loop reads)
     - If (rx_head + 1) % BUF_SIZE == rx_tail → drop byte (overflow)
     - Check ORE flag each ISR entry
```

---

## Dimension 7: Control Strategy & Abnormal Physical States (Prevent Mysterious Oscillations)

AI treats controllers as pure math — it doesn't know the difference between "correcting a small error" and "fighting physics until something breaks."

---

### Item 23: Balancing Robot Tilt Emergency Shutdown

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Balance bot tips past recovery angle → PID outputs maximum PWM trying to catch it → bot accelerates into the ground/people/wall at full motor power |
| **AI Common Mistake** | PID output continues to fight even when tilt angle exceeds the point of no return |
| **Trigger Condition** | **Check ONLY if the system is a two-wheel self-balancing robot (inverted pendulum). Skip this item for conventional 4-wheel or differential-drive vehicles.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — no tilt shutdown angle
float pid_output = PID_Compute(&balance_pid, 0.0f, pitch_angle);  // pitch_angle = 45°!
Motor_Drive(pid_output, heading_angle);  // Still driving at full power!

// ✅ CORRECT — safety tilt threshold
#define TILT_SAFETY_ANGLE_DEG   30.0f    // Absolute pitch/roll beyond which motors cut off

void Balance_Loop(void) {
    float pitch = g_sensors.pitch;
    float roll  = g_sensors.roll;

    // Emergency check: if tilt exceeds safety angle, IMMEDIATE motor disable
    if (fabsf(pitch) > TILT_SAFETY_ANGLE_DEG || fabsf(roll) > TILT_SAFETY_ANGLE_DEG) {
        Motor_Stop_Safe();          // Soft ramp-down (Item 18)

        // Disable motor enable pin — permanent until user manually resets
        HAL_GPIO_WritePin(MOTOR_EN_PORT, MOTOR_EN_PIN, GPIO_PIN_RESET);

        // Alarm: LED fast blink forever
        while (1) {
            LED_Toggle();
            Delay_ms(100);
            IWDG_ReloadCounter();  // Feed watchdog to prevent spurious MCU reset
        }
    }

    // Normal balance control
    float balance_output = PID_Compute(&balance_pid, 0.0f, pitch);
    Motor_Drive(balance_output, heading_correction);
}
```

**Verification rules:**
- A hard tilt threshold must be defined: typically **30°** for hobby-class balance bots. Above this angle, recovery is physically impossible — the motors lack the torque to overcome gravity × lever arm.
- When tilt exceeds threshold, the response must be: **immediate motor disable (EN pin LOW)**, not "set speed to 0" — the EN pin cut is the only guarantee.
- Must NOT auto-recover from tilt shutdown. The bot could be caught on an obstacle; auto-recovery would slam it into the ground. Require a physical button press or power cycle.
- The tilt safety check must run at the **same rate as the balance control loop** (typically 200 Hz–1 kHz). A 10 Hz check is too slow — the bot can fall past 30° in < 50 ms.
- Even during shutdown, the watchdog must be fed (if enabled) to prevent an uncontrolled MCU reset during the alarm state.

**Corrective action if FAIL:**
```c
#define TILT_SAFETY_DEG    30.0f

// Call in every balance control iteration:
static inline void Balance_SafetyCheck(float pitch, float roll) {
    if (fabsf(pitch) > TILT_SAFETY_DEG || fabsf(roll) > TILT_SAFETY_DEG) {
        // CRITICAL: cut motor enable — the ONLY guarantee motors stop
        Motor_Disable_Immediate();
        // Alarm loop — never returns, requires power cycle
        while (1) {
            LED_Toggle();
            for (volatile uint32_t i = 0; i < 500000; i++) __NOP();
            IWDG_ReloadCounter();
        }
    }
}
```

---

### Item 24: Speed Feedforward for Steering/Tracking

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Sharp turn at full speed → inner wheel reverses or locks → drivetrain stripped, gear stripped, car spins out of control |
| **AI Common Mistake** | Tracking PID and speed PID run independently — tracking PID demands a hard turn while speed PID demands full speed |

**What to check:**

```c
// ❌ CRITICAL FAIL — independent speed and steering
float speed = SpeedPID_Compute(&speed_pid, target_speed, current_speed);
float steer = LineTrackingPID_Compute(&line_pid, line_position, CENTER);
Motor_Drive(speed, steer);  // speed = 80, steer = 60 → inner wheel = 20, outer = 140!

// ✅ CORRECT — speed feedforward coupled to steering
#define STEER_COUPLING_COEFF    0.008f   // Tune: how aggressively speed drops with steer

void Motor_Drive_Coupled(float linear_speed, float angular_steer) {
    // Speed feedforward: faster you steer, slower you go
    float steer_factor = 1.0f - (fabsf(angular_steer) * STEER_COUPLING_COEFF);
    if (steer_factor < 0.2f) steer_factor = 0.2f;  // Floor: never below 20% speed in turns

    float coupled_speed = linear_speed * steer_factor;

    // Now compute wheel speeds using the REDUCED linear speed
    int16_t left  = (int16_t)(coupled_speed - angular_steer);
    int16_t right = (int16_t)(coupled_speed + angular_steer);

    // Standard duty limiting (Item 2)
    if (left  > MAX_DUTY)  left  = MAX_DUTY;
    if (left  < -MAX_DUTY) left  = -MAX_DUTY;
    if (right > MAX_DUTY)  right = MAX_DUTY;
    if (right < -MAX_DUTY) right = -MAX_DUTY;

    Motor_Left_Set(left);
    Motor_Right_Set(right);
}
```

**Verification rules:**
- For every degree of steering angle increase (or every unit of line position error), the base speed must be reduced. The coupling coefficient must be tunable (macro or config variable).
- At maximum steering angle, base speed must drop to ≤ **20%** of max speed. The exact floor depends on the vehicle's wheelbase and center of gravity.
- Check the differential math: with `speed=80, steer=60`, inner wheel = 80-60=20 (25% of outer). This causes a near-pivot turn — high mechanical stress. With coupling, `speed=80×0.52=42`, inner=42-60=-18 (reverse!) — still bad. The coupling coefficient must be tuned so that the **inner wheel never reverses** during normal tracking turns. A reasonable starting point: `STEER_COUPLING_COEFF = 0.6 / MAX_ANGULAR`.
- **Visual/line tracking**: this is where coupling matters most. When the line sensor detects a sharp curve, the tracking PID overshoots with a large steering correction. Without speed coupling, the car flies off the track. With coupling, it slows down into the turn.

**Corrective action if FAIL:**
```c
// Speed-steering coupling — tune COEFF empirically
#define STEER_COUPLING_COEFF    0.008f     // 0.008 × |60| = 0.48 → speed × 0.52
#define STEER_COUPLING_FLOOR    0.20f      // Never below 20% of requested speed

// Replace ALL calls to Motor_Drive(linear, angular) with:
// Motor_Drive_Coupled(linear, angular);

// Tuning procedure:
// 1. Set COEFF = 0 and run a sharp turn → car will spin out
// 2. Increase COEFF incrementally (0.002 per test) until car tracks smoothly
// 3. For each COEFF, verify inner wheel never reverses during normal turns
// 4. Document: "COEFF=0.008 @ 12V, carpet surface"
```

---

## Dimension 8: Data Persistence & Debugging Ergonomics (How Many Nights You Spend Tuning)

These aren't physical safety items, but they determine how efficiently you can tune the system. AI never generates these proactively.

---

### Item 25: PID Parameter Persistent Storage

| Aspect | Detail |
|--------|--------|
| **Consequence** | Every PID re-tune requires editing `#define KP 1.2`, recompiling, reflashing. 100 tuning iterations = 100 flash cycles = hours of wasted time. |
| **AI Common Mistake** | Hardcodes PID constants as `#define` or `const float`. Never mentions EEPROM or Flash storage. |

**What to check:**

```c
// ❌ FAIL — hardcoded constants, reflash for every change
#define SPEED_KP   2.0f
#define SPEED_KI   0.5f
#define SPEED_KD   0.1f

// ✅ CORRECT — runtime-configurable PID with Flash storage
typedef struct {
    float Kp, Ki, Kd;
    float output_limit, integral_limit;
    uint32_t crc32;  // Checksum to detect corruption
} PID_Config_Flash_t;

static PID_Config_Flash_t g_pid_config;

// Load from Flash on boot, use defaults if Flash is empty
void PID_Config_Load(void) {
    Flash_Read(PID_FLASH_ADDR, (uint32_t*)&g_pid_config, sizeof(g_pid_config));
    uint32_t calc_crc = CRC32_Calc((uint32_t*)&g_pid_config, sizeof(g_pid_config) - 4);
    if (calc_crc != g_pid_config.crc32) {
        // Flash empty or corrupted → use factory defaults
        g_pid_config.Kp = 2.0f;
        g_pid_config.Ki = 0.5f;
        g_pid_config.Kd = 0.1f;
        g_pid_config.output_limit = 85.0f;
        g_pid_config.integral_limit = 20.0f;
    }
}

// Save to Flash
void PID_Config_Save(void) {
    g_pid_config.crc32 = CRC32_Calc((uint32_t*)&g_pid_config, sizeof(g_pid_config) - 4);
    Flash_ErasePage(PID_FLASH_ADDR);
    Flash_Write(PID_FLASH_ADDR, (uint32_t*)&g_pid_config, sizeof(g_pid_config));
}
```

**Verification rules:**
- PID constants must be runtime-writable, not compile-time constants.
- Storage target: internal Flash (last page reserved for config), external EEPROM (AT24C02 via I2C), or emulated EEPROM in Flash (STM32F1 has no true EEPROM).
- Include a **CRC32 or checksum** to detect corrupted/uninitialized Flash. Factory defaults must be used when CRC fails.
- Provide a **calibration mode**: long-press a button → enter calibration → use UART/Bluetooth/knobs to adjust Kp/Ki/Kd → short-press to save → LED confirms save.
- This item is not a safety-critical FAIL — flag it as ⚠️ RECOMMENDATION rather than ❌ FAIL.

**Corrective action if FAIL:**
```
⚠️ RECOMMENDATION — Add PID persistent storage

  For STM32F103C8 (no EEPROM):
  - Reserve the last 1KB Flash page for config storage.
  - Use the STM32 Flash API: FLASH_ErasePage(), FLASH_ProgramWord().
  - Add a calibration mode triggered by a button long-press.
  - Store a CRC32 alongside the config for corruption detection.
  - Format: struct { float Kp, Ki, Kd; float out_limit, i_limit; uint32_t crc; }

  For chips with EEPROM (STM32L0, ESP32):
  - Use the EEPROM emulation library or Preferences (ESP32).
  - Much simpler — no page erase required.

  This is NOT a safety-critical item, but it will save you hours of reflashing.
```

---

### Item 26: Non-Blocking Debug Logging

| Aspect | Detail |
|--------|--------|
| **Consequence** | `printf()` in main loop → 115200 baud UART transmits ~11.5 KB/s → each `printf("yaw=%.2f pitch=%.2f roll=%.2f\n", ...)` is ~40 bytes → 3.5 ms blocked → at 200 Hz main loop, that's 70% of CPU time wasted → PWM update jitter → motor vibration |
| **AI Common Mistake** | Puts full `printf()` with floating-point formatting in the main control loop |

**What to check:**

```c
// ❌ FAIL — printf in every control loop iteration
while (1) {
    PID_Compute(...);
    Motor_Drive(...);
    printf("yaw=%.2f pitch=%.2f speed=%d\n", g_sensors.yaw, g_sensors.pitch, speed);
    // This printf takes 3.5 ms → at 200 Hz loop, 70% of time is blocked on UART!
}

// ✅ CORRECT — sparse logging, integer-only
#define LOG_INTERVAL_MS     100    // Print once every 100ms, NOT every loop cycle
#define LOG_INTERVAL_CYCLES 20     // Or once every 20 main loop cycles

static uint32_t last_log_tick = 0;
static uint32_t log_cycle_count = 0;

while (1) {
    PID_Compute(...);
    Motor_Drive(...);
    log_cycle_count++;

    if ((GetTick() - last_log_tick) >= LOG_INTERVAL_MS) {
        last_log_tick = GetTick();
        // Integer-only formatting — much faster than float printf
        int16_t yaw_x10 = (int16_t)(g_sensors.yaw * 10.0f);
        int16_t pitch_x10 = (int16_t)(g_sensors.pitch * 10.0f);
        int16_t speed_int = (int16_t)Motor_GetLeftSpeed();
        printf("Y%d P%d S%d\n", yaw_x10, pitch_x10, speed_int);
        // This printf is ~15 bytes → ~1.3 ms → still heavy but 10x less frequent
    }

    Delay_ms(5);
    IWDG_ReloadCounter();
}
```

**Verification rules:**
- Search for `printf()` inside `while(1)` main loops. If present, verify it fires no more than **10 times per second** (every 100ms minimum interval).
- Float formatting (`%f`, `%.2f`) in printf is **very slow** on Cortex-M without hardware FPU. Prefer integer representations (multiply by 10/100/1000, print as integer, parse on the PC side).
- Better: use a **binary logging protocol** (send raw structs over UART, parse on PC) or **ITM/SWO** (SWO pin to debug probe — zero CPU overhead).
- If UART is also used for command reception, the debug prints must yield priority — commands take precedence over logs.
- Production/competition firmware should have logging behind `#ifdef DEBUG_ENABLED` so it can be compiled out entirely for the race.

**Corrective action if FAIL:**
```c
// Non-blocking sparse log pattern
#define DEBUG_ENABLED  1   // Set to 0 for competition build

#if DEBUG_ENABLED
  #define LOG_INTERVAL_MS  100
  static uint32_t _log_timer = 0;
  #define DEBUG_LOG(fmt, ...) \
      do { \
          if ((uint32_t)(GetTick() - _log_timer) >= LOG_INTERVAL_MS) { \
              _log_timer = GetTick(); \
              printf(fmt, ##__VA_ARGS__); \
          } \
      } while(0)
#else
  #define DEBUG_LOG(fmt, ...)  ((void)0)
#endif

// Usage in main loop:
DEBUG_LOG("Y%d P%d S%d\n", yaw_x10, pitch_x10, speed_int);
```

---

## Dimension 9: AI Code Structural Traps (Programmer Self-Defense)

These are bugs that survive compilation and only manifest intermittently. Hardest to debug.

---

### Item 27: Volatile Qualification on Shared Variables

| Aspect | Detail |
|--------|--------|
| **Consequence** | Variable modified in ISR, read in main loop → compiler caches the value in a register → main loop reads stale register value forever → encoder count "stuck", sensor flag "never set", PWM value "frozen" |
| **AI Common Mistake** | Declares global variables without `volatile`, or marks `volatile` only on simple types but not on structs accessed across ISR/main boundaries |

**What to check:**

```c
// ❌ CRITICAL FAIL — shared variables without volatile
// In ISR:
void SysTick_Handler(void) {
    g_sys_tick_ms++;  // Modified in ISR
}

// In main loop:
uint32_t now = g_sys_tick_ms;  // Compiler may cache g_sys_tick_ms → never sees updates!

// ❌ ALSO WRONG — struct without volatile
SensorData_t g_sensors;  // Modified in multiple functions, read across contexts

// ✅ CORRECT — volatile on every ISR-shared variable
volatile uint32_t g_sys_tick_ms = 0;       // ISR writes, main loop reads
volatile CarState_t g_car_state;            // Modified across state machine paths
volatile SensorData_t g_sensors;            // Multiple writers, multiple readers
volatile uint8_t sensor_data_ready = 0;     // ISR sets, main loop clears
```

**Verification rules:**
- **Every global variable** must be audited: is it written in an ISR? Is it written in one function and read in another (especially across different `.c` files)? If yes → must be `volatile`.
- Struct members don't inherit `volatile` from the struct — if `g_sensors` is `volatile SensorData_t`, all members are volatile. But if `g_sensors` is NOT volatile and a member `g_sensors.yaw` is accessed in both ISR and main, the compiler can cache the member separately. The entire struct must be volatile.
- Stale data is especially dangerous for: encoder counts, sensor data ready flags, system tick, car state, PID setpoints changed via UART commands.
- Read-modify-write on volatile variables in main loop must be protected from ISR preemption during the RMW sequence. If the variable is > 32 bits (e.g., `uint64_t` on Cortex-M3), a single load/store is not atomic — use critical sections (`__disable_irq()` / `__enable_irq()`) or atomic access functions.

**Corrective action if FAIL:**

For your specific K题小车 code, the following variables need `volatile`:
```c
// main.c
volatile uint32_t g_sys_tick_ms = 0;    // ← already has volatile ✓
volatile CarState_t g_car_state;         // ← MISSING volatile!
// g_car_state is modified in Algorithm_RunMission() and Algorithm_StartMission()
// and read in main()'s while loop → MUST be volatile

// sensors.c
volatile SensorData_t g_sensors;         // ← MISSING volatile!
// g_sensors is written by Sensors_UpdateAll(), MPU6050_Update()
// and read by Algorithm_Navigate(), OLED_ShowDebug() → MUST be volatile

// algorithm.c
// CarOdom_t g_odom is declared extern in main.h but its struct members
// are written across different algorithm states → SHOULD be volatile
```

---

### Item 28: Implicit Type Conversion Overflow

| Aspect | Detail |
|--------|--------|
| **Consequence** | `int16_t` encoder count × `float` KP → intermediate int16 overflow before float conversion → garbage PID output → motor jump |
| **AI Common Mistake** | Mixes integer and float types without explicit casts in PID and sensor math |

**What to check:**

```c
// ❌ FAIL — integer overflow before float conversion
int16_t encoder_cnt = 32000;  // Near int16_t max
float speed = encoder_cnt * SPEED_SCALE;  // int16 × float → int16 multiply first!
// 32000 × 100 = 3,200,000 → overflow int16_t → -31072 → garbage!

// ❌ ALSO WRONG — PID output from float to int16 without range check
int16_t pwm_out = (int16_t)PID_Compute(&pid, input, dt);  // Float truncation!

// ✅ CORRECT — explicit cast BEFORE multiply
float speed = (float)encoder_cnt * SPEED_SCALE;  // Cast first → safe float multiply

// ✅ CORRECT — clamp before integer conversion
float pid_out = PID_Compute(&pid, input, dt);
if (pid_out > 32767.0f)  pid_out = 32767.0f;
if (pid_out < -32768.0f) pid_out = -32768.0f;
int16_t pwm_out = (int16_t)pid_out;  // Safe: float was clamped to int16 range
```

**Verification rules:**
- Search for all lines where an integer type (`int16_t`, `int32_t`, `uint8_t`) is multiplied by or divided by a float. The integer must be explicitly cast to `(float)` before the operation.
- Search for all `(int16_t)float_expression` or `(int32_t)float_expression` casts — these are the PID→PWM and sensor→int conversion points. The float must be range-checked before casting.
- **Specific danger zone**: `atan2f()` returns `float` in `[-π, π]`. This value converted to degrees (`180/π`) is `[-180, 180]` — safe for int16. But intermediate calculations (gyro integration, angle wrapping) can produce values outside this range.
- **PID accumulator**: `pid->integral` is a float. If integral_limit is not set, it can grow to ±1e38 and overflow even float precision. This is why integral_limit clamping (Item 7) matters for type safety too.

**Corrective action if FAIL:**
```c
// Safe type conversion macros
#define TO_FLOAT(x)      ((float)(x))   // Always cast before float ops
#define TO_INT16_CLAMP(x) ((int16_t)( \
    ((x) > 32767.0f) ? 32767 : ((x) < -32768.0f) ? -32768 : (int16_t)(x) \
))

// Usage:
float speed = TO_FLOAT(encoder_count) * SPEED_SCALE;    // Safe
int16_t pwm = TO_INT16_CLAMP(pid_output);                // Safe

// In your existing code, fix these patterns:
// algorithm.c:100
// OLD: angular_speed = (int16_t)PID_Compute(&g_heading_pid, g_sensors.yaw, dt);
// NEW:
float heading_raw = PID_Compute(&g_heading_pid, g_sensors.yaw, dt);
angular_speed = TO_INT16_CLAMP(heading_raw);

// algorithm.c:68 (and everywhere you assign float to int16_t)
// OLD: int16_t linear_speed = (int16_t)speed_pct;
// This is relatively safe (speed_pct ≤ 100) but still fragile:
int16_t linear_speed = TO_INT16_CLAMP(speed_pct);
```

---

## Dimension 10: Kinematics & Motion Smoothing

AI treats motor commands as pure math output — it doesn't understand inertia, gear backlash, or that wheel direction signs might be wrong.

---

### Item 29: Mecanum/Swerve Wheel Sign Verification

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Inverse kinematics sign errors → car moves diagonally or spins in place instead of going straight → all closed-loop control is garbage because the physical motion doesn't match the math |
| **AI Common Mistake** | Generates `V1 = Vx - Vy + W` without verifying that wheel 1 actually rotates in the expected direction |
| **Trigger Condition** | **Check ONLY if the system uses mecanum wheels, omni wheels, or swerve drive. Skip this item for conventional differential-drive or Ackermann-steering vehicles.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — unverified inverse kinematics matrix
void Mecanum_Drive(float vx, float vy, float omega) {
    // AI generated these signs — but are they correct for YOUR wheel arrangement?
    float v1 =  vx - vy - omega * (Lx + Ly);  // Wheel 1 (Front-Left)
    float v2 =  vx + vy + omega * (Lx + Ly);  // Wheel 2 (Front-Right)
    float v3 =  vx + vy - omega * (Lx + Ly);  // Wheel 3 (Rear-Right)
    float v4 =  vx - vy + omega * (Lx + Ly);  // Wheel 4 (Rear-Left)
    // No verification → signs may be wrong → car moves diagonally
}

// ✅ CORRECT — individual wheel test function before closed-loop operation
// Step 1: Hardware verification function (run ONCE during bring-up)
void Wheel_Direction_Test(void) {
    const char *wheel_names[] = {"FL", "FR", "RR", "RL"};
    const int16_t test_speed = 30;  // Low speed for visual check

    for (int i = 0; i < 4; i++) {
        OLED_ShowString(0, 0, "Test:");
        OLED_ShowString(36, 0, (char*)wheel_names[i]);
        OLED_ShowString(0, 2, "Should move FWD");
        OLED_Update();

        // Spin ONLY this one wheel forward
        Mecanum_SingleWheel(i, test_speed);
        Delay_ms(2000);  // Human observes: does this wheel push the car FORWARD?
        Mecanum_SingleWheel(i, 0);
        Delay_ms(1000);

        // Spin ONLY this one wheel backward
        OLED_ShowString(0, 2, "Should move REV");
        OLED_Update();
        Mecanum_SingleWheel(i, -test_speed);
        Delay_ms(2000);
        Mecanum_SingleWheel(i, 0);
        Delay_ms(1000);
    }
    OLED_ShowString(0, 4, "Test Complete");
    OLED_Update();
}

// Step 2: If any wheel moves in the wrong direction, fix in the matrix:
// - Motor wiring reversed? → swap IN1/IN2 for that motor
// - Encoder direction reversed? → add a DIR_INVERT flag per wheel

// Step 3: After sign verification, run a translation test:
//   vx=30, vy=0 → car must move straight forward
//   vx=0, vy=30 → car must move straight right
//   vx=0, vy=0, omega=20 → car must rotate CW
```

**Verification rules:**
- Before ANY closed-loop control, a **single-wheel test function** must exist and be documented in comments.
- Each wheel must be tested individually: spin forward → human confirms car moves forward; spin backward → human confirms car moves backward.
- The inverse kinematics matrix signs (`+` or `-` for each of vx, vy, omega) depend on **physical motor orientation, wiring polarity, and encoder mounting direction**. AI cannot know these — ONLY physical testing can verify.
- After individual wheel tests, run 3 translation tests (forward, right, CW rotation) and verify visually.
- If encoder-based speed control is used, also verify that each encoder counts up when the wheel rotates forward.

**Corrective action if FAIL:**
```
⚠️ NEEDS MANUAL VERIFICATION — Wheel direction test required

  Before trusting any closed-loop control:
  1. Flash the wheel direction test firmware.
  2. For each wheel (FL, FR, RR, RL), observe the car's movement:
     - Wheel spins forward → car should translate forward
     - Wheel spins backward → car should translate backward
  3. If any wheel moves wrong: swap the motor phase wires OR add a sign
     inversion flag for that wheel in the kinematics matrix.
  4. Document which wheels were inverted in code comments.
  5. Re-run the forward/right/CW translation test to confirm.

  Without this test, you'll debug "PID oscillation" for hours
  when the real problem is the car can't even go straight.
```

---

### Item 30: Soft Start / Soft Stop Acceleration Limiting

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Speed setpoint jumps 0→100 instantly → PID demands 100% duty → gearbox experiences hammer impact → first-stage plastic gear teeth shear off; simultaneous current surge causes battery voltage sag → MCU brownout reset |
| **AI Common Mistake** | Target speed changes are instantaneous — `target_speed = 100` happens in one line with no ramp |

**What to check:**

```c
// ❌ CRITICAL FAIL — instantaneous speed change
void Algorithm_Navigate(float target_heading, float speed_pct) {
    int16_t linear_speed = (int16_t)speed_pct;   // Jumps from 0 to 70 in one cycle!
    Motor_Drive(linear_speed, angular_speed);
}

// ✅ CORRECT — first-order low-pass filter on speed setpoint
static float g_smoothed_speed = 0.0f;
#define SPEED_SMOOTHING_ALPHA   0.10f   // 10% new + 90% old per cycle → ~50ms to 90% at 200Hz

void Algorithm_Navigate(float target_heading, float speed_pct) {
    // Smooth the speed setpoint with exponential moving average
    g_smoothed_speed = g_smoothed_speed * (1.0f - SPEED_SMOOTHING_ALPHA)
                       + speed_pct * SPEED_SMOOTHING_ALPHA;

    int16_t linear_speed = (int16_t)g_smoothed_speed;
    // ... rest of navigation logic using smoothed speed
}

// Alternative: linear ramp function
static float g_ramp_speed = 0.0f;
#define SPEED_RAMP_RATE   5.0f   // Max ±5% change per control cycle

float RampSpeed(float target) {
    float delta = target - g_ramp_speed;
    if (delta > SPEED_RAMP_RATE)       g_ramp_speed += SPEED_RAMP_RATE;
    else if (delta < -SPEED_RAMP_RATE) g_ramp_speed -= SPEED_RAMP_RATE;
    else                               g_ramp_speed = target;
    return g_ramp_speed;
}
```

**Verification rules:**
- All speed setpoints (linear, angular, individual wheel targets) must pass through a **ramp or low-pass filter** before being sent to the motor.
- The smoothing factor must be **tunable** (macro or config variable). Start with α=0.10 (at 200Hz: reaches 90% in ~22 cycles = 110ms). Too slow → sluggish response. Too fast → no smoothing.
- Search for every `Motor_Drive()` or `Motor_Left_Set()/Motor_Right_Set()` call site. Trace back the speed/steer values: do they come directly from a PID output or hardcoded constant? If so → FAIL.
- Exceptions: the emergency stop ramp (Item 18) can bypass this filter because it's an explicit override. But normal speed changes must go through the filter.
- For balancing robots: the balance PID output (corrective torque) should NOT be filtered — only the translational speed setpoint should be. Filtering the balance output would make the bot unable to react to tipping.

**Corrective action if FAIL:**
```c
// Speed smoothing — add to ALL speed setpoint paths
typedef struct {
    float current;
    float alpha;       // 0.05–0.20: lower = smoother but slower
    float max_delta;   // Max change per cycle (for ramp variant)
} SpeedSmoother_t;

float SpeedSmooth_EMA(SpeedSmoother_t *s, float target) {
    s->current = s->current * (1.0f - s->alpha) + target * s->alpha;
    return s->current;
}

// In your existing code:
// algorithm.c:68  OLD: int16_t linear_speed = (int16_t)speed_pct;
// algorithm.c:68  NEW: int16_t linear_speed = (int16_t)SpeedSmooth_EMA(&g_speed_smoother, speed_pct);
```

---

## Dimension 11: Sensor Calibration & Environmental Drift

AI generates calibration code that works once, at room temperature, on a bench. Competition reality is different.

---

### Item 31: Gyroscope Bias Thermal Drift Dynamic Compensation

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Gyro bias calibrated at cold boot → 3 minutes of operation → chip self-heats 10°–15°C → bias shifts → car drifts left/right in a straight line → balance bot leans progressively until it falls |
| **AI Common Mistake** | Calibrates gyro zero-rate offset once at startup, then uses that static value forever |

**What to check:**

```c
// ❌ FAIL — one-time calibration, never updated
void MPU6050_Init(void) {
    // Average 500 samples at startup for gyro bias
    float sum_gx = 0, sum_gy = 0, sum_gz = 0;
    for (int i = 0; i < 500; i++) {
        MPU6050_ReadRaw(&raw);
        sum_gx += raw.gyro_x;
        sum_gy += raw.gyro_y;
        sum_gz += raw.gyro_z;
        Delay_ms(2);
    }
    g_gyro_bias_x = sum_gx / 500.0f;
    g_gyro_bias_y = sum_gy / 500.0f;
    g_gyro_bias_z = sum_gz / 500.0f;
    // This bias is valid for ~30 seconds before chip heating shifts it
}

// ✅ CORRECT — static detection + sliding window dynamic recalibration
#define GYRO_BIAS_WINDOW      200    // 200 samples for sliding average
#define STATIC_VARIANCE_THRESH  0.005f  // Accel variance below this → truly static

static float g_gyro_bias_buffer[3][GYRO_BIAS_WINDOW];
static int   g_gyro_bias_idx = 0;
static int   g_gyro_bias_filled = 0;

static uint8_t IsVehicleStatic(void) {
    static float accel_history[3][20] = {0};
    static int accel_idx = 0;
    float accel[3] = {g_sensors.accel_x, g_sensors.accel_y, g_sensors.accel_z};
    float var[3] = {0};

    // Store in circular buffer
    accel_history[0][accel_idx] = accel[0];
    accel_history[1][accel_idx] = accel[1];
    accel_history[2][accel_idx] = accel[2];
    accel_idx = (accel_idx + 1) % 20;

    // Compute variance over 20 samples
    float mean[3] = {0};
    for (int i = 0; i < 20; i++) {
        mean[0] += accel_history[0][i];
        mean[1] += accel_history[1][i];
        mean[2] += accel_history[2][i];
    }
    mean[0] /= 20.0f; mean[1] /= 20.0f; mean[2] /= 20.0f;
    for (int i = 0; i < 20; i++) {
        var[0] += (accel_history[0][i] - mean[0]) * (accel_history[0][i] - mean[0]);
        var[1] += (accel_history[1][i] - mean[1]) * (accel_history[1][i] - mean[1]);
        var[2] += (accel_history[2][i] - mean[2]) * (accel_history[2][i] - mean[2]);
    }
    var[0] /= 20.0f; var[1] /= 20.0f; var[2] /= 20.0f;

    // All 3 axes must be nearly stationary
    return (var[0] < STATIC_VARIANCE_THRESH &&
            var[1] < STATIC_VARIANCE_THRESH &&
            var[2] < STATIC_VARIANCE_THRESH);
}

// Call in main loop (runs when vehicle is detected static for >1 second):
static uint32_t static_start_time = 0;
static uint8_t  was_static = 0;

void GyroBias_UpdateDynamic(void) {
    if (IsVehicleStatic()) {
        if (!was_static) {
            static_start_time = GetTick();
            was_static = 1;
        }
        // Only recalibrate after 1 second of continuous static state
        if (GetTick() - static_start_time > 1000) {
            // Sliding window: replace oldest sample with new
            g_gyro_bias_buffer[0][g_gyro_bias_idx] = g_sensors.gyro_x;
            g_gyro_bias_buffer[1][g_gyro_bias_idx] = g_sensors.gyro_y;
            g_gyro_bias_buffer[2][g_gyro_bias_idx] = g_sensors.gyro_z;
            g_gyro_bias_idx = (g_gyro_bias_idx + 1) % GYRO_BIAS_WINDOW;
            if (g_gyro_bias_idx == 0) g_gyro_bias_filled = 1;

            // Compute new bias from filled window
            if (g_gyro_bias_filled) {
                float sum[3] = {0};
                for (int i = 0; i < GYRO_BIAS_WINDOW; i++) {
                    sum[0] += g_gyro_bias_buffer[0][i];
                    sum[1] += g_gyro_bias_buffer[1][i];
                    sum[2] += g_gyro_bias_buffer[2][i];
                }
                g_gyro_bias_x = sum[0] / GYRO_BIAS_WINDOW;
                g_gyro_bias_y = sum[1] / GYRO_BIAS_WINDOW;
                g_gyro_bias_z = sum[2] / GYRO_BIAS_WINDOW;
            }
        }
    } else {
        was_static = 0;
    }
}
```

**Verification rules:**
- Gyro bias must be recalibrated **dynamically** during operation, not just once at boot.
- A **static detection** algorithm (accelerometer variance over ≥20 samples) must be present. When the vehicle is stationary for ≥1 second, gyro readings are re-averaged into a sliding window.
- The sliding window size should hold **≥100 samples** — enough to average out sensor noise but not so large that it takes minutes to adapt.
- Dynamic recalibration handles both thermal drift AND mechanical vibration shifts (e.g., after a hard turn shifts the IMU mounting).
- For competition: recalibration automatically triggers during pre-race waiting periods without the driver needing to do anything. If the competition has a 10-second countdown before start, that's 10 seconds of perfect static calibration.
- Over-correction guard: limit how much the bias can change per recalibration cycle (e.g., max ±0.5°/s change per update) to prevent a faulty "static" detection from corrupting the bias estimate.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Dynamic gyro bias recalibration

  Your current code (mpu6050.c) has no gyro bias calibration at all —
  it integrates raw gyro_z directly into yaw. This means yaw drift rate
  depends on the gyro's zero-rate offset, which shifts with temperature.

  Minimum fix: add a one-time bias capture at boot (average 500 samples
  while stationary before the 3-2-1 countdown LED sequence).

  Better fix: implement the sliding-window dynamic recalibration above.
  The car is naturally stationary during: countdown, obstacle stops,
  and mission completion. Each of these is free calibration data.
```

---

### Item 35: Sensor Confidence Check & Fallback Mode

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Sunlight hits the line sensor → all readings saturate → tracking algorithm computes garbage steering angle → car drives full-speed off the track into obstacles |
| **AI Common Mistake** | Assumes sensor data is always valid; feeds raw ADC readings directly into PID without sanity checks |
| **Trigger Condition** | **Check if the system uses line-following (gray/IR array), camera/vision tracking, or any environment-dependent sensor. Skip if the vehicle is purely IMU+encoder dead-reckoning with no optical sensors.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — no data confidence check
float line_position = ComputeLinePosition(gray_binary);  // gray_binary = {0,0,0,0,0} (all black = lost!)
float steer = LineTrackingPID_Compute(&pid, line_position, CENTER);  // Garbage in → garbage out
Motor_Drive(speed, steer);  // Car follows garbage steer → crashes

// ✅ CORRECT — confidence check before using sensor data
typedef enum {
    SENSOR_OK = 0,
    SENSOR_LOW_CONFIDENCE,  // Marginal — slow down, use last good value
    SENSOR_INVALID           // Garbage — switch to open-loop safety mode
} SensorConfidence_t;

SensorConfidence_t GraySensor_CheckConfidence(void) {
    // Check 1: min-max spread — if all sensors read nearly identical, no line is visible
    uint16_t min_val = 4095, max_val = 0;
    for (int i = 0; i < GRAY_SENSOR_NUM; i++) {
        if (g_sensors.gray_raw[i] < min_val) min_val = g_sensors.gray_raw[i];
        if (g_sensors.gray_raw[i] > max_val) max_val = g_sensors.gray_raw[i];
    }
    uint16_t spread = max_val - min_val;
    if (spread < 50) return SENSOR_INVALID;  // <50 ADC counts → no contrast → lost

    // Check 2: all-black or all-white (sensor saturation)
    uint8_t all_black = 1, all_white = 1;
    for (int i = 0; i < GRAY_SENSOR_NUM; i++) {
        if (g_sensors.gray_raw[i] > 200) all_black = 0;
        if (g_sensors.gray_raw[i] < 3900) all_white = 0;
    }
    if (all_black || all_white) return SENSOR_INVALID;

    // Check 3: number of detected line points
    uint8_t line_points = 0;
    for (int i = 0; i < GRAY_SENSOR_NUM; i++) {
        if (g_sensors.gray_binary[i] == 0) line_points++;
    }
    if (line_points < 2) return SENSOR_LOW_CONFIDENCE;  // Marginal detection
    if (line_points > 4) return SENSOR_INVALID;         // Too many → noise

    return SENSOR_OK;
}

// In main navigation loop:
void Algorithm_Navigate(float target_heading, float speed_pct) {
    Sensors_UpdateAll();
    SensorConfidence_t conf = GraySensor_CheckConfidence();

    if (conf == SENSOR_INVALID) {
        // FALLBACK: open-loop straight + gradual stop
        Motor_Drive(20, 0);  // Low speed straight ahead (don't guess!)
        // If no recovery after 2 seconds → emergency stop
        static uint32_t lost_timer = 0;
        if (lost_timer == 0) lost_timer = GetTick();
        if (GetTick() - lost_timer > 2000) {
            Motor_Stop_Safe();
            g_car_state = STATE_ERROR;
        }
    } else if (conf == SENSOR_LOW_CONFIDENCE) {
        // REDUCED speed, use last known good steer direction
        float steer = g_last_good_steer * 0.5f;  // Hold mild correction
        Motor_Drive(speed_pct * 0.4f, steer);
    } else {
        // Normal operation
        // ... compute line position and steer normally
    }
}
```

**Verification rules:**
- Every sensor type that feeds into a control loop must have a **confidence assessment function** returning at least 3 levels: OK / LOW_CONFIDENCE / INVALID.
- When confidence is INVALID: the system must NOT use the sensor for closed-loop control. Fallback to open-loop safety mode (straight + gradual stop).
- When confidence is LOW: reduce speed, reduce steering authority, and blend with last known good value.
- The confidence check must run every control cycle — not once per second.
- For vision/camera: check the number of detected features (line points, AprilTag corners, etc.). If below threshold → invalid.
- For IR distance: check that readings are within the sensor's valid range (e.g., E18-D80NK: 3–80 cm). Values at the extremes (0 cm or >80 cm) may indicate noise or saturation.

**Corrective action if FAIL:**
```c
// Add to your existing sensor code (sensors.c):
SensorConfidence_t IR_Sensor_CheckConfidence(void) {
    uint8_t active_count = 0;
    for (int i = 0; i < IR_SENSOR_NUM; i++) {
        if (g_sensors.ir_detected[i]) active_count++;
    }
    // If all 4 IR sensors detect obstacles simultaneously → likely noise/light interference
    if (active_count >= 4) return SENSOR_INVALID;
    // If 3 sensors detect → suspicious → low confidence
    if (active_count >= 3) return SENSOR_LOW_CONFIDENCE;
    return SENSOR_OK;
}
```

---

## Dimension 12: Real-Time CPU & DMA Integrity

AI generates code that compiles but violates real-time deadlines. These bugs survive compilation and only appear as intermittent glitches.

---

### Item 32: ISR Timing Overrun Verification

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Control ISR designed for 1ms period takes 1.5ms to execute → ISR re-enters before previous invocation finishes → stack grows without bound → HardFault OR main loop starved → PWM frozen → motor whines at fixed frequency |
| **AI Common Mistake** | Runs `double`-precision Kalman filter, `atan2()`, full PID computation, and OLED frame buffer update ALL inside a 1kHz timer ISR |

**What to check:**

```c
// ❌ CRITICAL FAIL — heavy computation in ISR with no timing verification
void TIM7_IRQHandler(void) {
    // 1kHz timer ISR — supposed to take <500µs
    MPU6050_ReadRaw(&raw);           // ~50µs (I2C)
    ComplementaryFilter(&raw, dt);   // ~200µs (float trigonometry)
    PID_Compute(&pid, input, dt);    // ~100µs
    Motor_Drive(speed, steer);       // ~50µs
    OLED_DrawGraph(&sensors);        // ~2000µs!!! OLED I2C in ISR!
    // Total: ~2400µs > 1000µs period → ISR overrun!
}

// ✅ CORRECT — ISR timing measurement + split heavy work to main loop
// Step 1: Toggle a GPIO at ISR entry/exit for oscilloscope measurement
#define ISR_TIMING_PIN    GPIO_Pin_15   // Use any free GPIO for timing probe
#define ISR_TIMING_PORT   GPIOA

void TIM7_IRQHandler(void) {
    GPIO_SetBits(ISR_TIMING_PORT, ISR_TIMING_PIN);  // Timing start

    // MINIMAL work in ISR:
    MPU6050_ReadRaw_DMA(&raw);        // DMA reads I2C → ~10µs CPU time
    sensor_data_ready = 1;            // Signal main loop

    GPIO_ResetBits(ISR_TIMING_PORT, ISR_TIMING_PIN);  // Timing end
    // Measure high-time with oscilloscope → MUST be < 50% of ISR period
}

// Step 2: Heavy math goes in main loop
while (1) {
    if (sensor_data_ready) {
        sensor_data_ready = 0;
        ComplementaryFilter(&raw, dt);    // Float math in main loop
        PID_Compute(&pid, input, dt);
        Motor_Drive(speed, steer);
    }
    OLED_DrawGraph(&sensors);             // Slow I2C in main loop (only when idle)
    IWDG_ReloadCounter();
}

// Step 3: Use float (not double) throughout — Cortex-M3/4 has single-precision FPU only
// ❌ double → software emulation, 10x slower
// ✅ float  → hardware FPU instruction, 1 cycle
```

**Verification rules:**
- Every timer ISR must have a **GPIO toggle** at entry and exit for oscilloscope measurement. The ISR execution time must be **measurable**, not inferred.
- ISR execution time must be **< 50% of the ISR period**. For a 1kHz timer (1000µs period), the ISR must complete in ≤ 500µs. The remaining 500µs are for the main loop and other ISRs.
- Search for blocking operations inside timer ISRs: I2C reads, SPI transfers, UART printf, OLED updates, Flash writes. Any of these in an ISR → FAIL.
- All floating-point operations must use `float`, not `double`. On Cortex-M3 (STM32F103), `double` is software-emulated and ~10× slower than `float`. On Cortex-M4F (STM32F407), `float` is hardware-accelerated but `double` is still software.
- If CMSIS-DSP functions are used (`arm_sqrt_f32`, `arm_mat_mult_f32`), verify they are the `f32` variants, not `f64`.
- For complex filters: if execution time is borderline, reduce iteration count (e.g., Mahony filter with 1 iteration instead of 3; complementary filter with 0.98/0.02 weights instead of Kalman).

**Corrective action if FAIL:**
```
⚠️ NEEDS MANUAL VERIFICATION — ISR timing measurement

  1. Add GPIO toggle at ISR entry/exit (free GPIO pin → oscilloscope probe).
  2. Measure high-time at full control rate (worst case: all sensors active, max PID iterations).
  3. If high-time > 50% of ISR period:
     a. Move heavy computation (filter, PID, OLED) from ISR to main loop.
     b. Change all 'double' to 'float' (especially in atan2, sqrt, sin/cos calls).
     c. Replace blocking I2C reads with DMA-based reads.
     d. If still over budget: reduce ISR frequency (500Hz is acceptable for most ground vehicles).
  4. Document the measured time in a code comment:
     // Measured ISR high-time: 340µs, period: 1000µs (34%) @ STM32F103 72MHz
```

---

### Item 39: DMA-CPU Cache Coherency (Torn Reads)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | DMA writes ADC data to array while CPU reads the same array → CPU sees half-old half-new data ("torn read") → battery voltage reads 25.3V for one cycle → false low-voltage trigger → motors cut out mid-race |
| **AI Common Mistake** | Configures DMA in circular mode to a single buffer, CPU reads directly from that buffer with no synchronization |

**What to check:**

```c
// ❌ CRITICAL FAIL — shared buffer, no synchronization
static uint16_t adc_dma_buffer[8];  // DMA writes, CPU reads — same buffer!

void ADC_DMA_Init(void) {
    DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;
    DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)adc_dma_buffer;  // Single buffer
    // ...
}

float Battery_ReadVoltage(void) {
    // CPU reads adc_dma_buffer[0] while DMA might be writing it!
    uint16_t raw = adc_dma_buffer[0];  // TORN READ: high byte old, low byte new
    return (raw / 4095.0f) * 3.3f * VOLTAGE_DIVIDER_RATIO;
}

// ✅ CORRECT — double buffering with completion flag
#define ADC_BUF_SIZE  8
static uint16_t adc_buffer_a[ADC_BUF_SIZE];  // Buffer A
static uint16_t adc_buffer_b[ADC_BUF_SIZE];  // Buffer B
static volatile uint8_t active_read_buffer = 0;  // 0=CPU reads B (DMA fills A), 1=CPU reads A (DMA fills B)
static volatile uint8_t buffer_updated = 0;

// DMA HT (Half Transfer) ISR: Buffer A full → switch DMA to Buffer B, signal CPU to read A
void DMA1_Channel1_IRQHandler(void) {
    if (DMA_GetITStatus(DMA1_IT_HT1)) {   // Half-transfer = Buffer A full
        DMA_ClearITPendingBit(DMA1_IT_HT1);
        active_read_buffer = 0;            // CPU should read Buffer B? No — HT means first half done
        // Actually for double-buffering, use TC (Transfer Complete) with M0/M1 addressing
    }
}

// Better approach: DMA with M0/M1 double-buffer mode (STM32F4/F7/H7)
void ADC_DMA_DoubleBuffer_Init(void) {
    DMA_DoubleBufferModeConfig(DMA2_Stream0, (uint32_t)adc_buffer_b, DMA_Memory_0);
    DMA_DoubleBufferModeCmd(DMA2_Stream0, ENABLE);
    // DMA fills Buffer A → interrupt → CPU reads Buffer A, DMA switches to Buffer B → etc.
}

// Simplest approach for STM32F1 (no double-buffer hardware):
static volatile uint8_t dma_complete = 0;
static uint16_t adc_safe_copy[ADC_BUF_SIZE];  // CPU-only buffer

void DMA1_Channel1_IRQHandler(void) {
    if (DMA_GetITStatus(DMA1_IT_TC1)) {
        DMA_ClearITPendingBit(DMA1_IT_TC1);
        // Copy DMA buffer to safe buffer INSIDE the ISR (DMA is stopped after TC in non-circular mode)
        memcpy(adc_safe_copy, adc_dma_buffer, sizeof(adc_safe_copy));
        dma_complete = 1;
    }
}

float Battery_ReadVoltage_Safe(void) {
    if (!dma_complete) return last_valid_voltage;  // Use previous value if DMA not ready
    uint16_t raw = adc_safe_copy[0];  // Safe: this buffer is only written in ISR
    return (raw / 4095.0f) * 3.3f * VOLTAGE_DIVIDER_RATIO;
}
```

**Verification rules:**
- DMA destination buffer and CPU read buffer must be **different memory regions**, or DMA writes must be synchronized with a completion flag/interrupt.
- For circular DMA: CPU must either (a) read only inside the DMA TC/HT interrupt handler, or (b) use hardware double-buffering (DMA M0/M1 mode on STM32F4+).
- Memory barriers: on Cortex-M4/M7 with cache (e.g., STM32F407 with ART accelerator), DMA writes may land in SRAM but the CPU's cache still holds the old value. Use `SCB_CleanInvalidateDCache()` or `__DSB()` before reading DMA buffers.
- Search for all `__HAL_DMA_GET_COUNTER()` or `DMA_GetCurrDataCounter()` calls — these are used to compute how much data DMA has transferred. The counter changes asynchronously; the result is only valid inside the DMA ISR.
- Array alignment: DMA buffers must be aligned to the data width (4-byte alignment for 32-bit, cache-line alignment for cached systems). Use `__attribute__((aligned(32)))` or `ALIGN_32BYTES()`.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — DMA double-buffering

  For STM32F103 (no hardware double-buffer):
  1. Use DMA TC interrupt to copy data to a CPU-safe buffer.
  2. CPU reads ONLY the safe copy, never the DMA target buffer.
  3. Alternatively: disable the DMA stream, read, re-enable — but this
     causes data gaps (~few µs). Acceptable for slow-changing signals
     like battery voltage (read every 100ms).

  For STM32F4/F7/H7:
  1. Use DMA M0/M1 double-buffer mode (hardware auto-switching).
  2. CPU reads the inactive buffer while DMA fills the active one.

  Minimum fix for your K题小车:
  - Battery voltage ADC (Item 5): sample 8× → average in ISR → store result
    in volatile float. Main loop reads the float directly (atomic on Cortex-M3
    for 32-bit values). This avoids DMA entirely for low-rate signals.
```

---

## Dimension 13: Wireless Control & Mode Transition

AI treats wireless commands as always-fresh and mode switches as instantaneous. In the real world, RF drops out and controllers have inertia.

---

### Item 33: Wireless Command Heartbeat Timeout

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Bluetooth/WiFi packet loss → last received command (`'F'` = forward) stays in buffer → car keeps going forward into a wall; or auto/manual mode switch → PID integrator still holds pre-switch error → violent jerk |
| **AI Common Mistake** | `if (Recv_Cmd == 'F') go_forward();` — the command variable persists forever. AI never adds a timeout. |
| **Trigger Condition** | **Check ONLY if the system uses Bluetooth, WiFi, NRF24L01, or any wireless remote control. Skip if the vehicle is fully autonomous with no remote command input.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — stale command, no heartbeat
volatile char g_uart_cmd = 0;  // Set by UART ISR, read by main loop

void MainLoop(void) {
    if (g_uart_cmd == 'F') {
        Motor_Drive(70, 0);  // Forward at 70% speed
    } else if (g_uart_cmd == 'S') {
        Motor_Stop();
    }
    // g_uart_cmd stays 'F' forever if Bluetooth disconnects → car never stops!
}

// ✅ CORRECT — heartbeat watchdog + command timeout
#define CMD_TIMEOUT_MS      200     // 200ms without new command → emergency stop
#define HEARTBEAT_INTERVAL_MS 50    // Host sends heartbeat every 50ms

static volatile uint32_t g_last_cmd_tick = 0;
static volatile uint8_t  g_uart_cmd = 0;
static volatile uint8_t  g_cmd_updated = 0;

void UART_ProcessInMainLoop(void) {
    if (g_cmd_updated) {
        g_cmd_updated = 0;
        g_last_cmd_tick = GetTick();  // Refresh heartbeat timer
        // Process command...
    }

    // Heartbeat timeout check — runs EVERY main loop iteration
    if ((GetTick() - g_last_cmd_tick) > CMD_TIMEOUT_MS) {
        Motor_Stop_Safe();                    // Emergency stop
        g_car_state = STATE_ERROR;            // Enter safe state
        LED_Alarm_Blink();                    // Alert user: signal lost
    }
}

// Mode switch with PID reset:
void SwitchToAutoMode(void) {
    Motor_Stop_Safe();
    PID_Reset(&g_speed_pid);       // Clear all integrators!
    PID_Reset(&g_heading_pid);
    g_smoothed_speed = 0.0f;      // Reset speed ramp
    g_car_state = STATE_NAVIGATE;
}

void SwitchToManualMode(void) {
    Motor_Stop_Safe();
    PID_Reset(&g_speed_pid);
    PID_Reset(&g_heading_pid);
    g_smoothed_speed = 0.0f;
    g_car_state = STATE_IDLE;      // Wait for manual commands
}
```

**Verification rules:**
- A command timeout (typically **200ms**) must be enforced. If no new command arrives within the timeout window, motors must stop.
- The host/remote should send periodic heartbeat packets (every 50–100ms) even when no control change occurs. This lets the vehicle distinguish "connected but idle" from "disconnected."
- On mode switch (manual↔auto, or between different auto sub-modes), ALL PID integrators must be zeroed, and the speed ramp must be reset to 0.
- The command buffer variable must be tagged with an update flag (`g_cmd_updated`) so the main loop can distinguish "same command sent again" from "no command received."
- **Never** call `HAL_Delay()` or any blocking function inside the command processing path. Command parsing is time-sensitive.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Wireless command heartbeat

  If your K题小车 currently operates fully autonomously (no remote control):
  mark this item as SKIP.

  If you add Bluetooth/WiFi remote control later:
  1. Implement a 200ms command timeout watchdog.
  2. Send heartbeat packets from host every 50ms (can be a single byte: 0xFF).
  3. On timeout: Motor_Stop_Safe() + STATE_ERROR.
  4. On any mode switch: PID_Reset() all controllers + reset speed ramp.
```

---

### Item 34: PID Parameter Bumpless Transfer

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Switching from "straight-line PID" (Kp=5.0) to "sharp-turn PID" (Kp=1.0) → current output was 70% at old Kp → new PID initializes with integral=0 → output jumps to ~10% → car suddenly decelerates → or worse, output sign flips → motor reverses mid-turn → gear stripped |
| **AI Common Mistake** | Loads new Kp/Ki/Kd values without re-initializing the integrator to maintain output continuity |

**What to check:**

```c
// ❌ CRITICAL FAIL — abrupt PID parameter switch
void SwitchToTurnPID(void) {
    g_pid.Kp = 1.0f;   // Was 5.0 → output instantly drops to 1/5!
    g_pid.Ki = 0.1f;
    g_pid.Kd = 0.3f;
    // integral still holds value from straight-line mode → huge overcorrection
}

// ✅ CORRECT — bumpless transfer: match old output with new parameters
void PID_SwitchParams_Bumpless(PID_t *pid, float new_Kp, float new_Ki, float new_Kd) {
    // Step 1: Save current output
    float current_output = pid->P_Term + pid->Ki * pid->integral + pid->D_Term;

    // Step 2: Apply new gains
    pid->Kp = new_Kp;
    pid->Ki = new_Ki;
    pid->Kd = new_Kd;

    // Step 3: Recalculate integral so P+I+D using new gains = same output
    // At steady state, D≈0, so: Kp_new * error + Ki_new * integral_new ≈ current_output
    // → integral_new = (current_output - Kp_new * pid->prev_error) / Ki_new
    if (fabsf(new_Ki) > 0.0001f) {
        pid->integral = (current_output - new_Kp * pid->prev_error) / new_Ki;
    } else {
        pid->integral = 0;  // Ki=0 means P-only controller
    }

    // Step 4: Clamp to new integral limit
    if (pid->integral > pid->integral_limit)  pid->integral = pid->integral_limit;
    if (pid->integral < -pid->integral_limit) pid->integral = -pid->integral_limit;

    // Result: output transitions smoothly — no jerk, no surge, no gear shock
}
```

**Verification rules:**
- Every location where PID gains are modified at runtime must use a bumpless transfer function, not direct assignment.
- The bumpless transfer formula: `I_new = (Output_old − Kp_new × Error_current − Kd_new × Derivative_current) / Ki_new`, then clamp to limits.
- This is especially critical for: multi-mode competition robots (straight/turn/park), balancing robots switching between balance and rest modes, and any controller with gain scheduling.
- **Edge case**: switching to Ki=0 (P-only or PD-only) → integrator must be zeroed since division by zero is undefined. The output will have a step change proportional to the old I-term contribution — this is unavoidable but safe if documented.
- Verify that after parameter switch, the actual PWM value changes by < 5% per control cycle. A larger change indicates incorrect bumpless math.

**Corrective action if FAIL:**
```c
// Add to pid.c:
void PID_SwitchGains(PID_t *pid, float kp, float ki, float kd) {
    // Capture current effective output before gain change
    float P_now = pid->Kp * pid->prev_error;
    float I_now = pid->Ki * pid->integral;
    float D_now = (pid->prev_error != pid->prev_prev_error)
                  ? pid->Kd * (pid->prev_error - pid->prev_prev_error) / pid->dt
                  : 0.0f;
    float output_before = P_now + I_now + D_now;

    // Apply new gains
    pid->Kp = kp;
    pid->Ki = ki;
    pid->Kd = kd;

    // Recompute integral for output continuity
    pid->integral = ((ki > 0.0001f) ? (output_before - kp * pid->prev_error) / ki : 0.0f);

    // Safety clamp
    float limit = pid->integral_limit;
    if (pid->integral > limit) pid->integral = limit;
    else if (pid->integral < -limit) pid->integral = -limit;
}
```

---

## Dimension 14: State Machine & Input Robustness

AI writes switch-case state machines that assume every transition condition will eventually be met. In the physical world, wheels slip, sensors miss, and buttons bounce.

---

### Item 36: State Machine Timeout Protection

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Wheels slip on a turn → encoder never reaches "90°" target → state machine stuck in TURNING state forever → car spins in place until battery dies. (Your K题小车 has EXACTLY this bug!) |
| **AI Common Mistake** | `case TURNING: if (encoder_angle >= 90) { state = MOVING; break; }` — with no timeout, no backup exit |

**What to check:**

```c
// ❌ CRITICAL FAIL — state with no timeout
// algorithm.c:143-161 (Algorithm_AvoidBlackCircles):
case 1: /* 右转绕过 */
    Motor_Drive(AVOID_SPEED, AVOID_TURN);
    if (now - phase_timer > 400) { avoid_phase = 2; phase_timer = now; }
    break;
// This has a phase timeout (400ms) — GOOD! But other states don't.
// algorithm.c:298-307 (STATE_NAVIGATE):
// Relies ONLY on ultrasonic distance OR state_elapsed > 10000 to exit.
// What if ultrasonic never reads < 20cm? Car drives straight into wall.

// ✅ CORRECT — every state has BOTH a success condition AND a timeout
typedef struct {
    uint32_t entry_time;       // Tick when state was entered
    uint32_t timeout_ms;       // Max allowed duration in this state
    uint8_t  timeout_action;   // What to do on timeout
} StateTimer_t;

#define STATE_TIMEOUT_MS    5000   // Default: 5 seconds per state

CarState_t Algorithm_RunMission(uint8_t mission_type) {
    static StateTimer_t state_timer = {0};
    uint32_t now = GetTick();

    // State transition → reset timer
    if (g_car_state != prev_state) {
        state_timer.entry_time = now;
        prev_state = g_car_state;
    }
    uint32_t elapsed = now - state_timer.entry_time;

    switch (g_car_state) {
        case STATE_NAVIGATE:
            Algorithm_Navigate(90.0f, 50.0f);

            // Success condition
            if (g_sensors.us_distance_cm[1] < 20.0f) {
                Motor_Stop();
                g_car_state = STATE_DONE;
            }
            // TIMEOUT protection ← THIS IS NEW!
            else if (elapsed > 15000) {   // 15 seconds max for navigation
                Motor_Stop_Safe();
                g_car_state = STATE_ERROR;
                LED_ErrorCode(1);  // Blink error code 1: navigation timeout
            }
            break;

        case STATE_AVOID_BLACK:
            // ... existing logic ...
            // ADD: timeout for the entire STATE, not just phases
            if (elapsed > 20000) {
                Motor_Stop_Safe();
                g_car_state = STATE_ERROR;
                LED_ErrorCode(2);  // Error code 2: avoid-black timeout
            }
            break;

        // ... same pattern for ALL states ...
    }
}
```

**Verification rules:**
- **Every** state in the state machine must have a timeout. No state should depend solely on a sensor condition for exit.
- Timeout values: navigation → 15s, obstacle avoidance → 20s, circling → 20s, exploration → 35s, racing → 15s. These are generous but finite.
- On state timeout: transition to `STATE_ERROR` (not `STATE_DONE` — don't silently succeed). `STATE_ERROR` = motors stopped, LED error code blinking, wait for manual intervention.
- The timeout must be per-state-entry (reset timer on each state transition), not a global timer from mission start.
- **Nested timeouts**: if a state has internal phases (like your `Algorithm_AvoidBlackCircles` phases 0-4), the outer state timeout is a separate backstop. If a phase loops back to itself indefinitely, the outer state timeout catches it.

**Corrective action if FAIL:**

For your K题小车, add to `Algorithm_RunMission()`:
```c
// After computing state_elapsed, BEFORE each switch case:
#define STATE_TIMEOUT_NAVIGATE    15000
#define STATE_TIMEOUT_AVOID       20000
#define STATE_TIMEOUT_CIRCLE      25000
#define STATE_TIMEOUT_RACING      15000
#define STATE_TIMEOUT_EXPLORE     35000

// In each case, add after the success condition:
else if (state_elapsed > STATE_TIMEOUT_XXX) {
    Motor_Stop_Safe();
    g_car_state = STATE_ERROR;
}
```

---

### Item 37: Button Debounce by Timer Polling (Not EXTI)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Mechanical button bounce → ISR fires 3–5 times on a single press → startup counter triggers 3 times → car does "start-stop-start" in 20ms → jerks off the starting line or (for balance bot) tips itself over |
| **AI Common Mistake** | `HAL_GPIO_EXTI_Callback` detects falling edge and immediately starts the mission, with no debounce |

**What to check:**

```c
// ❌ CRITICAL FAIL — bare EXTI, no debounce
void EXTI0_IRQHandler(void) {
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_Pin_0)) {
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_Pin_0);
        StartMission();  // Called 3–7 times per button press due to bounce!
    }
}

// ✅ CORRECT — timer interrupt polls button at 10ms, confirms with 3 consistent reads
#define BUTTON_DEBOUNCE_MS     10     // Poll every 10ms
#define BUTTON_CONSISTENT_CNT   3     // 3 consistent readings = confirmed

// NO EXTI for buttons — use a timer-driven poll instead:
static uint8_t button_history[4] = {0xFF, 0xFF, 0xFF, 0xFF};  // 4 buttons, 8 samples each

void Button_Poll_10ms(void) {   // Called from a 100Hz timer ISR or SysTick hook
    uint16_t raw = GPIO_ReadInputData(GPIOB) & 0x000F;  // PB0-PB3 = 4 buttons

    for (int i = 0; i < 4; i++) {
        // Shift history left, insert new sample at LSB
        button_history[i] = (button_history[i] << 1) | ((raw >> i) & 0x01);

        // Check for stable LOW (pressed): last 3 samples all 0
        if ((button_history[i] & 0x07) == 0x00) {
            Button_OnPress(i);  // Confirmed press
            button_history[i] = 0xFF;  // Reset to prevent repeat triggers
        }
        // Check for stable HIGH (released): last 3 samples all 1
        if ((button_history[i] & 0x07) == 0x07) {
            Button_OnRelease(i);  // Confirmed release
        }
    }
}

// Long-press detection (5 seconds):
static uint32_t button_press_times[4] = {0};
static uint8_t  button_long_press_reported[4] = {0};

void Button_OnPress(uint8_t btn_id) {
    button_press_times[btn_id] = GetTick();
    button_long_press_reported[btn_id] = 0;

    if (btn_id == 0) {  // Short press → start/stop
        if (g_car_state == STATE_IDLE) StartMission();
        else Motor_Stop_Safe();
    }
}

void Button_Poll_LongPress(void) {  // Call every 100ms in main loop
    for (int i = 0; i < 4; i++) {
        if (button_press_times[i] > 0 && !button_long_press_reported[i]) {
            if (GetTick() - button_press_times[i] > 5000) {  // 5 seconds
                button_long_press_reported[i] = 1;
                if (i == 0) {  // Button 0 long-press → factory reset
                    PID_Config_LoadDefaults();  // Restore default PID params
                    PID_Config_Save();          // Write to Flash
                    LED_BlinkPattern(0xAA);     // Confirm: fast double-blink
                }
            }
        }
    }
}
```

**Verification rules:**
- Mechanical buttons must NOT use EXTI (external interrupt) for mission-critical functions. EXTI is acceptable only for emergency stop (where false triggers cause safe stops, not dangerous starts).
- Button reading must be done by **timer-driven polling** at 10–20 ms intervals, with **≥3 consecutive consistent readings** required to confirm a state change.
- **Short press** (< 500ms) and **long press** (> 5 seconds) must be distinguished. Long press is the universal escape hatch for parameter recovery.
- After a confirmed press is detected, the history byte must be reset (set to 0xFF) to prevent auto-repeat.
- For DIP switches or coding switches (like your K題小车's ReadMissionSelector): debounce is still needed! Read DIP switches at startup only (before motors enable), read 3 times with 5ms intervals, and take the majority vote. DIP switches can also bounce during handling.

**Corrective action if FAIL:**
```c
// Replace EXTI-based button with timer-polled button:
// 1. Remove HAL_GPIO_EXTI_Callback for button pins
// 2. Add Button_Poll_10ms() call to SysTick_Handler (already at 1ms)
//    or create a dedicated 100Hz timer.
// 3. In SysTick_Handler:
void SysTick_Handler(void) {
    g_sys_tick_ms++;
    static uint8_t tick_div = 0;
    tick_div++;
    if (tick_div >= 10) {  // Every 10ms
        tick_div = 0;
        Button_Poll_10ms();
    }
}
```

---

## Dimension 15: System Startup & Debug Integrity

These items prevent the worst-case scenario: a bricked board or a HardFault with no debugger access, at the competition venue with 30 minutes left on the clock.

---

### Item 38: Stack Overflow Prevention (HardFault Hard Fall)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | `float FilterBuffer[256]` declared as local variable inside function → 1KB on stack → function called from ISR with already-partial stack → stack pointer crosses into heap/global data → registers corrupted → HardFault → motors freeze at last PWM value (Item 10 not implemented → no IWDG → permanent freeze) |
| **AI Common Mistake** | Declares large local arrays, deep function call chains, recursive patterns — all on the Cortex-M's tiny default stack (often 512 bytes) |

**What to check:**

```c
// ❌ CRITICAL FAIL — large local array in function
void ProcessSensorData(void) {
    float temp_buffer[256];   // 256 × 4 = 1024 bytes on stack!
    // Most STM32F103 startup files default Stack_Size = 0x00000400 (1024 bytes)
    // This single array consumes the ENTIRE stack → next function call overwrites heap
    for (int i = 0; i < 256; i++) {
        temp_buffer[i] = ComputeSomething(i);
    }
    // ...
}

// ✅ CORRECT — static or global for large arrays
static float g_sensor_buffer[256];  // Global: allocated in .bss, NOT on stack

void ProcessSensorData(void) {
    for (int i = 0; i < 256; i++) {
        g_sensor_buffer[i] = ComputeSomething(i);  // Safe: no stack consumption
    }
}
```

**Verification rules:**
- **Check startup file**: `startup_stm32f10x_md.s` (or equivalent). `Stack_Size` must be ≥ `0x00000800` (**2KB minimum**) for any motor control project. 8KB (`0x00002000`) recommended if using MPU6050 DMP or floating-point math.
- **Zero-tolerance**: NO local arrays larger than 64 bytes inside any function. Large buffers must be `static` or global.
- **Zero-tolerance for ISRs**: NO local arrays of ANY size in interrupt service routines. The ISR stack is even more constrained (often the Main stack pointer vs Process stack pointer).
- Search for `float ...[...];` or `uint8_t ...[...];` declarations inside function bodies. Flag any with size > 64 as FAIL.
- Recursive functions: NONE allowed on Cortex-M without an RTOS with stack overflow detection. AI sometimes writes recursive pathfinding or tree traversal for obstacle maps → immediate HardFault.
- Stack overflow detection: recommend using the MPU (Memory Protection Unit) to set a guard region below the stack. On overflow → MemManage fault → safe shutdown (better than HardFault corruption). Alternatively, fill the stack with a known pattern at startup and periodically check the high-water mark.

**Corrective action if FAIL:**
```
⚠️ CRITICAL FIX — Stack size check

  1. Open startup_stm32f10x_md.s (your project's startup file).
  2. Find: Stack_Size  EQU  0x00000400 (or similar).
  3. Change to: Stack_Size  EQU  0x00002000 (8KB).
  4. In ALL .c files: move arrays larger than 64 bytes from local scope
     to static or global scope.
  5. In ALL ISR handlers: verify no local arrays exist.
  6. Optional: add stack high-water mark check:
     - Fill stack with 0xDEADBEEF pattern at startup.
     - Periodically scan from stack top → find first non-pattern word.
     - Report "stack used: X / Y bytes" over UART.
```

---

### Item 40: HSE Crystal Frequency Verification

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Board has 12MHz crystal but AI defaults to `HSE_VALUE = 8000000` → PLL configured for 8MHz × 9 = 72MHz → actual frequency = 12MHz × 9 = 108MHz (overclock!) → MCU overheats, PWM frequencies wrong, UART baud rates wrong, timer periods wrong, `Delay_ms(1000)` actually ≈ 667ms |
| **AI Common Mistake** | Never checks the actual board crystal; always assumes 8MHz for STM32F1, 25MHz for STM32F4 |

**What to check:**

```c
// ❌ CRITICAL FAIL — mismatched HSE_VALUE
// In stm32f10x.h (or stm32f4xx_hal_conf.h):
#define HSE_VALUE    ((uint32_t)8000000)   // Assumes 8MHz — is your board 8MHz or 12MHz?

// If your board has a 12MHz crystal:
// - PLL configured as 8MHz × 9 = 72MHz → actually 12 × 9 = 108MHz
// - APB1 = 108/2 = 54MHz (rated for 36MHz max!) → peripheral damage possible
// - TIM3 PWM: 108MHz / 72 / 100 = 15kHz → actually 108MHz / 72 / 100 = 15kHz
//   Wait, this is independent of HSE if PCLK is correct... but PCLK is wrong
// - UART: 108MHz / 115200 = wrong divider → garbage on serial

// ✅ CORRECT — verify and hard-code the actual frequency
// Step 1: Find the crystal marking on your board (look at the metal can near the MCU)
// Step 2: Hard-code in system_stm32f10x.c or stm32f10x.h:
#define HSE_VALUE    ((uint32_t)12000000)   // CONFIRMED: board has 12MHz crystal!

// Step 3: VERIFY with hardware — toggle a GPIO, measure with logic analyzer
void Clock_Verify_Test(void) {
    GPIO_InitTypeDef g;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    g.GPIO_Pin = GPIO_Pin_15;
    g.GPIO_Mode = GPIO_Mode_Out_PP;
    g.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &g);

    while (1) {
        // Toggle at exactly 1kHz if clock is correct
        GPIO_SetBits(GPIOA, GPIO_Pin_15);
        Delay_ms(500);   // 500ms HIGH
        GPIO_ResetBits(GPIOA, GPIO_Pin_15);
        Delay_ms(500);   // 500ms LOW → 1Hz square wave on scope
    }
    // Measure with logic analyzer/oscilloscope:
    //   Period = 1.000s → clock is correct
    //   Period = 0.667s → HSE is 12MHz but HSE_VALUE is 8MHz → fix HSE_VALUE!
}
```

**Verification rules:**
- `HSE_VALUE` must match the **physical crystal on the PCB**, not the chip datasheet default. Read the crystal can marking or measure with an oscilloscope.
- Common traps: STM32F103C8 "Blue Pill" clones often use 12MHz (genuine: 8MHz). STM32F407 Discovery: 8MHz (not 25MHz — the 25MHz is for the ST-Link MCU). ESP32: 40MHz crystal, but most modules have it onboard.
- Always verify with a GPIO toggle test: 500ms HIGH, 500ms LOW → measure period with logic analyzer. If period ≠ 1.000s, the clock config is wrong.
- This also affects the **independent watchdog** (IWDG uses LSI, which is independent — PHEW) but **window watchdog** (WWDG) uses APB1 clock and would be affected.
- On STM32F1 Standard Peripheral Library: the startup file calls `SystemInit()` which sets `SystemCoreClock = 72000000` based on `HSE_VALUE`. If `HSE_VALUE` is wrong, `SystemCoreClock` is wrong, and `SysTick_Config(SystemCoreClock / 1000)` produces wrong 1ms ticks.

**Corrective action if FAIL:**
```
⚠️ CRITICAL FIX — Verify HSE crystal frequency

  1. Read the metal can marking on the crystal next to the MCU.
     Common markings:
     - "8.000" → 8MHz
     - "12.000" → 12MHz
     - "25.000" → 25MHz (STM32F4xx common)

  2. Set HSE_VALUE to the actual frequency in the HAL/StdPeriph config header.

  3. If you've been running with the wrong HSE_VALUE:
     - ALL your PID gains are wrong (dt in I-term is based on wrong tick rate)
     - ALL your encoder speed calculations are wrong
     - ALL your ultrasonic pulse timing is wrong
     → You must re-tune everything after fixing the clock!

  4. Flash the GPIO toggle test firmware and VERIFY with oscilloscope/logic analyzer
     before trusting any timing-critical code.
```

---

### Item 41: AI-Deleted Power-Up Delays (Restore Them)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | AI "optimizes" by deleting `HAL_Delay(100)` after MPU6050 init → MPU6050 hasn't finished internal PLL lock → first I2C read returns garbage → attitude initialized with garbage values → yaw offset = random angle → car drives in wrong direction from frame 1 |
| **AI Common Mistake** | Suggests removing "unnecessary delays" to improve loop speed, not understanding that sensors need power-up settling time |

**What to check:**

```c
// ❌ CRITICAL FAIL — delays deleted by AI "optimization"
void System_Init(void) {
    MPU6050_Init();    // AI deleted the Delay_ms(50) after PWR_MGMT_1 write
    OLED_Init();        // AI deleted the Delay_ms(100) after display init
    Motor_Init();
    // Immediately starts reading sensors → reads garbage → car misbehaves
}

// ✅ CORRECT — mandatory delays with retry-on-fail
void System_Init(void) {
    MPU6050_Init();         // Internal: writes 0x80 to PWR_MGMT_1 → 100ms delay → writes 0x01
    Delay_ms(50);           // MPU6050 PLL lock time (datasheet: 30ms typ, 50ms safe)

    OLED_Init();            // Internal: charge pump needs 100ms to reach 9V for OLED panel
    Delay_ms(100);

    Motor_Init();
    // ADD: verify that initialization actually succeeded
    if (!MPU6050_VerifyInit()) {   // Read WHO_AM_I register (0x75, should be 0x68)
        // Retry up to 3 times
        for (int retry = 0; retry < 3; retry++) {
            Delay_ms(200);
            MPU6050_Init();
            if (MPU6050_VerifyInit()) break;
        }
        if (!MPU6050_VerifyInit()) {
            // MPU6050 dead → enter safe mode, don't run motors with no attitude
            LED_Alarm_Blink();
            while (1) { IWDG_ReloadCounter(); }
        }
    }
}
```

**Verification rules:**
- Power-up delays after peripheral initialization are NOT optional. They exist because the peripherals have physical settling times.
- Key delays that AI likes to delete:
  - MPU6050: 100ms after reset (PWR_MGMT_1[7]=1), 50ms after wake (PWR_MGMT_1=0x01), 30ms after PLL select (PWR_MGMT_1=0x03 for gyro-Z reference)
  - OLED SSD1306: 100ms after VCC stable, 100ms after charge pump enable (0x8D → 0x14)
  - TB6612FNG: 100µs after STBY HIGH before PWM input (AI often confuses this with a "useless delay")
  - HC-SR04: 2ms between triggers (takes time for echo to die out from previous measurement)
- After every peripheral init, add an **init verification read** (read WHO_AM_I register, read a known-status register). If verification fails, retry up to 3 times. If still failing, enter safe mode — do not continue with uninitialized sensors.
- AI-generated code almost never has init verification. This is 100% on you to add.

**Corrective action if FAIL:**
```
⚠️ REQUIRED — Restore deleted delays + add init verification

  Your existing code (mpu6050.c:172-178) already has the correct delays:
  Delay_ms(100) after reset (line 174)
  Delay_ms(50) after wake (line 178)
  → KEEP THESE. Do NOT let AI remove them.

  Add after each init function:
  uint8_t whoami = MPU6050_ReadReg(0x75);
  if (whoami != 0x68) { /* retry or error */ }

  Same for OLED: write a test pixel, read back display status, verify it's on.
```

---

### Item 42: SWD Pin Protection — Don't Let AI Brick Your Board

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | AI configures PA13 (SWDIO) or PA14 (SWCLK) as GPIO_Out_PP to get "one more LED pin" → next flash attempt fails → "No Target Connected" in Keil/IAR → board is BRICKED → must use BOOT0 jumper + reset timing trick to erase Flash |
| **AI Common Mistake** | Treats PA13/PA14 as "free GPIOs" without understanding they're the debug interface |

**What to check:**

```c
// ❌ CRITICAL FAIL — SWD pins repurposed as GPIO
// In any GPIO init function, AI writes:
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13 | GPIO_Pin_14 | GPIO_Pin_15;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
GPIO_Init(GPIOA, &GPIO_InitStructure);
// PA13 and PA14 are now GPIO outputs → debugger disconnected → BRICK!

// ✅ CORRECT — conditional compilation protects SWD during development
// NEVER configure PA13 or PA14 unless explicitly unguarded for production

// Method 1: Exclude SWD pins from GPIO config entirely
void GPIO_Config(void) {
    // PC13: onboard LED (NOT PA13!)
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;  // This is PC13, not PA13!
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOC, &GPIO_InitStructure);
    // PA13 and PA14 are NEVER touched → they remain SWD by default
}

// Method 2: If you MUST reuse SWD pins (only in production build):
#if defined(PRODUCTION_BUILD)
    // Reuse PA13/PA14 as GPIO ONLY if you accept the risk
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);
    GPIO_PinRemapConfig(GPIO_Remap_SWJ_Disable, ENABLE);  // Disable SWJ → full SWJ disable
    // or GPIO_Remap_SWJ_JTAGDisable → disable JTAG, keep SWD
    // Now PA13=GPIO, PA14=GPIO, PA15=GPIO, PB3=GPIO, PB4=GPIO
    // WARNING: from this point, SWD is GONE. Only system bootloader can recover.
#endif

// Method 3: Better debug safety — disable JTAG, keep SWD
// (Frees PB3/PB4/PA15, but keeps PA13/PA14 as SWD)
GPIO_PinRemapConfig(GPIO_Remap_SWJ_JTAGDisable, ENABLE);
// Now you have 3 extra GPIOs (PB3, PB4, PA15) AND still have SWD!
```

**Verification rules:**
- **Absolute rule**: NEVER configure PA13 or PA14 in development firmware. If the code touches these pins, it's a FAIL unless `#ifdef PRODUCTION_BUILD` guards them.
- Search for `GPIO_Pin_13` in GPIOA context (NOT GPIOC). PA13 = SWDIO, PA14 = SWCLK. Many boards have an LED on PC13 (GPIOC Pin 13) — this is fine and completely unrelated to PA13.
- `GPIO_Remap_SWJ_JTAGDisable` is the safe option: it disables only JTAG (frees PB3/PB4/PA15), leaving SWD (PA13/PA14) functional. This gives you 3 extra GPIOs without losing debug access.
- `GPIO_Remap_SWJ_Disable`: disables BOTH JTAG and SWD. Only use in production build, annotated with `// WARNING: SWD permanently disabled`.
- **Recovery procedure** if you accidentally brick a board:
  1. Set BOOT0 jumper to HIGH (connect to 3.3V).
  2. Press and release RESET.
  3. The MCU boots into system bootloader (no user code runs → SWD pins stay as SWD).
  4. Use STM32CubeProgrammer or `st-flash erase` to mass-erase the Flash.
  5. Set BOOT0 back to LOW, press RESET → board is unbricked.

**Corrective action if FAIL:**
```
🔴 CRITICAL — SWD pins being repurposed

  Your K題小车 code (main.c:99, line 154) uses:
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_8 | GPIO_Pin_9 | GPIO_Pin_10 | GPIO_Pin_11;
    → These are PA8-PA11 — SAFE. Not SWD pins. ✓

  BUT verify in the FULL project: search ALL .c files for GPIO_Pin_13 and GPIO_Pin_14
  used with GPIOA (not GPIOC or GPIOB). If found and not guarded by #ifdef → remove.
  
  PC13 (GPIOC Pin 13) ≠ PA13 (GPIOA Pin 13, SWDIO).
  This is the #1 source of confusion: your LED on PC13 is fine,
  but AI may confuse it with PA13 when suggesting "add another LED."
```

---

## Dimension 16: Battery-Aware Control & Motion Decoupling

AI treats PWM% as speed, and treats IMU pitch as physical slope. Neither is true when the vehicle is actually moving.

---

### Item 43: Battery Voltage Feedforward for Speed Mapping

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Fresh battery (8.4V): 50% PWM = 1.0 m/s. Half-discharged (7.0V): 50% PWM = 0.6 m/s. PID tries to compensate → cranks PWM to 85% → current surges → driver overheats → thermal shutdown mid-race |
| **AI Common Mistake** | Assumes `PWM% ∝ Speed` is a hardware constant independent of battery voltage |

**What to check:**

```c
// ❌ CRITICAL FAIL — PWM directly mapped to speed, no voltage compensation
int16_t pwm = (int16_t)PID_Compute(&speed_pid, actual_speed, dt);
Motor_Drive(pwm, steer);  // PWM same regardless of battery at 8.4V or 7.0V

// ✅ CORRECT — battery voltage feedforward compensation
#define BATTERY_NOMINAL_VOLTAGE   8.4f   // 2S LiPo fully charged
#define BATTERY_MIN_VOLTAGE       7.0f   // 2S LiPo safe minimum (Item 5 cutoff)

float Battery_VoltageFeedforward(float pid_output) {
    float v_bat = Battery_ReadVoltage();  // From ADC (Item 5)

    // Safety floor: don't divide by zero or negative
    if (v_bat < 1.0f) v_bat = BATTERY_NOMINAL_VOLTAGE;

    // Feedforward: lower voltage → proportionally higher PWM to maintain same speed
    float ff_gain = BATTERY_NOMINAL_VOLTAGE / v_bat;  // 8.4/7.0 = 1.2× boost
    if (ff_gain > 1.5f) ff_gain = 1.5f;   // Cap at 1.5× to prevent dangerous over-drive

    return pid_output * ff_gain;
}

// In control loop:
float pid_out = PID_Compute(&speed_pid, target_speed_rpm, actual_speed_rpm);
int16_t pwm = (int16_t)Battery_VoltageFeedforward(pid_out);
Motor_Drive(pwm, steer);
```

**Verification rules:**
- Speed control must be in **engineering units** (cm/s or rpm), not raw PWM%. The PID controls speed → the feedforward maps speed to PWM based on battery voltage.
- Feedforward gain must be **capped** (max 1.5× nominal) to prevent the PID from requesting dangerous PWM at critically low voltages.
- The voltage feedforward must run **before** the duty cycle limiter (Item 2: MAX 85%). Combined: `ff_out = pid_out × (V_nominal / V_bat)` → `pwm = constrain(ff_out, -MAX_DUTY, MAX_DUTY)`.
- For systems **without encoder feedback** (like your K题小车 — Item 9 FAIL): battery voltage compensation is even MORE important because there's no speed loop to catch the drift. At minimum, multiply ALL open-loop speed commands by `(V_nominal / V_current)`.

**Corrective action if FAIL:**
```c
// Add to motor.c — voltage-compensated motor drive
static float g_battery_voltage = 8.4f;  // Updated by ADC in main loop
#define BATT_NOMINAL  8.4f

void Motor_Drive_Compensated(int16_t linear, int16_t angular) {
    float v_ratio = BATT_NOMINAL / g_battery_voltage;
    if (v_ratio > 1.5f) v_ratio = 1.5f;   // Safety cap
    if (v_ratio < 0.8f) v_ratio = 0.8f;   // Over-voltage guard (charger connected?)

    int16_t comp_linear  = (int16_t)(linear * v_ratio);
    int16_t comp_angular = (int16_t)(angular * v_ratio);

    Motor_Drive(comp_linear, comp_angular);
}
```

---

### Item 44: Acceleration-Induced Pitch Decoupling

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Hard acceleration → body pitches back 15° → IMU reports 15° incline → AI thinks car is climbing a slope → outputs nose-down correction → car oscillates violently (porpoising) or flips |
| **AI Common Mistake** | Complementary filter always uses a fixed weight (e.g., 0.98 gyro + 0.02 accel) regardless of vehicle motion state |

**What to check:**

```c
// ❌ CRITICAL FAIL — fixed complementary filter weight
pitch = 0.98f * (pitch + gyro_x * dt) + 0.02f * accel_pitch;
// During hard acceleration, the accel_pitch is dominated by inertial tilt (NOT gravity!)
// → filter trusts a corrupted accelerometer reference → pitch estimate drifts

// ✅ CORRECT — dynamic weight based on acceleration magnitude
void Attitude_Update_Dynamic(float dt) {
    // Compute total acceleration magnitude (should be ~1g when static)
    float accel_mag = sqrtf(g_sensors.accel_x * g_sensors.accel_x +
                            g_sensors.accel_y * g_sensors.accel_y +
                            g_sensors.accel_z * g_sensors.accel_z);

    // Dynamic weight: when accel_mag deviates from 1g, reduce accel trust
    float accel_error = fabsf(accel_mag - 1.0f);  // 0 = perfect static, >0 = moving
    float accel_weight = 0.02f;  // Base: 2% accel weight when static

    if (accel_error > 0.05f) {  // > 0.05g deviation → vehicle accelerating/braking
        // Reduce accel weight proportionally to acceleration
        accel_weight = 0.02f * (1.0f - fminf(accel_error / 0.5f, 1.0f));
        // At 0.5g acceleration: weight = 0.02 × (1 - 1.0) = 0 → pure gyro integration
    }

    float gyro_weight = 1.0f - accel_weight;

    // Complementary filter with dynamic weights
    pitch = gyro_weight * (pitch + g_sensors.gyro_x * dt)
          + accel_weight * atan2f(g_sensors.accel_y, g_sensors.accel_z) * 57.29578f;

    roll  = gyro_weight * (roll + g_sensors.gyro_y * dt)
          + accel_weight * atan2f(g_sensors.accel_x, g_sensors.accel_z) * 57.29578f;
}
```

**Verification rules:**
- The complementary filter weight for the accelerometer must be **dynamic**, not a hardcoded `0.98/0.02`.
- When `|accel_magnitude - 1.0g| > threshold` (typically 0.05g–0.10g), reduce accelerometer weight toward zero.
- At high acceleration/deceleration (>0.5g deviation), the accelerometer weight should be effectively zero — attitude relies purely on gyroscope integration, which drifts but doesn't have the systematic motion-induced error.
- The dynamic weight formula in your existing code (`mpu6050.c:243-244`) uses fixed 0.98/0.02 weights → this is acceptable for slow-moving robots but will cause problems at competition speeds.
- **Special case — balancing robot**: the balance controller ITSELF depends on accurate pitch. Acceleration-induced pitch error creates a **positive feedback loop** (pitch error → motor acceleration → more pitch error). For balance bots, this item is NOT optional — it's CRITICAL.

**Corrective action if FAIL:**
```c
// Replace fixed weights in mpu6050.c:243-244 with dynamic weights:
float accel_mag = sqrtf(sensor->accel_x * sensor->accel_x +
                        sensor->accel_y * sensor->accel_y +
                        sensor->accel_z * sensor->accel_z);
float motion_factor = fminf(fabsf(accel_mag - 1.0f) / 0.5f, 1.0f);
float gyro_w  = 0.98f + 0.02f * motion_factor;   // 0.98→1.00 as motion increases
float accel_w = 1.0f - gyro_w;                     // 0.02→0.00 as motion increases

pitch = gyro_w * (pitch + sensor->gyro_x * dt) + accel_w * acc_pitch;
roll  = gyro_w * (roll  + sensor->gyro_y * dt) + accel_w * acc_roll;
```

---

### Item 45: Wheel Slip Odometry Rejection

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Sharp turn on dusty floor → wheels spin in place → encoder counts 200 pulses (thinks it moved 1m) → actual movement = 0.6m → odometry drifts → autonomous parking crashes into wall |
| **AI Common Mistake** | `distance = encoder_count × WHEEL_CIRCUMFERENCE / PPR` — unconditionally trusts encoder data even during slip |
| **Trigger Condition** | **Check ONLY if the system has wheel encoders and uses them for odometry/dead-reckoning. Skip if the vehicle has no encoders (pure open-loop or pure line-tracking).** |

**What to check:**

```c
// ❌ CRITICAL FAIL — blind encoder integration
float odom_x = 0, odom_y = 0;
void UpdateOdometry(void) {
    int16_t left_delta  = encoder_left_count  - encoder_left_prev;
    int16_t right_delta = encoder_right_count - encoder_right_prev;
    float distance = (left_delta + right_delta) * 0.5f * WHEEL_CIRCUMFERENCE / PPR;
    // Distance accumulated even during wheel slip → position estimate drifts
    odom_x += distance * cosf(heading);
    odom_y += distance * sinf(heading);
}

// ✅ CORRECT — IMU-assisted slip detection + confidence-weighted integration
void UpdateOdometry_Safe(void) {
    int16_t left_delta  = encoder_left_count  - encoder_left_prev;
    int16_t right_delta = encoder_right_count - encoder_right_prev;
    float encoder_distance = (left_delta + right_delta) * 0.5f * WHEEL_CIRCUMFERENCE / PPR;

    // Slip detection: check IMU for anomalous motion
    float accel_mag = sqrtf(g_sensors.accel_x * g_sensors.accel_x +
                            g_sensors.accel_y * g_sensors.accel_y +
                            g_sensors.accel_z * g_sensors.accel_z);

    uint8_t is_slipping = 0;
    float slip_confidence = 1.0f;  // 1.0 = full trust, 0.0 = ignore encoder

    // Condition 1: Acceleration anomaly (wheel spin → high accel but no real movement)
    if (fabsf(accel_mag - 1.0f) > 0.2f) {  // > 0.2g deviation from gravity
        slip_confidence = 0.3f;  // Only 30% trust in encoder during anomalous acceleration
        is_slipping = 1;
    }

    // Condition 2: High angular rate (sharp turn → likely slip)
    if (fabsf(g_sensors.gyro_z) > 300.0f) {  // > 300 deg/s
        slip_confidence = 0.3f;
        is_slipping = 1;
    }

    // Condition 3: Encoder delta vs IMU delta mismatch
    // (Encoder says 100cm/s but IMU says no acceleration → slip)
    static float prev_speed = 0;
    float encoder_speed = encoder_distance / dt;
    float imu_speed_est = prev_speed + g_sensors.accel_x * dt * 100.0f;  // cm/s
    if (fabsf(encoder_speed - imu_speed_est) > encoder_speed * 0.3f) {  // >30% mismatch
        slip_confidence = 0.2f;  // Heavy slip → mostly ignore encoder
        is_slipping = 1;
    }
    prev_speed = encoder_speed * slip_confidence + imu_speed_est * (1.0f - slip_confidence);

    // Apply slip confidence to odometry update
    float trusted_distance = encoder_distance * slip_confidence;
    odom_x += trusted_distance * cosf(heading);
    odom_y += trusted_distance * sinf(heading);

    if (is_slipping) {
        g_odom.slip_flag = 1;  // Mark odometry as degraded for higher-level planning
    }
}
```

**Verification rules:**
- Encoder odometry must NOT be unconditionally integrated. A slip detection function must gate the integration.
- Three slip indicators: (a) acceleration magnitude deviates from 1g, (b) gyro Z exceeds cornering threshold, (c) encoder-derived speed mismatches IMU-derived acceleration.
- When slip is detected: reduce encoder trust weight (0.2–0.3), increase IMU dead-reckoning weight (0.7–0.8). IMU-only dead reckoning drifts but doesn't have systematic slip bias.
- The `slip_flag` must propagate to higher-level planning: if the robot knows its odometry is degraded, it can slow down or use other localization sources (landmarks, lines).
- **Important**: slip detection is a band-aid. The real fix is mechanical (better tires, lower CG, slower turns). Flag the physical issue in the review report.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Encoder slip detection

  Your K题小车 currently has no encoder reading code (Item 9 FAIL).
  If encoders are added later, the odometry update MUST include:
  1. IMU acceleration magnitude check: |accel - 1g| > 0.2 → slip likely
  2. Gyro Z threshold: |gyro_z| > 300 dps → sharp turn → slip likely
  3. Encoder-vs-IMU speed mismatch check: >30% difference → slip confirmed
  4. Slip confidence multiplier (0.0–1.0) on encoder distance accumulation
```

---

### Item 46: Chassis Kinematics Model Verification

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Ackermann chassis gets differential-drive code → rear wheels fight each other → tires leave black marks on floor → rear motor driver overheats and burns within minutes |
| **AI Common Mistake** | Universally applies `V_left = V - W, V_right = V + W` to every robot regardless of chassis type |

**What to check:**

```c
// ❌ CRITICAL FAIL — differential model applied to Ackermann chassis
void Motor_Drive(int16_t linear, int16_t angular) {
    int16_t left  = linear - angular;   // Rear left gets DIFFERENT speed from rear right!
    int16_t right = linear + angular;   // But rear axle is mechanically linked in Ackermann!
    Motor_Left_Set(left);    // Rear left wheel
    Motor_Right_Set(right);  // Rear right wheel → fighting each other!
    Servo_Set(angular * 0.5f);  // Front steering
    // RESULT: rear tires scrub, motor current spikes, driver burns
}

// ✅ CORRECT — chassis-type-specific kinematics
// OPTION A: Differential Drive (2WD, 4WD skid-steer)
void Motor_Drive_Differential(int16_t linear, int16_t angular) {
    int16_t left  = linear - angular;
    int16_t right = linear + angular;
    // Standard differential: both sides independently driven
    Motor_Left_Set(left);
    Motor_Right_Set(right);
}

// OPTION B: Ackermann (front steering, rear single-speed or locked diff)
void Motor_Drive_Ackermann(int16_t linear, int16_t angular) {
    // Rear: BOTH wheels SAME speed (mechanically linked via differential or solid axle)
    Motor_Left_Set(linear);   // Rear left
    Motor_Right_Set(linear);  // Rear right — IDENTICAL to left!
    // Front: steering servo
    Servo_SetAngle(90 + angular);  // 90° = center, angular = steer offset
}

// OPTION C: Mecanum/Omni (see Item 29 for sign verification)
void Motor_Drive_Mecanum(float vx, float vy, float omega) {
    // Must match wheel orientation (A/B type) — see Item 29
    float v_fl =  vx - vy - omega * (Lx + Ly);  // Verify signs by physical test!
    float v_fr =  vx + vy + omega * (Lx + Ly);
    float v_rr =  vx + vy - omega * (Lx + Ly);
    float v_rl =  vx - vy + omega * (Lx + Ly);
    // ...
}
```

**Verification rules:**
- **Before writing a single line of motor control code**, identify the chassis type. The three common types:
  - **Differential (skid-steer)**: 2 or 4 motors, each side independently driven. Turning = different speeds per side. Most common in small competition robots (including your K题小车).
  - **Ackermann**: 1 rear motor (or 2 driven identically) + 1 front steering servo. Turning = servo changes front wheel angle, rear wheels go same speed.
  - **Mecanum/Omni**: 4 motors with rollers at 45°. Translation in any direction. Requires inverse kinematics matrix.
- For **Ackermann**: `Motor_Left_Set(speed)` and `Motor_Right_Set(speed)` must receive the **same value**. If they differ by more than 5%, it's a FAIL.
- For **differential**: verify that the `linear ± angular` formula doesn't cause inner wheel reversal at max steer (already covered by Item 24's coupling).
- This item may seem obvious, but AI code generators do NOT ask "what chassis is this?" — they blindly copy the first motor control template they find online (usually differential).

**Corrective action if FAIL:**
```
⚠️ CRITICAL — Chassis kinematics model mismatch

  Step 1: Physically look at your robot. Count:
  - How many drive motors? (exclude steering servo)
  - Do rear wheels turn at different speeds in a turn? (differential)
    Or do they always turn together? (Ackermann with locked diff)
  - Are the wheels normal or with angled rollers? (mecanum)

  Step 2: Match the kinematics:
  - 2 motors (left/right independent) → differential model
  - 1 drive motor + 1 steering servo → Ackermann
  - 4 mecanum wheels → Mecanum inverse kinematics (Item 29 + verify orientation)

  Step 3: For your K题小车:
  - TB6612FNG + 2× N20 motors controlling left/right sides independently
  - This IS differential drive → your current Motor_Drive() is correct ✓
  - But verify no AI accidentally added servo steering code!
```

---

## Dimension 17: Multi-Sensor Fusion & Competition Logic

AI fuses sensors as if they all report data from the same instant, and treats competition start/finish lines as simple threshold triggers. Neither survives real-world latency or edge cases.

---

### Item 47: Visual-IMU Temporal Alignment

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Camera exposure + transfer = 30ms latency. IMU read = 0.5ms. AI fuses "30ms-old image error" with "current IMU angle" → control loop drives with 30ms of phase lag → at 2m/s, that's 6cm of position error → high-speed cornering overshoots by 6cm → misses the apex → runs off track |
| **AI Common Mistake** | Reads camera and IMU sequentially in main loop and feeds them into the same fusion algorithm without timestamps |
| **Trigger Condition** | **Check ONLY if the system fuses camera/vision data with IMU data. Skip if the vehicle uses only one sensor type for navigation (pure line-tracking, pure IMU, or pure encoder).** |

**What to check:**

```c
// ❌ CRITICAL FAIL — sensor fusion with unaligned timestamps
void MainLoop(void) {
    Camera_ReadLine(&line_pos);      // Image captured ~30ms ago!
    MPU6050_Update(&g_sensors);      // Current IMU data (0.5ms old)
    float fused_angle = Fuse(line_pos, g_sensors.yaw);  // 30ms time mismatch!
    Motor_Drive(speed, fused_angle);
}

// ✅ CORRECT — timestamp-aware fusion with IMU back-extrapolation
typedef struct {
    float value;
    uint32_t timestamp_ms;   // When was this data captured?
} TimestampedData_t;

static TimestampedData_t g_camera_data;
static TimestampedData_t g_imu_yaw_history[10];  // Rolling buffer of recent IMU data
static uint8_t g_imu_hist_idx = 0;

// Camera ISR or DMA callback: record capture timestamp
void Camera_FrameReady_Callback(float line_position) {
    g_camera_data.value = line_position;
    g_camera_data.timestamp_ms = GetTick();  // Timestamp of image CAPTURE, not processing
}

// IMU update: store timestamped yaw history for back-extrapolation
void MPU6050_Update_Timestamped(SensorData_t *sensor) {
    // ... read IMU ...
    sensor->yaw = g_mpu_internal_yaw - g_yaw_offset;
    sensor->timestamp_ms = GetTick();

    // Store in rolling history
    g_imu_yaw_history[g_imu_hist_idx].value = sensor->yaw;
    g_imu_yaw_history[g_imu_hist_idx].timestamp_ms = sensor->timestamp_ms;
    g_imu_hist_idx = (g_imu_hist_idx + 1) % 10;
}

// Fusion: align IMU to camera timestamp by angular integration
float Fuse_Aligned(void) {
    float camera_yaw_error = g_camera_data.value;    // Line offset at camera timestamp
    uint32_t camera_ts = g_camera_data.timestamp_ms;

    // Find IMU reading closest to (but before) camera timestamp
    float imu_yaw_at_camera_time = g_sensors.yaw;
    for (int i = 0; i < 10; i++) {
        uint8_t idx = (g_imu_hist_idx - 1 - i + 10) % 10;
        if (g_imu_yaw_history[idx].timestamp_ms <= camera_ts) {
            // Found IMU reading at or just before camera time
            imu_yaw_at_camera_time = g_imu_yaw_history[idx].value;
            // Integrate forward from that IMU reading to camera timestamp
            float dt = (camera_ts - g_imu_yaw_history[idx].timestamp_ms) / 1000.0f;
            imu_yaw_at_camera_time += g_sensors.gyro_z * dt;  // Angular integration
            break;
        }
    }

    // Now camera_yaw_error and imu_yaw_at_camera_time are from the SAME moment
    return camera_yaw_error - imu_yaw_at_camera_time;
}
```

**Verification rules:**
- Every sensor data structure that feeds into fusion must carry a **timestamp** field (uint32_t in ms, from system tick).
- Camera/vision data must be timestamped at **frame capture time** (when the image sensor exposure ends), not at processing-complete time.
- When fusing camera data (30ms latency) with IMU data (sub-ms latency): the IMU must be **back-extrapolated** via angular integration to the camera's timestamp.
- Without timestamp alignment, the control loop's effective phase margin is reduced by `ω × latency`, where ω is the control bandwidth. A 50Hz control loop with 30ms latency has 540° of phase lag — it's unstable by definition.
- Rolling buffer size: keep 10–20 recent IMU samples. At 200Hz, 20 samples = 100ms of history, enough to cover typical camera latency (30–60ms).

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Sensor timestamp alignment

  If your K题小车 uses ONLY line tracking (gray sensors) with no camera+IMU fusion:
  mark this item as SKIP. Line sensors have negligible latency (~1ms ADC read).

  If you add a camera (OpenMV, K210, etc.) later:
  1. Camera must output a timestamp with each frame.
  2. IMU must maintain a rolling history buffer (10-20 samples).
  3. Fusion function must find the IMU value at camera timestamp by back-integration.
  4. Without this, your "high-speed tracking" will overshoot every turn.
```

---

### Item 48: Start/Finish Line False Trigger Prevention

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Start line: wheel slips on start → car twitches forward 1cm → line sensor crosses start line → AI thinks "lap complete" → car stops immediately 1cm from start. Finish line: car crosses finish at an angle → only one wheel touches the line → line sensor doesn't trigger → car keeps going full speed into the wall |
| **AI Common Mistake** | `if (gray_sensor == BLACK) { stop(); }` — a single sensor reading triggers a race-critical event with no debounce, no confirmation, no minimum distance guard |
| **Trigger Condition** | **Check if the vehicle has start/finish line detection logic for competition. Skip for purely exploratory or non-racing robots.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — single-trigger start/finish detection
void CheckFinishLine(void) {
    if (g_sensors.gray_binary[2] == 0) {  // Center sensor sees black
        Motor_Stop();  // Immediate stop on single reading! False trigger guaranteed.
        g_car_state = STATE_DONE;
    }
}

// ✅ CORRECT — multi-condition confirmation with minimum distance guard
#define FINISH_LINE_MIN_DISTANCE_CM   190.0f  // Must have traveled >95% of lap
#define FINISH_LINE_CONSECUTIVE_MS     50     // Black line must be seen for 50ms continuous
#define START_MIN_STATIC_MS           500     // Must be stationary for 500ms before start

// === START detection ===
static uint8_t  start_confirmed = 0;
static uint32_t static_start_time = 0;
static uint32_t button_press_time = 0;

uint8_t CheckStartCondition(void) {
    // Condition 1: Button pressed (debounced — Item 37)
    uint8_t button_ok = Button_IsPressed(START_BUTTON);

    // Condition 2: Vehicle stationary (accel variance < threshold for 500ms)
    uint8_t static_ok = 0;
    if (IsVehicleStatic()) {  // Reuse Item 31's static detection
        if (static_start_time == 0) static_start_time = GetTick();
        if (GetTick() - static_start_time > START_MIN_STATIC_MS) static_ok = 1;
    } else {
        static_start_time = 0;
    }

    // Condition 3: Start line detected (gray sensor sees black line)
    uint8_t line_ok = 0;
    static uint32_t line_seen_time = 0;
    if (Gray_Sensor_OnLine()) {  // ≥3 of 5 sensors see black
        if (line_seen_time == 0) line_seen_time = GetTick();
        if (GetTick() - line_seen_time > 50) line_ok = 1;  // 50ms continuous
    } else {
        line_seen_time = 0;
    }

    // ALL THREE must be true simultaneously
    return button_ok && static_ok && line_ok;
}

// === FINISH detection ===
static uint8_t finish_deadbolt = 0;  // Once triggered, never re-trigger

uint8_t CheckFinishCondition(void) {
    if (finish_deadbolt) return 0;

    // Condition 1: Minimum distance traveled (>95% of lap length)
    float distance_traveled = g_odom.pos_x_cm;  // Or encoder-based estimate
    if (distance_traveled < FINISH_LINE_MIN_DISTANCE_CM) return 0;

    // Condition 2: Finish line detected for 50ms continuous
    static uint32_t finish_seen_time = 0;
    if (Gray_Sensor_OnLine()) {
        if (finish_seen_time == 0) finish_seen_time = GetTick();
        if (GetTick() - finish_seen_time > FINISH_LINE_CONSECUTIVE_MS) {
            finish_deadbolt = 1;  // Lock — never trigger again
            return 1;
        }
    } else {
        finish_seen_time = 0;
    }
    return 0;
}
```

**Verification rules:**
- **Start**: must satisfy ALL of: (1) debounced button press, (2) vehicle stationary ≥ 500ms, (3) start line detected. Individual wheel slip cannot trigger it.
- **Finish**: must satisfy ALL of: (1) distance traveled ≥ 95% of lap length, (2) line detected for ≥ 50ms continuous, (3) deadbolt flag not already set.
- The **deadbolt flag** (`finish_deadbolt`) is critical: once finish is triggered, the flag is set and never cleared (except by explicit reset before next race). This prevents double-counting on multi-lap courses or false re-triggers during post-race movement.
- For **multi-lap** races: use a lap counter instead of a boolean deadbolt. Increment lap count only when: distance > (lap_length × 0.95 × lap_count) AND line detected for 50ms.
- Line detection must use **multiple sensors** (≥3 of 5 for a 5-sensor array), not a single sensor that might miss the line if the car crosses at an angle.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Competition start/finish logic

  Your K题小车 currently has NO explicit start/finish detection in the code.
  The mission starts via ReadMissionSelector() + 3-2-1 LED countdown in main().
  The mission ends via ultrasonic distance < 20cm or elapsed time > 10s.

  For competition reliability, add:
  1. START: button + static 500ms + line detected
  2. FINISH: distance > 95% lap + line 50ms continuous + deadbolt flag
  3. FALSE START PREVENTION: ignore line triggers for first 50cm of travel
  4. FINISH DEADBOLT: once triggered, disables line detection permanently
```

---

### Item 49: Surface Friction Online Identification

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | PID tuned on slippery tile → competition uses high-friction blue carpet → turn radius shrinks by 30% → car understeers into outer wall; or PID tuned on carpet → qualification on polished concrete → snap oversteer → car spins out |
| **AI Common Mistake** | PID gains are compile-time constants. AI never mentions that different surfaces need different gains. |
| **Trigger Condition** | **This is a performance optimization item, not a strict safety item. Check if the competition involves varying floor surfaces. Still flag it as ⚠️ RECOMMENDATION even when skipped.** |

**What to check:**

```c
// ❌ FAIL — hardcoded PID gains, same for all surfaces
#define TURN_KP   3.0f    // Tuned on one surface, assumed universal
#define TURN_KI   0.1f

// ✅ CORRECT — online friction identification + adaptive gain
static float g_surface_friction_est = 1.0f;  // 1.0 = nominal, >1 = high friction, <1 = low

void SurfaceFriction_Identify(void) {
    // Method: apply a known steer PWM step, measure turn rate response
    static uint32_t test_start_time = 0;
    static float test_start_yaw = 0;
    static uint8_t test_active = 0;
    static float test_steer_pwm = 25.0f;  // Test: 25% steer

    // Run identification at startup (during pre-race countdown or first straight segment)
    if (!test_active) {
        test_active = 1;
        test_start_time = GetTick();
        test_start_yaw = g_sensors.yaw;
        Motor_Drive(30, test_steer_pwm);  // Low speed, known steer
    }

    if (test_active && (GetTick() - test_start_time > 300)) {  // 300ms test pulse
        float yaw_change = g_sensors.yaw - test_start_yaw;
        float turn_rate = yaw_change / 0.3f;  // deg/s for the given steer PWM

        // Nominal response (from calibration on reference surface): 25% steer → 45°/s
        // Current response: turn_rate
        // Friction factor = actual / nominal
        float nominal_rate = 45.0f;  // Calibrated on reference surface
        g_surface_friction_est = turn_rate / nominal_rate;

        // Clamp to sane range
        if (g_surface_friction_est > 2.0f)  g_surface_friction_est = 2.0f;
        if (g_surface_friction_est < 0.3f)  g_surface_friction_est = 0.3f;

        test_active = 0;

        // Apply adaptation: scale PID gains by inverse of friction
        // High friction (2.0) → reduce Kp (1.5×) to avoid over-correction
        // Low friction (0.5) → increase Kp (0.75× wait that's wrong)
        // Actually: high friction → more responsive → REDUCE Kp to avoid overshoot
        float gain_scale = 1.0f / g_surface_friction_est;
        if (gain_scale > 2.0f) gain_scale = 2.0f;
        if (gain_scale < 0.5f) gain_scale = 0.5f;

        g_heading_pid.Kp = HEADING_KP_NOMINAL * gain_scale;  // Item 34: use bumpless!
        g_heading_pid.Ki = HEADING_KI_NOMINAL * gain_scale;

        // Log: "Surface friction: %.2f, PID gain scale: %.2f", g_surface_friction_est, gain_scale
    }
}
```

**Verification rules:**
- PID gains tuned on one surface WILL fail on a different surface. This is not a maybe — it's a certainty.
- Online identification: apply a fixed steer PWM step at low speed, measure the achieved turn rate. Compare to a known "reference surface" calibration. Ratio = friction multiplier.
- Gain adaptation: `Kp_new = Kp_nominal / friction_ratio`. High friction → vehicle turns MORE for same PWM → reduce Kp to avoid oversteer. Low friction → vehicle turns LESS → increase Kp (within stability limits) to maintain tracking.
- The identification should run **once at race start** (during the 3-2-1 countdown or the first 300ms of straight motion). It doesn't need to run continuously.
- For servo-based steering (Ackermann): the adaptation adjusts servo angle feedforward instead of differential motor gains. Same principle, different output.
- This item is a ⚠️ RECOMMENDATION level (not safety-critical CRITICAL). It's about competitive performance rather than hardware destruction. Still flag it because AI will NEVER suggest this.

**Corrective action if FAIL:**
```
⚠️ RECOMMENDATION — Surface friction adaptation

  Your K题小车 uses hardcoded PID gains in algorithm.c:37-43:
  - Speed PID:  Kp=2.0, Ki=0.5, Kd=0.1
  - Heading PID: Kp=3.0, Ki=0.1, Kd=0.5
  These were presumably tuned on one specific surface.

  For competition reliability:
  1. Run a surface identification test during the 3-2-1 LED countdown.
  2. Apply 25% steer for 300ms, measure achieved turn rate.
  3. Scale heading PID gains by (nominal_rate / measured_rate).
  4. Use bumpless transfer (Item 34) when applying new gains.
  5. If surface changes mid-race (unlikely in 电赛): re-identify during straight segments.

  IMPORTANT: test this on MULTIPLE surfaces before competition day.
  An incorrectly tuned adaptive gain is worse than a fixed gain.
```

---

## Dimension 18: Competition Field Survival & Pre-Race Lockdown

These items are invisible on the lab bench. They only strike on competition day — under venue lighting, in a sea of 2.4GHz interference, with a battery that's been cycled 20 times. AI has zero awareness of any of this.

---

### Item 50: Dynamic Load Voltage Sag Detection

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | LiPo resting voltage = 7.2V (looks fine). Hard acceleration → motor draws 3A → voltage sags to 5.5V for 50ms → MCU brownout reset → GPIOs float momentarily → TB6612 inputs glitch → car lurches backward at full speed → hits a judge |
| **AI Common Mistake** | `if (voltage < 6.5V) stop();` — samples voltage once every 100ms during idle moments, completely missing the load-induced sag that triggers brownout |

**What to check:**

```c
// ❌ CRITICAL FAIL — static voltage check, misses load sag
float v_bat = Battery_ReadVoltage();  // Sampled at idle → 7.2V, looks fine!
if (v_bat < 7.0f) Motor_Disable();    // Never triggers before brownout happens

// ✅ CORRECT — dual-threshold: static low + dynamic sag
#define BATT_STATIC_LOW_2S      7.0f   // Resting voltage low threshold (existing Item 5)
#define BATT_SAG_THRESHOLD_2S   0.8f   // Max allowed sag under load
#define BATT_SAG_SAMPLE_WINDOW  5      // Compare 5 samples (~25ms at 200Hz)

static float g_batt_voltage_history[BATT_SAG_SAMPLE_WINDOW];
static uint8_t g_batt_hist_idx = 0;
static float g_batt_min_under_load = 99.0f;

// Must be called at HIGH control rate (≥200Hz) during motor drive
void Battery_Monitor_Dynamic(void) {
    float v_now = Battery_ReadVoltage();

    // Rolling history for sag detection
    g_batt_voltage_history[g_batt_hist_idx] = v_now;
    g_batt_hist_idx = (g_batt_hist_idx + 1) % BATT_SAG_SAMPLE_WINDOW;

    // Track minimum voltage seen under load
    int16_t motor_pwm = Motor_GetCurrentDutyLeft() + Motor_GetCurrentDutyRight();
    if (abs(motor_pwm) > 20) {  // Under significant load
        if (v_now < g_batt_min_under_load) g_batt_min_under_load = v_now;
    }

    // Check 1: Static low voltage (existing Item 5)
    if (Motor_IsEnabled() && v_now < BATT_STATIC_LOW_2S) {
        Motor_Disable_Immediate();
        LED_Alarm_LowBattery();
        return;
    }

    // Check 2: Dynamic sag detection ← THIS IS NEW
    // Compare current voltage to the average of recent samples
    float avg_recent = 0;
    for (int i = 0; i < BATT_SAG_SAMPLE_WINDOW; i++) {
        avg_recent += g_batt_voltage_history[i];
    }
    avg_recent /= BATT_SAG_SAMPLE_WINDOW;

    float sag = avg_recent - v_now;  // How much did voltage drop in this window?
    if (sag > BATT_SAG_THRESHOLD_2S) {
        // Voltage is sagging hard → limit acceleration to prevent brownout
        g_max_accel_limit = g_max_accel_limit * 0.5f;   // Halve allowed acceleration
        g_batt_sag_warning = 1;
        // The next acceleration demand (Item 30's ramp) will be capped
    } else if (sag < 0.2f && g_batt_sag_warning) {
        // Sag recovered → gradually restore acceleration
        g_max_accel_limit = fminf(g_max_accel_limit * 1.1f, MAX_ACCEL_NOMINAL);
        if (g_max_accel_limit >= MAX_ACCEL_NOMINAL * 0.95f) {
            g_batt_sag_warning = 0;  // Fully recovered
        }
    }

    // Check 3: Coulomb counting (mAh-based SOC estimation)
    // Integrate current draw over time to estimate remaining capacity
    static float g_mah_consumed = 0;
    float current_est = fabsf(motor_pwm) * MAX_CURRENT_AMPS / 100.0f;  // Approximate
    g_mah_consumed += current_est * (1000.0f / 200.0f) / 3600.0f;  // mAh per 5ms cycle @ 200Hz

    // If cumulative consumption exceeds 80% of rated capacity → treat as "low"
    if (g_mah_consumed > BATTERY_CAPACITY_MAH * 0.8f) {
        g_batt_low_by_coulomb = 1;
        // Reduce max speed, flash warning LED, but don't disable motors yet
    }

    // Reset coulomb counter on fresh battery detection (voltage spike after swap)
    if (v_now > BATT_STATIC_LOW_2S + 1.5f && g_batt_low_by_coulomb) {
        g_mah_consumed = 0;
        g_batt_low_by_coulomb = 0;
    }
}
```

**Verification rules:**
- Voltage monitoring must run at the **control loop rate** (≥200 Hz), not at a slow "health check" rate (1 Hz). Load-induced sags last 20–80ms and are invisible to slow sampling.
- **Dual threshold**: (a) static low-voltage cutoff (Item 5, resting voltage < 7.0V → motors off), (b) dynamic sag detection (voltage drops > 0.8V in 25ms window → limit acceleration).
- Sag response must be **proportional and reversible**: limit acceleration (not cut motors entirely). The sag is temporary (PWM duty drops after acceleration ends → voltage recovers). Only static low voltage should cut motors permanently.
- **Coulomb counting** (mAh integration) provides a second independent SOC estimate. Voltage alone is misleading for LiPo (flat discharge curve). When mAh consumed > 80% of rated capacity: warn, reduce max speed, but don't stop.
- Battery capacity degrades over cycles. A "2000mAh" battery after 50 cycles might only deliver 1600mAh. Use a conservative estimate: `BATTERY_CAPACITY_MAH = RATED_MAH * 0.7`.

**Corrective action if FAIL:**
```c
// Add to main.h:
typedef struct {
    float static_low_threshold;   // 7.0V for 2S
    float sag_threshold;           // 0.8V max allowed sag
    float mah_consumed;            // Cumulative discharge
    float mah_capacity;            // Conservative capacity estimate
    float max_accel_current;       // Current acceleration limit (dynamic)
    uint8_t sag_warning;           // Currently in sag condition
    uint8_t low_by_coulomb;        // Low SOC by coulomb counting
    float voltage_history[5];      // Rolling window for sag detection
} BatteryMonitor_t;

// Call in main loop at control rate, BEFORE Motor_Drive():
// Battery_Monitor_Dynamic();
// Motor_Drive(linear * g_batt.max_accel_current / MAX_ACCEL_NOMINAL, angular);
```

---

### Item 51: Dynamic Lighting Adaptive Threshold (OTSU)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Venue sunlight slants through windows at 10 AM → half the track is 3× brighter than the other half → AI's fixed threshold (calibrated at 8 AM) sees the dark half as "all black line" → car veers off at the light/dark boundary at full speed |
| **AI Common Mistake** | Hardcodes a grayscale threshold at init time (e.g., `#define LINE_THRESHOLD 500`), or auto-calibrates once at boot and never updates |
| **Trigger Condition** | **Check ONLY if the system uses camera-based or grayscale-array line tracking. Skip for pure IR-sensor or pure magnetic-tape tracking.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — fixed threshold, calibrated once at boot
#define LINE_THRESHOLD  500    // "Tuned in the lab at 8 PM under fluorescent lights"
void Gray_Sensor_Read(void) {
    for (int i = 0; i < 5; i++) {
        g_sensors.gray_binary[i] = (g_sensors.gray_raw[i] < LINE_THRESHOLD) ? 0 : 1;
    }
}

// ✅ CORRECT — dynamic OTSU or min-max adaptive threshold, updated continuously
#define ADAPTIVE_THRESH_UPDATE_MS   100   // Update threshold every 100ms

static uint8_t g_adaptive_threshold = 128;  // Starts at mid-scale, adapts automatically

void Gray_Sensor_AdaptiveThreshold(void) {
    static uint32_t last_update = 0;
    if (GetTick() - last_update < ADAPTIVE_THRESH_UPDATE_MS) return;
    last_update = GetTick();

    // Method 1: Min-Max adaptive (fast, good enough for most cases)
    uint16_t min_val = 4095, max_val = 0;
    for (int i = 0; i < GRAY_SENSOR_NUM; i++) {
        if (g_sensors.gray_raw[i] < min_val) min_val = g_sensors.gray_raw[i];
        if (g_sensors.gray_raw[i] > max_val) max_val = g_sensors.gray_raw[i];
    }

    uint16_t spread = max_val - min_val;
    if (spread > 100) {  // Valid contrast exists
        // Threshold = midpoint between min and max (adapts to ambient brightness)
        g_adaptive_threshold = (min_val + max_val) / 2;
    }
    // If spread < 100, keep previous threshold (sensor might be over all-white or all-black surface)

    // Method 2: Full OTSU (more robust, computationally heavier — use for camera)
    // uint8_t otsu = OTSU_Compute(g_sensors.gray_raw, GRAY_SENSOR_NUM, 256);
    // g_adaptive_threshold = otsu;

    // Apply adaptive threshold
    for (int i = 0; i < GRAY_SENSOR_NUM; i++) {
        g_sensors.gray_binary[i] = (g_sensors.gray_raw[i] < g_adaptive_threshold) ? 0 : 1;
    }
}

// Optional: Full OTSU implementation for camera or multi-level grayscale
static uint8_t OTSU_Compute(const uint16_t *data, int len, int bins) {
    int histogram[256] = {0};
    for (int i = 0; i < len; i++) {
        int bin = (data[i] * (bins - 1)) / 4095;  // Scale 12-bit to bin range
        if (bin >= 0 && bin < bins) histogram[bin]++;
    }

    float total = len;
    float sum = 0;
    for (int i = 0; i < bins; i++) sum += i * histogram[i];

    float sumB = 0, wB = 0, wF = 0;
    float max_var = 0;
    uint8_t best_thresh = bins / 2;  // Default: mid-scale

    for (int t = 0; t < bins; t++) {
        wB += histogram[t];
        if (wB == 0) continue;
        wF = total - wB;
        if (wF == 0) break;

        sumB += t * histogram[t];
        float mB = sumB / wB;
        float mF = (sum - sumB) / wF;
        float var_between = wB * wF * (mB - mF) * (mB - mF);

        if (var_between > max_var) {
            max_var = var_between;
            best_thresh = t;
        }
    }
    return best_thresh;
}
```

**Verification rules:**
- Fixed threshold = FAIL. The code must either (a) continuously recompute threshold from recent sensor data, or (b) use a min-max midpoint updated at least every 100ms.
- Min-max adaptive is the minimum acceptable approach: `threshold = (max_recent + min_recent) / 2`, updated every 100ms. This handles gradual lighting changes.
- OTSU is preferred for camera-based line tracking: it finds the optimal threshold that maximizes between-class variance. It's more robust to bimodal lighting (half-bright half-dark track).
- **Edge case**: what if the entire sensor array is over a uniform surface (all white or all black)? The spread will be near zero. In this case: **keep the previous threshold** — do NOT blindly compute a midpoint, which would be meaningless.
- Init-time calibration (sensor over white + sensor over black) is still useful as a **fallback** if the first 100ms of adaptive data is invalid. But it must NOT be the primary threshold.

**Corrective action if FAIL:**
```
⚠️ CRITICAL FIX — Replace fixed threshold with adaptive thresholding

  Your K题小车 (sensors.c:59-63) uses GPIO_ReadInputDataBit for TCRT5000
  digital output — these are already binary (comparator on the sensor module).
  If the TCRT5000 modules have on-board potentiometers for threshold adjustment,
  the adaptive thresholding must happen in the analog domain (adjust pot)
  or by using the raw ADC reading instead of the digital output.

  For camera-based line tracking:
  1. Implement OTSU or min-max adaptive threshold.
  2. Update every 100ms.
  3. If spread < threshold, keep previous value (don't update from uniform surface).
  4. Log the adaptive threshold value to OLED during debug — watch it change
     as you move the car from bright to dark areas of the track.
```

---

### Item 52: Co-Channel Interference & Competition Wireless Survival

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | 200+ teams at venue, all running Bluetooth/WiFi/2.4GHz → 2.4GHz spectrum saturated → 200ms heartbeat timeout (Item 33) triggers constantly → car emergency-stops mid-mission → task incomplete → score = 0 |
| **AI Common Mistake** | Assumes the wireless channel is clean; uses a short heartbeat timeout (200ms); allows remote commands to interrupt mission-critical autonomous phases |
| **Trigger Condition** | **Check ONLY if the system uses Bluetooth, WiFi, NRF24L01, or any 2.4GHz wireless remote control at a multi-team competition.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — 200ms heartbeat on saturated channel
#define CMD_TIMEOUT_MS  200   // Every packet loss → emergency stop!
// On competition day with 200 teams: 30-50% packet loss → car stops every 400-600ms

// ✅ CORRECT — adaptive heartbeat + mission-critical lockout
#define CMD_TIMEOUT_NORMAL_MS     500    // Normal operation: 500ms (forgiving)
#define CMD_TIMEOUT_CRITICAL_MS   2000   // Critical maneuver: 2 seconds
#define CMD_LOSS_MAX_CONSECUTIVE   10    // After 10 consecutive losses → real disconnect

static uint8_t  g_cmd_consecutive_losses = 0;
static uint32_t g_cmd_last_received = 0;
static uint8_t  g_mission_critical_phase = 0;  // Set TRUE during parking/finish

void Wireless_CommandCheck(void) {
    uint32_t elapsed = GetTick() - g_cmd_last_received;

    // Adaptive timeout: longer during critical phases
    uint32_t timeout = g_mission_critical_phase ? CMD_TIMEOUT_CRITICAL_MS : CMD_TIMEOUT_NORMAL_MS;

    if (elapsed > timeout) {
        g_cmd_consecutive_losses++;

        if (g_cmd_consecutive_losses > CMD_LOSS_MAX_CONSECUTIVE) {
            // Genuine disconnect — not just interference
            Motor_Stop_Safe();
            g_car_state = STATE_ERROR;
            LED_Alarm_Blink();
            return;
        }

        // Temporary interference → hold last valid command (slow down, don't stop)
        if (!g_mission_critical_phase) {
            // Non-critical: gradually reduce speed (not emergency stop!)
            g_smoothed_speed = g_smoothed_speed * 0.8f;  // 20% speed reduction per timeout
        }
        // Critical phase: IGNORE timeout entirely — car must finish autonomously
    } else {
        g_cmd_consecutive_losses = 0;  // Packet received → reset loss counter
    }

    // KEY: During mission-critical phases, DISREGARD remote commands entirely
    if (g_mission_critical_phase) {
        return;  // Ignore incoming commands — car relies on autonomous closed-loop only
    }

    // Process command normally (only in non-critical phases)
    if (g_cmd_updated) {
        g_cmd_updated = 0;
        g_cmd_last_received = GetTick();
        // ... process command ...
    }
}

// Mission-critical phase entry (called at parking approach, finish approach, etc.)
void Mission_CriticalPhase_Enter(void) {
    g_mission_critical_phase = 1;
    // Pre-compute the entire critical maneuver trajectory NOW (no remote override later)
    // e.g., compute parking path, set target pose, arm finish-line detector
}

void Mission_CriticalPhase_Exit(void) {
    g_mission_critical_phase = 0;
}
```

**Verification rules:**
- Heartbeat timeout for normal operation must be **≥ 500ms**, not 200ms. Multi-team venues routinely see 300-400ms packet gaps even when connected.
- **Consecutive loss counter**: a single timeout is NOT a disconnect. Require ≥ 10 consecutive timeouts before declaring disconnect. This prevents interference-induced false emergency stops.
- **Mission-critical lockout**: during parking, finish-line approach, and any irreversible maneuver, the car must ignore ALL remote commands and complete the maneuver autonomously. The remote cannot "save" the car at this point — it can only interfere.
- When interference is detected (timeouts but not yet disconnect): gradually **reduce speed** rather than emergency-stop. The car may still complete the mission at reduced speed.
- **Bluetooth AFH (Adaptive Frequency Hopping)**: enable if using Bluetooth. AFH dynamically avoids congested channels. Most BLE stacks enable this by default; HC-05 classic Bluetooth may need AT command `AT+CLASS=0` to enable AFH.
- **Frequency diversity**: if budget allows, use 433MHz (LoRa/SI4432) as a backup command channel. 433MHz is far less congested than 2.4GHz at competitions.
- **Pre-competition test**: at the venue, run a packet-loss test before the match. Send 100 packets at 50ms intervals, count ACKs. If loss > 20%, switch to a different channel or reduce reliance on remote commands.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Competition wireless hardening

  If your K题小车 does not use wireless remote control: mark as SKIP.

  If it does:
  1. Increase heartbeat timeout to 500ms normal / 2000ms critical.
  2. Add consecutive loss counter (≥10 losses = real disconnect).
  3. During parking and finish approach: IGNORE remote commands entirely.
  4. Pre-competition: run packet-loss test at venue, log results.
  5. If using HC-05/HC-06 Bluetooth: switch to BLE (nRF52840) or add 433MHz backup.
```

---

### Item 53: Flash Write PWM Stall Prevention

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | PID parameter saved to Flash while motors are running → CPU halts for 3–5ms during Flash page program → PWM timer ISR missed → motor phases lose sync → BLDC desyncs (or brushed motor twitches) → you spend 3 days chasing a "PID oscillation" that's actually a Flash write stall |
| **AI Common Mistake** | Calls Flash write on every parameter change for "data persistence," unaware that STM32 Flash writes pause the entire CPU |
| **Trigger Condition** | **Check if the code writes to internal Flash (for PID params, calibration, or logging).** |

**What to check:**

```c
// ❌ CRITICAL FAIL — Flash write during active motor control
void PID_Config_Update(float kp, float ki, float kd) {
    g_pid_config.Kp = kp;   // Update RAM
    g_pid_config.Ki = ki;
    g_pid_config.Kd = kd;
    Flash_Write(PID_FLASH_ADDR, &g_pid_config, sizeof(g_pid_config));  // CPU STALLS HERE!
    // During these 3-5ms: no ISRs, no PWM updates → motor glitch
}

// ✅ CORRECT — RAM-only update during operation, Flash write only when motors STOPPED
void PID_Config_Update_RAM(float kp, float ki, float kd) {
    g_pid_config.Kp = kp;   // RAM only — instant, no stall
    g_pid_config.Ki = ki;
    g_pid_config.Kd = kd;
    PID_SwitchGains(&g_pid, kp, ki, kd);  // Apply immediately (bumpless — Item 34)
    g_pid_config.dirty = 1;               // Mark: needs save to Flash later
}

// Flash save ONLY in safe conditions:
void PID_Config_MaybeSave(void) {
    if (!g_pid_config.dirty) return;

    // Condition: motors MUST be stopped for > 1 second
    if (Motor_IsEnabled() || (GetTick() - g_motor_stop_time) < 1000) {
        return;  // Don't even try — motors are running or just stopped
    }

    // Condition: user triggered save (long-press 5 seconds — Item 37)
    if (!Button_IsLongPress(SAVE_BUTTON, 5000)) {
        return;  // Don't auto-save — wait for explicit user command
    }

    // OK to write Flash now
    __disable_irq();  // Brief critical section for Flash unlock+write
    Flash_Unlock();
    Flash_ErasePage(PID_FLASH_ADDR);
    Flash_ProgramWord(PID_FLASH_ADDR, &g_pid_config, sizeof(g_pid_config));
    Flash_Lock();
    __enable_irq();

    g_pid_config.dirty = 0;
    LED_BlinkPattern(0x0F);  // Confirm: slow single blink = saved
}
```

**Verification rules:**
- Flash write must NEVER occur while motors are enabled. Check: `if (Motor_IsEnabled()) return;` before any Flash write operation.
- Flash write must be triggered **only by explicit user action** (long-press button ≥ 5 seconds, or specific UART save command), never automatically.
- Normal PID tuning (via UART/Bluetooth) must write to **RAM only**. The `dirty` flag pattern ensures persistence without runtime stalls.
- **Flash write duration context**: STM32F1 Flash word program = ~20µs per half-word, page erase = ~20ms. During this entire time, the CPU is stalled (Flash bus is shared with instruction fetch). No ISRs execute. No PWM updates. No I2C. Nothing.
- Alternative for frequent writes: external EEPROM (AT24C02 via I2C) or FRAM (MB85RC via I2C). These don't stall the CPU. But for competition robots with only occasional parameter saves, Flash + dirty flag is sufficient.
- If using STM32 with **dual-bank Flash** (STM32F4xx with 1MB+): write to the inactive bank while executing from the active bank → no CPU stall. This is the only safe way to write Flash during operation. But for STM32F103 (single bank), it's impossible.

**Corrective action if FAIL:**
```
⚠️ CRITICAL FIX — Guard Flash writes behind motor-disabled check

  Your K题小车 likely has no Flash write code yet (Item 25 is a recommendation).
  When implementing PID parameter storage:

  1. PID tuning via UART/Bluetooth → RAM ONLY. Set dirty flag.
  2. Flash write ONLY on long-press button (5 seconds) AND motors stopped.
  3. Before Flash write: verify Motor_IsEnabled() == 0.
  4. After Flash write: clear dirty flag, LED confirm blink.
  5. On boot: load from Flash → if CRC invalid → use defaults → mark dirty
     (so next explicit save will persist the defaults or tuned values).

  NEVER auto-save to Flash. NEVER save on every parameter change.
  One Flash write per tuning session is enough.
```

---

### Item 54: Mechanical Weight Imbalance Compensation

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Battery mounted on right side → right wheels bear 30% more weight → 50% PWM on right produces only 90% of left wheel speed → car constantly drifts left → heading PID fights the drift → motors run in perpetual "tug-of-war" → energy wasted, turning asymmetric, high-speed rollover risk |
| **AI Common Mistake** | Assumes left and right motor+wheel+load are perfectly symmetric. Generates `Motor_Left(speed)` and `Motor_Right(speed)` with identical PWM. |

**What to check:**

```c
// ❌ FAIL — symmetric PWM, ignoring physical weight distribution
void Motor_Drive(int16_t linear, int16_t angular) {
    int16_t left  = linear - angular;
    int16_t right = linear + angular;
    Motor_Left_Set(left);    // Same PWM command to both sides
    Motor_Right_Set(right);  // But physically: right side is heavier → turns slower!
}

// ✅ CORRECT — mechanical offset calibration + compensation
#define MOTOR_LEFT_OFFSET    0     // Left motor PWM offset (% points)
#define MOTOR_RIGHT_OFFSET   5     // Right motor needs +5% to match left at same speed
// These values come from PHYSICAL MEASUREMENT, not guessing!

// Calibration procedure (run ONCE during bring-up):
void Motor_Balance_Calibrate(void) {
    // Step 1: Suspend wheels off ground (car on blocks)
    OLED_ShowString(0, 0, "Wheel Cal...");
    OLED_Update();

    // Step 2: Drive left motor at fixed PWM (e.g., 30%), measure actual RPM
    Motor_Left_Set(30);
    Motor_Right_Set(0);
    Delay_ms(500);  // Let speed stabilize
    float left_rpm = Encoder_ReadRPM_Left();
    Motor_Left_Set(0);

    // Step 3: Drive right motor at SAME PWM, measure RPM
    Motor_Right_Set(30);
    Motor_Left_Set(0);
    Delay_ms(500);
    float right_rpm = Encoder_ReadRPM_Right();
    Motor_Right_Set(0);

    // Step 4: Compute offset
    // If right_rpm < left_rpm (heavier side), right needs MORE PWM to match
    if (right_rpm < left_rpm) {
        g_motor_right_offset = (int16_t)((left_rpm / right_rpm - 1.0f) * 30.0f);
        // e.g., left=100rpm, right=90rpm → offset = (100/90 - 1) × 30 = 3.3 → +3%
    }

    // Step 5: Verify — run both with compensation, measure RPM again
    Motor_Left_Set(30 + MOTOR_LEFT_OFFSET);
    Motor_Right_Set(30 + g_motor_right_offset);
    Delay_ms(500);
    left_rpm = Encoder_ReadRPM_Left();
    right_rpm = Encoder_ReadRPM_Right();
    float mismatch = fabsf(left_rpm - right_rpm) / fmaxf(left_rpm, right_rpm);
    if (mismatch > 0.05f) {
        OLED_ShowString(0, 0, "Cal FAIL");
        // Mismatch still > 5% → mechanical issue (binding, bad bearing, tire diameter difference)
    } else {
        OLED_ShowString(0, 0, "Cal OK");
    }
    Motor_Stop();

    // Step 6: Save offsets to... RAM (not Flash — Item 53). Flash save on long-press button only.
}

// Apply compensation in motor drive:
void Motor_Drive_Compensated(int16_t linear, int16_t angular) {
    int16_t left  = linear - angular + MOTOR_LEFT_OFFSET;
    int16_t right = linear + angular + g_motor_right_offset;

    // Standard duty limiting (Item 2)
    if (left  > MAX_DUTY)  left  = MAX_DUTY;
    if (left  < -MAX_DUTY) left  = -MAX_DUTY;
    if (right > MAX_DUTY)  right = MAX_DUTY;
    if (right < -MAX_DUTY) right = -MAX_DUTY;

    Motor_Left_Set(left);
    Motor_Right_Set(right);
}
```

**Verification rules:**
- Left and right motor PWM paths must have independent offset parameters. These are NOT tuning parameters — they're mechanical constants that change only if the weight distribution changes.
- Calibration must be done **under load** (car on ground, battery in position), not with wheels in the air. The weight imbalance only manifests when the wheels contact the ground.
- Calibration method with encoders (recommended): drive straight at fixed PWM, measure actual left/right speed from encoders, compute offset = `PWM × (faster_side_rpm / slower_side_rpm - 1)`.
- Calibration method without encoders (minimum viable): drive forward with `angular=0` on a flat surface for 2 meters. Measure lateral deviation from straight line. Compute offset = `deviation_cm / 200 × PWM` (approximate). Iterate 2-3 times.
- **What AI doesn't know**: the battery location, the PCB component layout, the motor manufacturing tolerance (even "identical" N20 motors can have 5-10% RPM difference at the same PWM). This is physical reality that only measurement can capture.
- The offsets should be stored alongside PID parameters (Item 25) in persistent storage, as they survive across power cycles.

**Corrective action if FAIL:**
```
⚠️ NEEDS MANUAL CALIBRATION — Mechanical weight imbalance

  Your K题小车 code (motor.c) applies symmetric PWM to left and right motors.
  This assumes perfect mechanical symmetry, which does not exist.

  Measurement procedure:
  1. Place car on flat ground with fully charged battery in normal position.
  2. Drive forward at 30% PWM (angular=0) for 2 meters.
  3. Measure lateral deviation (how far did it drift left/right?).
  4. Compute right_motor_offset = deviation_cm * 30 / 200 (approximate).
  5. If car drifts LEFT (right side heavier): right_motor_offset = +N.
  6. If car drifts RIGHT (left side heavier): right_motor_offset = -N.
  7. Iterate: adjust offset, re-run 2m test, until deviation < 5cm over 2m.

  With encoders (recommended):
  1. Drive straight at 30% PWM for 1 second.
  2. Compare left_encoder_delta vs right_encoder_delta.
  3. Offset = 30 × (delta_left/delta_right - 1) for the slower side.
  4. This is much faster and more accurate than the deviation method.
```

---

## Dimension 19: Defense-in-Depth Hardware Interface Guards

A single `constrain()` in the PID function is not enough. AI trusts one layer of protection; hardware demands two. These three items enforce that every critical hardware interface has a redundant, visible, last-line-of-defense check immediately at the register-write boundary.

---

### Item 55: Peripheral Clock-Gate Reset Recovery

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | I2C peripheral state machine hangs → SDA stuck low → software returns `HAL_ERROR` but the bus remains physically locked → all subsequent I2C transactions fail silently → sensors return stale data → car crashes with no obstacle detection |
| **AI Common Mistake** | On I2C/SPI timeout, returns an error code and lets the caller retry — never resets the peripheral hardware itself |
| **Relation** | **Extends Item 21.** Item 21 covers GPIO power-cycle (needs extra hardware — P-MOSFET on VCC). This item covers RCC clock-gate reset (pure software, works on any board). Use clock-gate first, escalate to power-cycle if clock-gate fails. |

**What to check:**

```c
// ❌ FAIL — error code only, peripheral may still be in hung state
if (I2C_WaitAck_Timeout(1000) != 0) {
    return I2C_ERROR_TIMEOUT;  // Returns error, but SDA may still be stuck LOW!
}

// ✅ CORRECT — tiered recovery: clock-gate → bus recovery → power-cycle
typedef enum { RECOV_CLOCK_GATE = 0, RECOV_BUS_RESET, RECOV_POWER_CYCLE } RecoveryLevel_t;

uint8_t I2C_Recover_Peripheral(RecoveryLevel_t level) {
    if (level == RECOV_CLOCK_GATE) {
        // Step 1: Disable I2C peripheral clock (RCC reset)
        RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, DISABLE);
        Delay_ms(10);  // Let peripheral fully discharge
        RCC_APB1PeriphResetCmd(RCC_APB1Periph_I2C1, ENABLE);  // Assert reset
        Delay_ms(1);
        RCC_APB1PeriphResetCmd(RCC_APB1Periph_I2C1, DISABLE); // Release reset
        RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE);
        Delay_ms(1);

        // Step 2: Re-initialize I2C registers completely
        I2C_Init();  // Full re-init: speed, addressing, interrupts
        // Also re-init GPIO pins (SCL/SDA) to clear any stuck states
        GPIO_Init(I2C_SCL_PORT, &i2c_scl_config);
        GPIO_Init(I2C_SDA_PORT, &i2c_sda_config);

        // Step 3: Bus recovery (9 SCL pulses to release stuck SDA)
        I2C_Bus_Recovery();  // Existing in your mpu6050.c:164-170 ✓

        return 1;
    }

    if (level == RECOV_POWER_CYCLE) {
        // Only if clock-gate failed 3 consecutive times → escalate to Item 21's power-cycle
        MPU6050_PowerCycle();  // Requires GPIO-controlled P-MOSFET on VCC
        return 1;
    }

    return 0;
}

// Unified I2C transaction with auto-recovery:
uint8_t I2C_Read_WithRecovery(uint8_t dev_addr, uint8_t reg, uint8_t *data, uint8_t len) {
    static uint8_t consecutive_failures = 0;

    for (int retry = 0; retry < 3; retry++) {
        if (I2C_Transaction(dev_addr, reg, data, len) == I2C_OK) {
            consecutive_failures = 0;  // Success → reset counter
            return 1;
        }
    }

    // 3 retries exhausted → attempt recovery
    consecutive_failures++;

    if (consecutive_failures <= 3) {
        I2C_Recover_Peripheral(RECOV_CLOCK_GATE);  // First: software reset
    } else {
        I2C_Recover_Peripheral(RECOV_POWER_CYCLE);  // Escalate: hardware power-cycle
    }

    return 0;  // Recovery attempted, caller should use last-known-good data
}
```

**Verification rules:**
- On I2C/SPI timeout, the code must NOT just return an error code. It must execute a **physical peripheral reset sequence**: disable clock → delay → enable reset → release reset → re-enable clock → re-init registers.
- The RCC (Reset and Clock Control) peripheral on every STM32 can reset any peripheral without rebooting the MCU. AI never uses this feature.
- **Tiered recovery**: (1) clock-gate reset (RCC, pure software, 15ms total) → (2) bus recovery (9 SCL pulses, 1ms) → (3) power-cycle slave VCC (GPIO + MOSFET, 150ms, Item 21). Each tier is attempted before escalating.
- The `consecutive_failures` counter must be global/static: a single I2C glitch should not trigger escalation, but persistent hangs should escalate to more aggressive recovery.
- After clock-gate reset, the peripheral's ALL registers return to power-on defaults. The complete init function must be called again — not just a partial re-configuration.

**Corrective action if FAIL:**
```c
// Add to your I2C driver (mpu6050.c):
// Replace bare "return error" with the tiered recovery pattern above.
// Your existing code has bus recovery (9 SCL pulses) at init time (lines 164-170).
// Add: runtime recovery on consecutive failures using RCC clock-gate.
```

---

### Item 56: Clock Tree Math Documentation

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Wrong HSE_VALUE → PLL outputs 108MHz instead of 72MHz → all timing wrong → UART garbage, PWM frequency off, timer periods incorrect → car behavior inexplicable → 3 days of debugging |
| **AI Common Mistake** | Writes `RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9);` with no comment about where "9" comes from or what crystal it assumes |
| **Relation** | **Extends Item 40.** Item 40 covers oscilloscope verification. This item covers code-comment verification — catching the error before flashing. |

**What to check:**

```c
// ❌ FAIL — no documentation, magic numbers
RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9);
RCC_HCLKConfig(RCC_SYSCLK_Div1);
RCC_PCLK2Config(RCC_HCLK_Div1);
RCC_PCLK1Config(RCC_HCLK_Div2);
// What frequency does this produce? No one knows without a calculator.

// ✅ CORRECT — fully documented clock tree with verification
/*
 * CLOCK TREE CONFIGURATION (STM32F103C8T6)
 * ========================================
 * HSE (External Crystal): 8.000 MHz    ← VERIFIED: metal can marking "8.000"
 *   ↓ HSE / 1 = 8 MHz into PLL
 * PLL: 8 MHz × 9 = 72 MHz             ← PLLMUL = 9
 *   ↓ SYSCLK = PLL = 72 MHz
 * AHB (HCLK):  SYSCLK / 1 = 72 MHz    ← HPRE = DIV1
 * APB2 (PCLK2): HCLK  / 1 = 72 MHz    ← PPRE2 = DIV1  (GPIO, TIM1, ADC1, USART1, SPI1)
 * APB1 (PCLK1): HCLK  / 2 = 36 MHz    ← PPRE1 = DIV2  (TIM2-7, I2C, USART2-3, SPI2)
 *
 * VERIFICATION:
 *   TIM3 clock = APB1 × 2 = 72 MHz (APB1 prescaler ≠ 1 → timer clock ×2)
 *     TIM3 PWM: 72MHz / (71+1) / (99+1) = 10.000 kHz  ✓
 *   SysTick:   72MHz / 1000 = 72,000 counts/ms → RELOAD = 71999  ✓
 *   I2C:       APB1 = 36 MHz → I2C_CCR = 180 for 100kHz standard mode  ✓
 *   USART1:    APB2 = 72 MHz → BRR = 72,000,000 / 115,200 = 625 = 0x271  ✓
 *
 * INTEGRITY CHECK: all results are integers → no rounding errors → PASS ✓
 */
RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9);     // 8 × 9 = 72 MHz
RCC_HCLKConfig(RCC_SYSCLK_Div1);                           // AHB = 72 MHz
RCC_PCLK2Config(RCC_HCLK_Div1);                            // APB2 = 72 MHz
RCC_PCLK1Config(RCC_HCLK_Div2);                            // APB1 = 36 MHz
```

**Verification rules:**
- Every clock tree register write must be accompanied by a comment block showing: (a) crystal frequency source, (b) mathematical derivation from crystal to final frequency, (c) integer check for all derived clocks.
- The comment block must include derived frequencies for: timer clocks, UART baud rate divisors, I2C CCR value, SysTick reload value. If any calculation produces a non-integer result → FAIL (the clock tree needs adjustment).
- The `HSE_VALUE` must be stated explicitly in the comment with a **verification source** ("metal can marking 8.000" or "oscilloscope measured 12.04 MHz").
- This is a CODE REVIEW check, not a hardware check (Item 40 handles the hardware side). During review: if the clock tree comment is missing or has non-integer derived values → flag as FAIL.
- AI-generated code NEVER includes this level of clock tree documentation. This item is 100% manual verification.

**Corrective action if FAIL:**
```
⚠️ MISSING DOCUMENTATION — Clock tree math verification

  Add a comment block above ALL RCC register writes showing:
  1. Crystal source and frequency
  2. PLL multiplier → SYSCLK derivation
  3. AHB/APB1/APB2 prescaler choices
  4. Derived frequencies for: timers, UART, I2C, SysTick
  5. Integer check: mark ✓ or ✗ for each derived clock

  If any derived clock is non-integer: choose different prescaler values.
  For example, 12MHz crystal with PLLMUL=6 → 72MHz → but APB1/2 divider
  choices remain the same. The point is the MATH must be shown.
```

---

### Item 57: Register-Level PWM Guard (Last Line of Defense)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | `constrain()` macro in PID function is the ONLY limit. If ANY code path bypasses the PID function (direct motor control for testing, emergency maneuvers, calibration routines), the PWM register gets written with unbounded values → duty cycle could be 100%+ → driver MOSFET overcurrent |
| **AI Common Mistake** | Places `constrain(pid_out, -MAX, MAX)` in the PID function and trusts that all PWM writes go through that one path. AI never adds a second constraint at the hardware register boundary. |
| **Relation** | **Extends Item 2.** Item 2 covers the PID-output-level constraint. This item adds a redundant, inlined, visible guard immediately before every `TIM_SetCompare` call — the last line of defense before the hardware. |

**What to check:**

```c
// ❌ FAIL — only constrains at PID level; direct Motor_Left_Set bypasses it
// In pid.c or algorithm.c:
float pid_out = PID_Compute(&pid, input, dt);
int16_t pwm = (int16_t)constrain(pid_out, -MAX_DUTY, MAX_DUTY);  // OK here
Motor_Left_Set(pwm);  // But what if Motor_Left_Set is called with raw values elsewhere?

// In motor.c — NO redundant check before register write:
void Motor_Left_Set(int16_t speed) {
    // ... direction logic ...
    TIM_SetCompare1(TIM3, pwm_val);  // Direct register write — no guard!
}

// ✅ CORRECT — hardware register guard INSIDE the setter function
void Motor_Left_Set(int16_t speed) {
    uint16_t pwm_val;

    // === HARDWARE REGISTER GUARD (last line of defense) ===
    // This check runs regardless of who called Motor_Left_Set —
    // PID, calibration, UART command, or emergency maneuver.
    if (speed > MAX_DUTY_HARD)  speed = MAX_DUTY_HARD;    // e.g., 84
    if (speed < -MAX_DUTY_HARD) speed = -MAX_DUTY_HARD;
    // =====================================================

    int16_t abs_speed = (speed > 0) ? speed : -speed;

    // Dead zone (Item 4)
    if (abs_speed > 0 && abs_speed < MOTOR_DEADZONE) {
        pwm_val = MOTOR_DEADZONE;
    } else {
        pwm_val = (uint16_t)abs_speed;
    }

    // ARR boundary guard — can't exceed timer period
    if (pwm_val > TIM3_ARR_VALUE) pwm_val = TIM3_ARR_VALUE;  // e.g., 99

    if (speed > 0) {
        GPIO_SetBits(GPIOB, GPIO_Pin_12);
        GPIO_ResetBits(GPIOB, GPIO_Pin_13);
    } else if (speed < 0) {
        GPIO_ResetBits(GPIOB, GPIO_Pin_12);
        GPIO_SetBits(GPIOB, GPIO_Pin_13);
    } else {
        pwm_val = 0;  // Exact zero: coast stop
        GPIO_ResetBits(GPIOB, GPIO_Pin_12);
        GPIO_ResetBits(GPIOB, GPIO_Pin_13);
    }

    TIM_SetCompare1(TIM3, pwm_val);  // Safe: pwm_val is guaranteed in [0, ARR]
}
```

**Verification rules:**
- Search for ALL `TIM_SetCompare` or `__HAL_TIM_SET_COMPARE` calls in the codebase. For each one, verify that an **inline `if` guard** (not just a macro called elsewhere) exists within the same function, within 5 lines preceding the register write.
- The guard must be a **hardcoded comparison** (`if (val > MAX) val = MAX;`), not solely a `constrain()` macro. The intent is visibility: a human reviewer can see the guard without tracing macro definitions.
- Two bounds must be checked: (a) the logical duty limit (85% = `MAX_DUTY_HARD`), and (b) the timer's ARR value (physical maximum of the compare register). Writing `CCR > ARR` causes undefined PWM behavior (possible 100% duty or 0%).
- The guard must be inside the **lowest-level setter function** (`Motor_Left_Set`, `Motor_Right_Set`, `Servo_SetAngle`), not at the caller level. This ensures that direct calls for testing/calibration are also protected.
- This is defense-in-depth: Item 2 constrains the PID output path; Item 57 constrains the hardware interface. Both must pass for the item to be marked ✅.

**Corrective action if FAIL:**

For your K题小车 (`motor.c:68-95`), add the hardware guard at the top of `Motor_Left_Set` and `Motor_Right_Set`:
```c
void Motor_Left_Set(int16_t speed) {
    // Hardware register guard — runs before anything else
    #define MAX_DUTY_HARD   84    // 85% of ARR=99
    if (speed > MAX_DUTY_HARD)  speed = MAX_DUTY_HARD;
    if (speed < -MAX_DUTY_HARD) speed = -MAX_DUTY_HARD;
    // ... existing code follows ...
}
```

---

## Dimension 20: Emergency Stop & Post-Mortem Diagnostics

These three items are the last line of defense when everything else fails. No AI will ever generate them — they're pure competition survival instinct, earned through the pain of watching a robot destroy itself with zero diagnostic data.

---

### Item 58: Physical Emergency-Stop Button (Human Override)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Watchdog reset glitches GPIOs → EN pin momentarily floats HIGH → motors surge. Or: PID goes unstable at 2 m/s → car heads for the crowd → no human can stop it because the wireless link is also jammed (Item 52) |
| **AI Common Mistake** | Assumes software (watchdog, state machine error, wireless stop command) is sufficient for emergency stop. Never provides a direct hardware kill switch. |

**What to check:**

```c
// ❌ CRITICAL FAIL — no physical emergency stop
// The ONLY way to stop the car is: (a) wait for IWDG reset, (b) wireless command,
// (c) state machine timeout. None of these work if MCU is in a tight ISR loop.

// ✅ CORRECT — dedicated EXTI button, highest priority, minimal ISR
// Hardware: SPST-NC (normally-closed) button between EN pin and GND,
//           with 10kΩ pull-up to 3.3V on EN pin.
//           Button pressed → EN pin pulled to GND → motor driver physically disabled.
//           This works even if MCU is completely dead.

// Software backup: EXTI on a separate GPIO, highest preemption priority
#define EMERGENCY_STOP_PIN      GPIO_Pin_12   // PB12
#define EMERGENCY_STOP_PORT     GPIOB
#define EMERGENCY_STOP_EXTI     EXTI_Line12
#define EMERGENCY_STOP_IRQn     EXTI15_10_IRQn
#define EMERGENCY_STOP_PRIORITY 0             // HIGHEST — preempts everything

void EMERGENCY_STOP_GPIO_Init(void) {
    GPIO_InitTypeDef g;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB | RCC_APB2Periph_AFIO, ENABLE);

    g.GPIO_Pin = EMERGENCY_STOP_PIN;
    g.GPIO_Mode = GPIO_Mode_IPU;  // Internal pull-up: normally HIGH, LOW = STOP
    g.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(EMERGENCY_STOP_PORT, &g);

    // EXTI: falling edge (button press = LOW)
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource12);
    EXTI_InitTypeDef e;
    e.EXTI_Line = EMERGENCY_STOP_EXTI;
    e.EXTI_Mode = EXTI_Mode_Interrupt;
    e.EXTI_Trigger = EXTI_Trigger_Falling;
    e.EXTI_LineCmd = ENABLE;
    EXTI_Init(&e);

    // Set priority AFTER NVIC_PriorityGroupConfig (done in main())
    NVIC_SetPriority(EMERGENCY_STOP_IRQn, EMERGENCY_STOP_PRIORITY);
    NVIC_EnableIRQ(EMERGENCY_STOP_IRQn);
}

// ISR: ABSOLUTE MINIMUM — clear PWM, pull EN, set flag. Nothing else.
void EXTI15_10_IRQHandler(void) {
    if (EXTI_GetITStatus(EMERGENCY_STOP_EXTI) != RESET) {
        EXTI_ClearITPendingBit(EMERGENCY_STOP_EXTI);

        // Step 1: Kill ALL PWM channels IMMEDIATELY (no ramp — this is emergency!)
        TIM_SetCompare1(TIM3, 0);
        TIM_SetCompare2(TIM3, 0);
        TIM_SetCompare3(TIM3, 0);
        TIM_SetCompare4(TIM3, 0);

        // Step 2: Pull motor enable LOW (hardware: NC button already does this too)
        GPIO_ResetBits(MOTOR_EN_PORT, MOTOR_EN_PIN);

        // Step 3: Set emergency flag for main loop awareness
        g_emergency_stop_triggered = 1;

        // Step 4: Signal via LED (raw GPIO — don't call LED functions from ISR)
        GPIO_ResetBits(GPIOC, GPIO_Pin_13);  // LED ON = emergency

        // DO NOT: call Motor_Stop(), printf(), Delay_ms(), IWDG_Refresh(), or ANY function
        // DO NOT: clear g_emergency_stop_triggered here — main loop must handle recovery
    }
}

// In main loop: handle emergency stop aftermath
if (g_emergency_stop_triggered) {
    Motor_Stop_Safe();  // Now safe to call functions
    LED_ErrorCode(0xFF);  // Solid LED = emergency stop
    g_car_state = STATE_ERROR;
    // Stay in error state until physical button released + manual reset
    while (GPIO_ReadInputDataBit(EMERGENCY_STOP_PORT, EMERGENCY_STOP_PIN) == Bit_RESET) {
        IWDG_ReloadCounter();  // Feed watchdog while waiting
    }
    // Button released → wait for explicit restart command (long-press, or power cycle)
}
```

**Verification rules:**
- A dedicated physical emergency-stop pin must exist in hardware. Minimum viable: SPST-NC button between EN and GND (works even with MCU dead).
- The EXTI ISR for emergency stop must have the **highest preemption priority** (0) — higher than sensor, motor, or communication ISRs.
- The ISR must do exactly 3 things: (1) set all PWM CCR registers to 0, (2) set EN pin LOW, (3) set a volatile flag. No function calls, no delays, no IWDG refresh.
- Recovery from emergency stop must require **explicit human action**: button release + long-press confirmation, or full power cycle. Never auto-recover.
- The emergency stop button is an **exception to Item 37** (which forbids bare EXTI for buttons). Here, false trigger = safe stop, not dangerous start, so EXTI is acceptable and preferred for minimal latency.

**Corrective action if FAIL:**
```
🔴 CRITICAL — Add physical emergency stop

  Minimum hardware (works without any code):
  1. SPST normally-closed button between TB6612 STBY pin and GND.
  2. 10kΩ pull-up from STBY to 3.3V.
  3. Button pressed → STBY = GND → motors off, regardless of MCU state.

  Software backup (required for competition):
  1. Add EXTI on a free GPIO with highest priority (0).
  2. ISR: clear PWM → pull EN → set flag → LED on. Nothing else.
  3. Recovery: requires button release + long-press or power cycle.
```

---

### Item 59: Black-Box Error Log (RAM Buffer + LED Post-Mortem)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Competition run fails → car stops in middle of field → no serial cable, no debugger, no OLED (battery dying) → zero diagnostic data → you don't know if it was I2C hang, PID NaN, encoder slip, or state machine timeout → fix is pure guesswork → next run fails the same way |
| **AI Common Mistake** | Either (a) no logging at all, or (b) `printf()` every sensor value at 100Hz (Item 26). Neither helps when the car is dead on the field with no UART connected. |

**What to check:**

```c
// ❌ CRITICAL FAIL — zero error logging
void Algorithm_RunMission(void) {
    if (state_timeout) {
        Motor_Stop();
        g_car_state = STATE_ERROR;
        // No record of WHAT caused the error or WHEN
    }
}

// ✅ CORRECT — RAM error log + LED blink playback
#define ERROR_LOG_SIZE   16

typedef struct {
    uint32_t timestamp_ms;   // When the error occurred
    uint8_t  error_code;     // What went wrong
    uint8_t  state_at_error; // Which state the car was in
    uint16_t spare_data;     // Context-specific (e.g., sensor value at error time)
} ErrorLogEntry_t;

static ErrorLogEntry_t g_error_log[ERROR_LOG_SIZE];
static volatile uint8_t g_error_log_idx = 0;

// Error codes (keep under 256 for uint8_t)
#define ERR_I2C_TIMEOUT      0x01
#define ERR_STATE_TIMEOUT    0x02
#define ERR_PID_NAN          0x03
#define ERR_VOLTAGE_SAG      0x04
#define ERR_ENCODER_SLIP     0x05
#define ERR_WIRELESS_LOST    0x06
#define ERR_IMU_STALE        0x07
#define ERR_EMERGENCY_STOP   0x08
#define ERR_WATCHDOG_RESET   0x09   // Check RCC_CSR register for IWDG reset flag
#define ERR_HARDFAULT        0x0A   // Only if we somehow recover from HardFault
#define ERR_SENSOR_CONFIDENCE 0x0B

void ErrorLog_Write(uint8_t code, uint16_t spare) {
    if (g_error_log_idx >= ERROR_LOG_SIZE) {
        // Buffer full — overwrite oldest entry (circular buffer)
        // Shift all entries left by 1, lose oldest
        for (int i = 0; i < ERROR_LOG_SIZE - 1; i++) {
            g_error_log[i] = g_error_log[i + 1];
        }
        g_error_log_idx = ERROR_LOG_SIZE - 1;
    }

    g_error_log[g_error_log_idx].timestamp_ms = GetTick();
    g_error_log[g_error_log_idx].error_code = code;
    g_error_log[g_error_log_idx].state_at_error = (uint8_t)g_car_state;
    g_error_log[g_error_log_idx].spare_data = spare;
    g_error_log_idx++;
}

// === POST-MORTEM PLAYBACK via LED ===
// Trigger: long-press button (2 seconds) while motors stopped
void ErrorLog_PlaybackLED(void) {
    if (g_error_log_idx == 0) {
        // No errors → single slow blink
        LED_BlinkPattern(0x01);  // 1 blink, long pause, repeat
        return;
    }

    // Blink error count first: N fast blinks = number of errors
    for (int i = 0; i < g_error_log_idx; i++) {
        GPIO_ResetBits(GPIOC, GPIO_Pin_13);  // LED ON
        Delay_ms(150);
        GPIO_SetBits(GPIOC, GPIO_Pin_13);    // LED OFF
        Delay_ms(150);
    }
    Delay_ms(1000);  // 1-second gap between count and data

    // Then blink each error code in binary (8 bits, MSB first)
    for (int i = 0; i < g_error_log_idx; i++) {
        uint8_t code = g_error_log[i].error_code;
        for (int bit = 7; bit >= 0; bit--) {
            if (code & (1 << bit)) {
                GPIO_ResetBits(GPIOC, GPIO_Pin_13);  // LED ON = 1
            } else {
                GPIO_SetBits(GPIOC, GPIO_Pin_13);    // LED OFF = 0
            }
            Delay_ms(300);  // 300ms per bit — human-readable
        }
        // Gap between error entries
        GPIO_SetBits(GPIOC, GPIO_Pin_13);
        Delay_ms(1500);
    }
}

// Watchdog reset detection (check at startup):
void ErrorLog_CheckWDReset(void) {
    if (RCC_GetFlagStatus(RCC_FLAG_IWDGRST) != RESET) {
        RCC_ClearFlag();  // Clear all reset flags
        // We just recovered from a watchdog reset — log it!
        ErrorLog_Write(ERR_WATCHDOG_RESET, 0);
        // Don't clear g_error_log_idx — preserve the log across reset!
        // (This requires the error log to be in a no-init RAM section)
    }
}
```

**Verification rules:**
- A RAM-based error log buffer (≥ 10 entries, circular) must exist, storing at minimum: timestamp, error code, and state at error time.
- The error log must be written to on: I2C timeout, state machine timeout, PID NaN detection (Item 13), voltage sag (Item 50), encoder slip (Item 45), wireless timeout (Item 33/52), and emergency stop (Item 58).
- Post-mortem playback must be triggerable WITHOUT a serial console: long-press button → LED blinks error codes in a human-readable pattern (binary, Morse, or blink count). The pattern must be documented in code comments or a README.
- For watchdog reset survival: the error log buffer should be placed in a **no-init RAM section** (not zeroed at startup) so the pre-reset log survives. Alternatively, write critical errors to a dedicated Flash page (with Item 53's caveat — only when motors are stopped by the error condition itself).
- The LED playback speed must be slow enough for a human to count: ≥ 200ms per bit. A smartphone video recording of the LED can be reviewed frame-by-frame later.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Black-box error log

  Add to your K题小车:
  1. ErrorLogEntry_t g_error_log[16] in RAM.
  2. ErrorLog_Write() calls at every error detection point:
     - algorithm.c: state timeout → ERR_STATE_TIMEOUT
     - algorithm.c: PID NaN → ERR_PID_NAN
     - mpu6050.c: I2C timeout → ERR_I2C_TIMEOUT
     - sensors.c: confidence invalid → ERR_SENSOR_CONFIDENCE
  3. Post-mortem trigger: hold button 2s while motors stopped.
  4. LED playback: blink error count → pause → blink codes in binary.
  5. Document the blink pattern so you can decode it at competition.
```

---

### Item 60: Dual-Config Fallback (Safe Mode Hardware Pin)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | You tuned aggressive PID gains for maximum speed. On competition day, the floor is slightly different → car oscillates → you have 2 minutes between runs → you can't re-tune 6 PID parameters in 2 minutes → you either run the aggressive tune (risk crash) or forfeit the run |
| **AI Common Mistake** | Stores one set of PID parameters. AI assumes the tuned values are correct and final. The concept of "revert to known-good defaults via hardware pin" is alien to AI. |

**What to check:**

```c
// ❌ FAIL — single parameter set, no fallback
#define SPEED_KP   2.0f    // Only one version exists
#define SPEED_KI   0.5f    // If this is wrong, you MUST reflash

// ✅ CORRECT — dual config with hardware pin selection
#define SAFE_MODE_PIN    GPIO_Pin_15   // PA15 (freed by JTAG disable — Item 42)
#define SAFE_MODE_PORT   GPIOA

typedef struct {
    float Kp, Ki, Kd;
    float output_limit, integral_limit;
    uint32_t crc32;
} PID_ConfigSet_t;

typedef struct {
    PID_ConfigSet_t conservative;  // Version A: guaranteed to finish (slow but stable)
    PID_ConfigSet_t aggressive;    // Version B: fast but risky
    uint8_t active_set;            // Which set is currently loaded? (0=A, 1=B)
    uint32_t config_crc32;         // CRC over entire struct
} PID_DualConfig_t;

static PID_DualConfig_t g_pid_dual_config;

// Must be called VERY EARLY in main(), before ANY peripheral init
uint8_t Boot_CheckSafeMode(void) {
    GPIO_InitTypeDef g;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    g.GPIO_Pin = SAFE_MODE_PIN;
    g.GPIO_Mode = GPIO_Mode_IPU;  // Pull-up: normally HIGH = aggressive mode
    g.GPIO_Speed = GPIO_Speed_2MHz;  // Slow is fine — read once at boot
    GPIO_Init(SAFE_MODE_PORT, &g);

    // Read pin level (before any other GPIO config might affect it)
    uint8_t pin_level = GPIO_ReadInputDataBit(SAFE_MODE_PORT, SAFE_MODE_PIN);

    if (pin_level == Bit_RESET) {
        // Pin is grounded → SAFE MODE → load conservative config
        return 1;  // SAFE MODE active
    }
    return 0;  // Normal mode — use aggressive config
}

void PID_Config_LoadDual(void) {
    Flash_Read(PID_FLASH_ADDR, (uint32_t*)&g_pid_dual_config, sizeof(g_pid_dual_config));

    // Verify CRC
    uint32_t calc_crc = CRC32_Calc((uint32_t*)&g_pid_dual_config,
                                    sizeof(g_pid_dual_config) - 4);
    if (calc_crc != g_pid_dual_config.config_crc32) {
        // Flash empty/corrupt → load hardcoded factory defaults
        PID_Config_LoadFactoryDefaults();
        return;
    }

    uint8_t safe_mode = Boot_CheckSafeMode();

    PID_ConfigSet_t *selected;
    if (safe_mode) {
        selected = &g_pid_dual_config.conservative;
        g_pid_dual_config.active_set = 0;
        // Signal safe mode: LED slow blink (different from normal boot)
        LED_BlinkPattern(0x03);  // 3 slow blinks = safe mode
    } else {
        selected = &g_pid_dual_config.aggressive;
        g_pid_dual_config.active_set = 1;
    }

    // Apply selected config
    PID_Init(&g_speed_pid,   selected[0].Kp, selected[0].Ki, selected[0].Kd, ...);
    PID_Init(&g_heading_pid, selected[1].Kp, selected[1].Ki, selected[1].Kd, ...);
    // (In practice: store all controllers' params in the config struct)

    // If safe mode: LOCK Flash writes (Item 53) — don't allow saving over conservative set
    if (safe_mode) {
        g_flash_write_locked = 1;  // Prevent PID_Config_MaybeSave() from writing
    }
}

// Factory defaults — hardcoded conservative values that ARE tested and WILL finish
static void PID_Config_LoadFactoryDefaults(void) {
    // Speed PID: very conservative
    g_pid_dual_config.conservative.Kp = 1.5f;   // Low gain → won't oscillate
    g_pid_dual_config.conservative.Ki = 0.3f;
    g_pid_dual_config.conservative.Kd = 0.05f;
    g_pid_dual_config.conservative.output_limit = 50.0f;   // 50% max speed
    g_pid_dual_config.conservative.integral_limit = 10.0f;

    // Heading PID: very conservative
    g_pid_dual_config.conservative.Kp = 2.0f;
    // ... etc

    // Aggressive: copied from conservative initially (user tunes later)
    g_pid_dual_config.aggressive = g_pid_dual_config.conservative;
    g_pid_dual_config.active_set = 0;

    PID_Config_SaveDual();  // Write to Flash for persistence
}
```

**Verification rules:**
- Two complete PID parameter sets must exist in persistent storage: "conservative" (factory-tested, guaranteed to finish) and "aggressive" (user-tuned for speed).
- A hardware pin with internal pull-up must be read ONCE at boot, BEFORE any other GPIO configuration. If the pin is LOW (externally grounded via jumper): force-load conservative set, lock Flash writes, signal via LED pattern.
- The factory defaults must be **physically tested** — not guessed. "Conservative" means: this exact config has completed the full course at least once without incident. Document the test date and conditions in a comment.
- The hardware safe-mode pin should be clearly labeled on the PCB silkscreen: "SAFE MODE — SHORT TO GND FOR RECOVERY."
- This is NOT the same as Item 25 (single-set Flash storage) or Item 37 (button handling). It's a **hardware-level configuration selector** that works before any software initialization. If the aggressive config causes an instant HardFault on boot, the user can ground the pin, power-cycle, and the firmware will boot into safe mode before any PID code runs.

**Corrective action if FAIL:**
```
⚠️ RECOMMENDATION — Dual-config safe mode

  For your K题小车:
  1. Reserve one Flash page for PID_DualConfig_t (2 complete PID sets + CRC).
  2. Choose a free GPIO with internal pull-up (e.g., PA15, freed if JTAG disabled).
  3. At boot, check pin level BEFORE any PID init.
  4. If LOW: load conservative set, lock Flash writes, blink LED 3× slow.
  5. Factory defaults: set speed PID output limit to 50%, heading PID Kp low.
     Test this conservative config on the full course at least once.
  6. Label the safe-mode pin on your PCB/enclosure.

  Competition recovery procedure:
  1. Aggressive tune fails → car crashes.
  2. Pull battery, short SAFE_MODE pin to GND with jumper.
  3. Reconnect battery → car boots in safe mode (LED 3 slow blinks).
  4. Run the course at reduced speed → at least you finish and get a score.
```

---

## AI Prompt Prefix Template (Updated — 60 items)

```
请严格遵循以下嵌入式安全约束（违反任何一条都可能导致硬件烧毁、人员受伤或板子变砖）：

【物理层】
1. PWM固定15kHz，占空比限幅[-85%,+85%]，速度闭环用rpm/cm·s⁻¹非PWM%
2. 急停50ms斜坡减速，禁止瞬间拉0%；优先滑行停止
3. 传感器输入引脚IN_FLOATING/IPU/AF_OD，绝不用Out_PP
4. EN默认低电平；上电：PWM=0%→延时100ms→EN拉高
5. 电池电压ADC实时监测+前馈补偿：PWM×(额定/当前)，上限1.5×
6. 动态负载检测：急加速电压跌>0.8V→限制加速度；累计mAh做独立电量
7. 每个TIM_SetCompare前5行内必须有内联if(pwm>MAX→pwm=MAX)，不依赖宏
8. 外接急停按键(常闭)：硬件直连EN到GND+软件EXTI最高优先级(0)，ISR内只清PWM+拉EN+置标志

【算法层】
9. PID积分分离+I项限幅+输出限幅；参数切换bumpless transfer
10. asin/sqrt/除前值域检查；PID输出isnan()守护
11. 姿态更新硬件定时器中断(≥200Hz)，禁止delay()
12. 编码器定时器模式TI12+5-10窗口滑动平均
13. 转向-速度耦合：speed×(1-|steer|×coeff)；阿克曼后轮禁止差速
14. 速度目标值一阶低通滤波，禁止0→100跳变
15. 陀螺零偏动态补偿(静止+滑动窗口)；加速度计运动时动态降权
16. 编码器里程计+IMU联合抗打滑
17. 摄像头与IMU时间戳对齐
18. 视觉/灰度阈值动态更新(OTSU/min-max每100ms)

【逻辑层】
19. 使能IWDG(4秒)，仅主循环末尾喂狗
20. 中断优先级：急停>姿态/电机(0-1)>I2C/ADC(2-5)>串口(14-15)
21. 电池低于阈值关EN，不自动恢复
22. 状态机每个状态带超时→STATE_ERROR
23. 按键定时器10ms轮询+3次一致，区分短按/长按
24. 起跑三重确认(按键+静止500ms+线检测)；冲线(里程>95%+线50ms+死锁)

【总线层】
25. I2C/SPI≤10ms超时；超时后RCC时钟关断+外设复位+重初始化
26. 串口DMA环形缓冲区+IDLE中断
27. 无线心跳500ms；≥10次丢包=断开；关键任务期间无视遥控
28. DMA-CPU双缓冲/ISR内拷贝

【代码层】
29. ISR中修改的全局变量volatile
30. 整数参与浮点运算前显式(float)；浮点转整前值域限幅
31. printf每≥100ms，比赛固件#ifdef DEBUG_ENABLED
32. PID平时只存RAM；仅长按5秒+电机停转才写Flash
33. ISR内禁止局部数组；大数组(>64B)static/全局
34. Stack_Size≥0x00002000(8KB)
35. 时钟树所有寄存器配置注释HSE_VALUE来源+数学验算+整数校验

【永不触碰】
36. PA13(SWDIO)/PA14(SWCLK)调试阶段绝不复用
37. 外设初始化延时不可删除；初始后WHO_AM_I验证，失败重试3次

【赛后诊断】
38. RAM错误日志(≥16条环形缓冲)：记录时间戳+错误码+当时状态；长按2秒LED闪烁回放
39. Flash双配置(保守+激进)：启动时检测硬件引脚电平，低电平→强制加载保守版+锁定Flash写

【机械校准】
40. 左右轮机械零偏必须实测补偿

【条件项】
41. 若含舵机：角度10°-170°，极限>2s强制回中
42. 若为平衡车：Pitch/Roll>30°切电机EN
43. 若含麦轮：先跑单轮测试人眼确认方向
44. 若含循迹/视觉：置信度判断+自适应阈值
45. 若含阿克曼：后轮左右PWM完全相等
46. 场地摩擦在线辨识+PID增益自适应
```

---

## Final Output Report Template (All 20 Dimensions)

```
# ============================================================
#  EMBEDDED HARDWARE SAFETY REVIEW REPORT
# ============================================================
# Project: [name]    Target: [MCU]    Date: [today]
# ============================================================

## OVERALL SEVERITY: [CRITICAL / HIGH / MEDIUM / LOW]

## D1: Hardware Physical Safety (1-5)
## D2: Sensors & Algorithms (6-9)
## D3: Logic & Exception Handling (10-13)
## D4: [RESERVED] (14-17)
## D5: Power & Electrical Integrity (18-20)
## D6: Communication Bus Deadlock (21-22)
## D7: Control Strategy & Physical Limits (23-24)
## D8: Data & Debugging (25-26)
## D9: Code Structural Safety (27-28)
## D10: Kinematics & Motion Smoothing (29-30)
## D11: Sensor Calibration & Drift (31, 35)
## D12: Real-Time CPU & DMA Integrity (32, 39)
## D13: Wireless Control & Mode Transition (33-34)
## D14: State Machine & Input Robustness (36-37)
## D15: System Startup & Debug Integrity (38, 40-42)
## D16: Battery-Aware Control & Motion Decoupling (43-46)
## D17: Multi-Sensor Fusion & Competition Logic (47-49)
## D18: Competition Field Survival (50-54)
## D19: Defense-in-Depth Hardware Guards (55-57)
## D20: Emergency Stop & Post-Mortem Diagnostics (58-60)

## CRITICAL FIXES REQUIRED (before power-on)
## NEEDS MANUAL VERIFICATION
## CORRECTIVE CODE SNIPPETS
# ============================================================
```

---

## Quick Reference: AI's Top 60 Embedded Safety Sins

**D1 — Physical (1-5):**
1. PWM=15kHz? 2. PID≤85%? 3. IN pins IN/IPU/AF_OD? 4. Dead zone measured? 5. ADC+MotorDisable()?

**D2 — Sensors (6-9):**
6. ResetYaw all statics+FIFO? 7. Integral separation+clamp? 8. No delay(), timer ISR? 9. TI12+MA filter?

**D3 — Logic (10-13):**
10. IWDG main loop only? 11. PWM=0%→100ms→EN=HIGH? 12. ISR prio: sensor 0, UART 14? 13. safe_asin/safe_sqrt+isnan()?

**D4 — [RESERVED 14-17]:** ⏳

**D5 — Power (18-20):**
18. Soft ramp 50ms? 19. 100nF+10µF MPU6050? 20. [SERVO] 10°-170°, >2s→center?

**D6 — Bus (21-22):**
21. I2C≤10ms, >5 fail→RCC clock-gate+power-cycle? 22. UART DMA+IDLE?

**D7 — Control (23-24):**
23. [BALANCE] Pitch/Roll>30°→disable? 24. Speed×(1-|steer|×coeff)?

**D8 — Data (25-26):**
25. PID Flash+CRC? 26. printf≤10Hz, #ifdef DEBUG?

**D9 — Code (27-28):**
27. ISR globals volatile? 28. (float) cast, clamp before (int16_t)?

**D10 — Kinematics (29-30):**
29. [MECANUM] Single-wheel test? 30. EMA/ramp on speed setpoint?

**D11 — Calibration (31, 35):**
31. Gyro bias dynamic? 35. [VISION] Confidence+fallback?

**D12 — CPU & DMA (32, 39):**
32. ISR GPIO toggle<50%? 39. DMA double-buffer?

**D13 — Wireless (33-34):**
33. [WIRELESS] 500ms+lockout? 34. PID bumpless transfer?

**D14 — State Machine (36-37):**
36. State timeout→STATE_ERROR? 37. Button poll 10ms+3 reads?

**D15 — System (38, 40-42):**
38. Stack≥0x2000? 40. HSE=xtal+GPIO verified? 41. Delays+WHO_AM_I? 42. PA13/14 untouched?

**D16 — Battery & Motion (43-46):**
43. cm/s, V feedforward? 44. Accel weight dynamic? 45. [ENCODER] Slip? 46. Chassis type?

**D17 — Fusion & Comp (47-49):**
47. [CAM+IMU] Timestamps? 48. [COMP] Start triple+finish triple? 49. Friction adaptive?

**D18 — Field Survival (50-54):**
50. V sag+coulomb? 51. [VISION] OTSU adaptive? 52. [WIRELESS] Co-channel? 53. Flash only stopped? 54. L/R calibrated?

**D19 — Defense-in-Depth (55-57):**
55. I2C→RCC clock off→re-init? 56. Clock tree math in comments? 57. Inline if(pwm>MAX) before TIM_SetCompare?

**D20 — Emergency & Post-Mortem (58-60):**
58. NC button EN→GND + EXTI highest prio? ISR: PWM=0 + EN=LOW only?
59. RAM error log (≥16 entries)? Long-press→LED blink playback? Error codes documented?
60. Flash dual config (conservative+aggressive)? Safe-mode pin at boot? Factory defaults tested?
