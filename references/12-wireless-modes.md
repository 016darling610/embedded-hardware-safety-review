## Dimension 13: Wireless Control & Mode Transition

AI treats wireless commands as always-fresh and mode switches as instantaneous. In the real world, RF drops out and controllers have inertia.

---

### Item 33: Wireless Command Heartbeat Timeout

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Bluetooth/WiFi packet loss → last received command (`'F'` = forward) stays in buffer → car keeps going forward into a wall; or auto/manual mode switch → PID integrator still holds pre-switch error → violent jerk |
| **AI Common Mistake** | `if (Recv_Cmd == 'F') go_forward();` — the command variable persists forever. AI never adds a timeout. |
| **Trigger Condition** | **Check ONLY if the system uses Bluetooth, WiFi, NRF24L01, or any wireless remote control. Skip if the vehicle is fully autonomous with no remote command input.** |

**What to check:**

```c
// ❌ CRITICAL FAIL — stale command, no heartbeat
volatile char g_uart_cmd = 0;  // Set by UART ISR, read by main loop

void MainLoop(void) {
    if (g_uart_cmd == 'F') {
        Motor_Drive(70, 0);  // Forward at 70% speed
    } else if (g_uart_cmd == 'S') {
        Motor_Stop();
    }
    // g_uart_cmd stays 'F' forever if Bluetooth disconnects → car never stops!
}

// ✅ CORRECT — heartbeat watchdog + command timeout
#define CMD_TIMEOUT_MS      200     // 200ms without new command → emergency stop
#define HEARTBEAT_INTERVAL_MS 50    // Host sends heartbeat every 50ms

static volatile uint32_t g_last_cmd_tick = 0;
static volatile uint8_t  g_uart_cmd = 0;
static volatile uint8_t  g_cmd_updated = 0;

void UART_ProcessInMainLoop(void) {
    if (g_cmd_updated) {
        g_cmd_updated = 0;
        g_last_cmd_tick = GetTick();  // Refresh heartbeat timer
        // Process command...
    }

    // Heartbeat timeout check — runs EVERY main loop iteration
    if ((GetTick() - g_last_cmd_tick) > CMD_TIMEOUT_MS) {
        Motor_Stop_Safe();                    // Emergency stop
        g_car_state = STATE_ERROR;            // Enter safe state
        LED_Alarm_Blink();                    // Alert user: signal lost
    }
}

// Mode switch with PID reset:
void SwitchToAutoMode(void) {
    Motor_Stop_Safe();
    PID_Reset(&g_speed_pid);       // Clear all integrators!
    PID_Reset(&g_heading_pid);
    g_smoothed_speed = 0.0f;      // Reset speed ramp
    g_car_state = STATE_NAVIGATE;
}

void SwitchToManualMode(void) {
    Motor_Stop_Safe();
    PID_Reset(&g_speed_pid);
    PID_Reset(&g_heading_pid);
    g_smoothed_speed = 0.0f;
    g_car_state = STATE_IDLE;      // Wait for manual commands
}
```

**Verification rules:**
- A command timeout (typically **200ms**) must be enforced. If no new command arrives within the timeout window, motors must stop.
- The host/remote should send periodic heartbeat packets (every 50–100ms) even when no control change occurs. This lets the vehicle distinguish "connected but idle" from "disconnected."
- On mode switch (manual↔auto, or between different auto sub-modes), ALL PID integrators must be zeroed, and the speed ramp must be reset to 0.
- The command buffer variable must be tagged with an update flag (`g_cmd_updated`) so the main loop can distinguish "same command sent again" from "no command received."
- **Never** call `HAL_Delay()` or any blocking function inside the command processing path. Command parsing is time-sensitive.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Wireless command heartbeat

  If your K题小车 currently operates fully autonomously (no remote control):
  mark this item as SKIP.

  If you add Bluetooth/WiFi remote control later:
  1. Implement a 200ms command timeout watchdog.
  2. Send heartbeat packets from host every 50ms (can be a single byte: 0xFF).
  3. On timeout: Motor_Stop_Safe() + STATE_ERROR.
  4. On any mode switch: PID_Reset() all controllers + reset speed ramp.
