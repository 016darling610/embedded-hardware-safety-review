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

