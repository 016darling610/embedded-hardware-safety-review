## Dimension 14: State Machine & Input Robustness

AI writes switch-case state machines that assume every transition condition will eventually be met. In the physical world, wheels slip, sensors miss, and buttons bounce.

---

### Item 36: State Machine Timeout Protection

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Wheels slip on a turn → encoder never reaches "90°" target → state machine stuck in TURNING state forever → car spins in place until battery dies. (Your K题小车 has EXACTLY this bug!) |
| **AI Common Mistake** | `case TURNING: if (encoder_angle >= 90) { state = MOVING; break; }` — with no timeout, no backup exit |

**What to check:**

```c
// ❌ CRITICAL FAIL — state with no timeout
// algorithm.c:143-161 (Algorithm_AvoidBlackCircles):
case 1: /* 右转绕过 */
    Motor_Drive(AVOID_SPEED, AVOID_TURN);
    if (now - phase_timer > 400) { avoid_phase = 2; phase_timer = now; }
    break;
// This has a phase timeout (400ms) — GOOD! But other states don't.
// algorithm.c:298-307 (STATE_NAVIGATE):
// Relies ONLY on ultrasonic distance OR state_elapsed > 10000 to exit.
// What if ultrasonic never reads < 20cm? Car drives straight into wall.

// ✅ CORRECT — every state has BOTH a success condition AND a timeout
typedef struct {
    uint32_t entry_time;       // Tick when state was entered
    uint32_t timeout_ms;       // Max allowed duration in this state
    uint8_t  timeout_action;   // What to do on timeout
} StateTimer_t;

#define STATE_TIMEOUT_MS    5000   // Default: 5 seconds per state

CarState_t Algorithm_RunMission(uint8_t mission_type) {
    static StateTimer_t state_timer = {0};
    uint32_t now = GetTick();

    // State transition → reset timer
    if (g_car_state != prev_state) {
        state_timer.entry_time = now;
        prev_state = g_car_state;
    }
    uint32_t elapsed = now - state_timer.entry_time;

    switch (g_car_state) {
        case STATE_NAVIGATE:
            Algorithm_Navigate(90.0f, 50.0f);

            // Success condition
            if (g_sensors.us_distance_cm[1] < 20.0f) {
                Motor_Stop();
                g_car_state = STATE_DONE;
            }
            // TIMEOUT protection ← THIS IS NEW!
            else if (elapsed > 15000) {   // 15 seconds max for navigation
                Motor_Stop_Safe();
                g_car_state = STATE_ERROR;
                LED_ErrorCode(1);  // Blink error code 1: navigation timeout
            }
            break;

        case STATE_AVOID_BLACK:
            // ... existing logic ...
            // ADD: timeout for the entire STATE, not just phases
            if (elapsed > 20000) {
                Motor_Stop_Safe();
                g_car_state = STATE_ERROR;
                LED_ErrorCode(2);  // Error code 2: avoid-black timeout
            }
            break;

        // ... same pattern for ALL states ...
    }
}
```