```

---

### Item 34: PID Parameter Bumpless Transfer

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Switching from "straight-line PID" (Kp=5.0) to "sharp-turn PID" (Kp=1.0) → current output was 70% at old Kp → new PID initializes with integral=0 → output jumps to ~10% → car suddenly decelerates → or worse, output sign flips → motor reverses mid-turn → gear stripped |
| **AI Common Mistake** | Loads new Kp/Ki/Kd values without re-initializing the integrator to maintain output continuity |

**What to check:**

```c
// ❌ CRITICAL FAIL — abrupt PID parameter switch
void SwitchToTurnPID(void) {
    g_pid.Kp = 1.0f;   // Was 5.0 → output instantly drops to 1/5!
    g_pid.Ki = 0.1f;
    g_pid.Kd = 0.3f;
    // integral still holds value from straight-line mode → huge overcorrection
}

// ✅ CORRECT — bumpless transfer: match old output with new parameters
void PID_SwitchParams_Bumpless(PID_t *pid, float new_Kp, float new_Ki, float new_Kd) {
    // Step 1: Save current output
    float current_output = pid->P_Term + pid->Ki * pid->integral + pid->D_Term;

    // Step 2: Apply new gains
    pid->Kp = new_Kp;
    pid->Ki = new_Ki;
    pid->Kd = new_Kd;

    // Step 3: Recalculate integral so P+I+D using new gains = same output
    // At steady state, D≈0, so: Kp_new * error + Ki_new * integral_new ≈ current_output
    // → integral_new = (current_output - Kp_new * pid->prev_error) / Ki_new
    if (fabsf(new_Ki) > 0.0001f) {
        pid->integral = (current_output - new_Kp * pid->prev_error) / new_Ki;
    } else {
        pid->integral = 0;  // Ki=0 means P-only controller
    }

    // Step 4: Clamp to new integral limit
    if (pid->integral > pid->integral_limit)  pid->integral = pid->integral_limit;
    if (pid->integral < -pid->integral_limit) pid->integral = -pid->integral_limit;

    // Result: output transitions smoothly — no jerk, no surge, no gear shock
}
```

**Verification rules:**
- Every location where PID gains are modified at runtime must use a bumpless transfer function, not direct assignment.
- The bumpless transfer formula: `I_new = (Output_old − Kp_new × Error_current − Kd_new × Derivative_current) / Ki_new`, then clamp to limits.
- This is especially critical for: multi-mode competition robots (straight/turn/park), balancing robots switching between balance and rest modes, and any controller with gain scheduling.
- **Edge case**: switching to Ki=0 (P-only or PD-only) → integrator must be zeroed since division by zero is undefined. The output will have a step change proportional to the old I-term contribution — this is unavoidable but safe if documented.
- Verify that after parameter switch, the actual PWM value changes by < 5% per control cycle. A larger change indicates incorrect bumpless math.

**Corrective action if FAIL:**
```c
// Add to pid.c:
void PID_SwitchGains(PID_t *pid, float kp, float ki, float kd) {
    // Capture current effective output before gain change
    float P_now = pid->Kp * pid->prev_error;
    float I_now = pid->Ki * pid->integral;
    float D_now = (pid->prev_error != pid->prev_prev_error)
                  ? pid->Kd * (pid->prev_error - pid->prev_prev_error) / pid->dt
                  : 0.0f;
    float output_before = P_now + I_now + D_now;

    // Apply new gains
    pid->Kp = kp;
    pid->Ki = ki;
    pid->Kd = kd;

    // Recompute integral for output continuity
    pid->integral = ((ki > 0.0001f) ? (output_before - kp * pid->prev_error) / ki : 0.0f);

    // Safety clamp
    float limit = pid->integral_limit;
    if (pid->integral > limit) pid->integral = limit;
    else if (pid->integral < -limit) pid->integral = -limit;
}
```

---

