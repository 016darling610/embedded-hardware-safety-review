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

