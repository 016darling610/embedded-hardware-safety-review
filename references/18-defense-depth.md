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

