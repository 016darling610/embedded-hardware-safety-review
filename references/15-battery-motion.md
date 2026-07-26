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

