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

