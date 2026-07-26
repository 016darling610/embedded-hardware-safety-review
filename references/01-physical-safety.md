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

