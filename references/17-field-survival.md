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

