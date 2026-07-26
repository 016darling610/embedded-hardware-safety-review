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

