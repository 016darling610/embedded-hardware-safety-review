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

