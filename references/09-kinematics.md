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