**Verification rules:**
- **Every** state in the state machine must have a timeout. No state should depend solely on a sensor condition for exit.
- Timeout values: navigation → 15s, obstacle avoidance → 20s, circling → 20s, exploration → 35s, racing → 15s. These are generous but finite.
- On state timeout: transition to `STATE_ERROR` (not `STATE_DONE` — don't silently succeed). `STATE_ERROR` = motors stopped, LED error code blinking, wait for manual intervention.
- The timeout must be per-state-entry (reset timer on each state transition), not a global timer from mission start.
- **Nested timeouts**: if a state has internal phases (like your `Algorithm_AvoidBlackCircles` phases 0-4), the outer state timeout is a separate backstop. If a phase loops back to itself indefinitely, the outer state timeout catches it.

**Corrective action if FAIL:**

For your K题小车, add to `Algorithm_RunMission()`:
```c
// After computing state_elapsed, BEFORE each switch case:
#define STATE_TIMEOUT_NAVIGATE    15000
#define STATE_TIMEOUT_AVOID       20000
#define STATE_TIMEOUT_CIRCLE      25000
#define STATE_TIMEOUT_RACING      15000
#define STATE_TIMEOUT_EXPLORE     35000

// In each case, add after the success condition:
else if (state_elapsed > STATE_TIMEOUT_XXX) {
    Motor_Stop_Safe();
    g_car_state = STATE_ERROR;
}
```

---

### Item 37: Button Debounce by Timer Polling (Not EXTI)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Mechanical button bounce → ISR fires 3–5 times on a single press → startup counter triggers 3 times → car does "start-stop-start" in 20ms → jerks off the starting line or (for balance bot) tips itself over |
| **AI Common Mistake** | `HAL_GPIO_EXTI_Callback` detects falling edge and immediately starts the mission, with no debounce |

**What to check:**

```c
// ❌ CRITICAL FAIL — bare EXTI, no debounce
void EXTI0_IRQHandler(void) {
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_Pin_0)) {
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_Pin_0);
        StartMission();  // Called 3–7 times per button press due to bounce!
    }
}

// ✅ CORRECT — timer interrupt polls button at 10ms, confirms with 3 consistent reads
#define BUTTON_DEBOUNCE_MS     10     // Poll every 10ms
#define BUTTON_CONSISTENT_CNT   3     // 3 consistent readings = confirmed

// NO EXTI for buttons — use a timer-driven poll instead:
static uint8_t button_history[4] = {0xFF, 0xFF, 0xFF, 0xFF};  // 4 buttons, 8 samples each

void Button_Poll_10ms(void) {   // Called from a 100Hz timer ISR or SysTick hook
    uint16_t raw = GPIO_ReadInputData(GPIOB) & 0x000F;  // PB0-PB3 = 4 buttons

    for (int i = 0; i < 4; i++) {
        // Shift history left, insert new sample at LSB
        button_history[i] = (button_history[i] << 1) | ((raw >> i) & 0x01);

        // Check for stable LOW (pressed): last 3 samples all 0
        if ((button_history[i] & 0x07) == 0x00) {
            Button_OnPress(i);  // Confirmed press
            button_history[i] = 0xFF;  // Reset to prevent repeat triggers
        }
        // Check for stable HIGH (released): last 3 samples all 1
        if ((button_history[i] & 0x07) == 0x07) {
            Button_OnRelease(i);  // Confirmed release
        }
    }
}

// Long-press detection (5 seconds):
static uint32_t button_press_times[4] = {0};
static uint8_t  button_long_press_reported[4] = {0};

void Button_OnPress(uint8_t btn_id) {
    button_press_times[btn_id] = GetTick();
    button_long_press_reported[btn_id] = 0;

    if (btn_id == 0) {  // Short press → start/stop
        if (g_car_state == STATE_IDLE) StartMission();
        else Motor_Stop_Safe();
    }
}

void Button_Poll_LongPress(void) {  // Call every 100ms in main loop
    for (int i = 0; i < 4; i++) {
        if (button_press_times[i] > 0 && !button_long_press_reported[i]) {
            if (GetTick() - button_press_times[i] > 5000) {  // 5 seconds
                button_long_press_reported[i] = 1;
                if (i == 0) {  // Button 0 long-press → factory reset
                    PID_Config_LoadDefaults();  // Restore default PID params
                    PID_Config_Save();          // Write to Flash
                    LED_BlinkPattern(0xAA);     // Confirm: fast double-blink
                }
            }
        }
    }
}
```

**Verification rules:**
- Mechanical buttons must NOT use EXTI (external interrupt) for mission-critical functions. EXTI is acceptable only for emergency stop (where false triggers cause safe stops, not dangerous starts).
- Button reading must be done by **timer-driven polling** at 10–20 ms intervals, with **≥3 consecutive consistent readings** required to confirm a state change.
- **Short press** (< 500ms) and **long press** (> 5 seconds) must be distinguished. Long press is the universal escape hatch for parameter recovery.
- After a confirmed press is detected, the history byte must be reset (set to 0xFF) to prevent auto-repeat.
- For DIP switches or coding switches (like your K題小车's ReadMissionSelector): debounce is still needed! Read DIP switches at startup only (before motors enable), read 3 times with 5ms intervals, and take the majority vote. DIP switches can also bounce during handling.

**Corrective action if FAIL:**
```c
// Replace EXTI-based button with timer-polled button:
// 1. Remove HAL_GPIO_EXTI_Callback for button pins
// 2. Add Button_Poll_10ms() call to SysTick_Handler (already at 1ms)
//    or create a dedicated 100Hz timer.
// 3. In SysTick_Handler:
void SysTick_Handler(void) {
    g_sys_tick_ms++;
    static uint8_t tick_div = 0;
    tick_div++;
    if (tick_div >= 10) {  // Every 10ms
        tick_div = 0;
        Button_Poll_10ms();
    }
}
```

---

