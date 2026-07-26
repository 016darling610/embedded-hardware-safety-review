## Dimension 20: Emergency Stop & Post-Mortem Diagnostics

These three items are the last line of defense when everything else fails. No AI will ever generate them — they're pure competition survival instinct, earned through the pain of watching a robot destroy itself with zero diagnostic data.

---

### Item 58: Physical Emergency-Stop Button (Human Override)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Watchdog reset glitches GPIOs → EN pin momentarily floats HIGH → motors surge. Or: PID goes unstable at 2 m/s → car heads for the crowd → no human can stop it because the wireless link is also jammed (Item 52) |
| **AI Common Mistake** | Assumes software (watchdog, state machine error, wireless stop command) is sufficient for emergency stop. Never provides a direct hardware kill switch. |

**What to check:**

```c
// ❌ CRITICAL FAIL — no physical emergency stop
// The ONLY way to stop the car is: (a) wait for IWDG reset, (b) wireless command,
// (c) state machine timeout. None of these work if MCU is in a tight ISR loop.

// ✅ CORRECT — dedicated EXTI button, highest priority, minimal ISR
// Hardware: SPST-NC (normally-closed) button between EN pin and GND,
//           with 10kΩ pull-up to 3.3V on EN pin.
//           Button pressed → EN pin pulled to GND → motor driver physically disabled.
//           This works even if MCU is completely dead.

// Software backup: EXTI on a separate GPIO, highest preemption priority
#define EMERGENCY_STOP_PIN      GPIO_Pin_12   // PB12
#define EMERGENCY_STOP_PORT     GPIOB
#define EMERGENCY_STOP_EXTI     EXTI_Line12
#define EMERGENCY_STOP_IRQn     EXTI15_10_IRQn
#define EMERGENCY_STOP_PRIORITY 0             // HIGHEST — preempts everything

void EMERGENCY_STOP_GPIO_Init(void) {
    GPIO_InitTypeDef g;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB | RCC_APB2Periph_AFIO, ENABLE);

    g.GPIO_Pin = EMERGENCY_STOP_PIN;
    g.GPIO_Mode = GPIO_Mode_IPU;  // Internal pull-up: normally HIGH, LOW = STOP
    g.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(EMERGENCY_STOP_PORT, &g);

    // EXTI: falling edge (button press = LOW)
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource12);
    EXTI_InitTypeDef e;
    e.EXTI_Line = EMERGENCY_STOP_EXTI;
    e.EXTI_Mode = EXTI_Mode_Interrupt;
    e.EXTI_Trigger = EXTI_Trigger_Falling;
    e.EXTI_LineCmd = ENABLE;
    EXTI_Init(&e);

    // Set priority AFTER NVIC_PriorityGroupConfig (done in main())
    NVIC_SetPriority(EMERGENCY_STOP_IRQn, EMERGENCY_STOP_PRIORITY);
    NVIC_EnableIRQ(EMERGENCY_STOP_IRQn);
}

// ISR: ABSOLUTE MINIMUM — clear PWM, pull EN, set flag. Nothing else.
void EXTI15_10_IRQHandler(void) {
    if (EXTI_GetITStatus(EMERGENCY_STOP_EXTI) != RESET) {
        EXTI_ClearITPendingBit(EMERGENCY_STOP_EXTI);

        // Step 1: Kill ALL PWM channels IMMEDIATELY (no ramp — this is emergency!)
        TIM_SetCompare1(TIM3, 0);
        TIM_SetCompare2(TIM3, 0);
        TIM_SetCompare3(TIM3, 0);
        TIM_SetCompare4(TIM3, 0);

        // Step 2: Pull motor enable LOW (hardware: NC button already does this too)
        GPIO_ResetBits(MOTOR_EN_PORT, MOTOR_EN_PIN);

        // Step 3: Set emergency flag for main loop awareness
        g_emergency_stop_triggered = 1;

        // Step 4: Signal via LED (raw GPIO — don't call LED functions from ISR)
        GPIO_ResetBits(GPIOC, GPIO_Pin_13);  // LED ON = emergency

        // DO NOT: call Motor_Stop(), printf(), Delay_ms(), IWDG_Refresh(), or ANY function
        // DO NOT: clear g_emergency_stop_triggered here — main loop must handle recovery
    }
}

// In main loop: handle emergency stop aftermath
if (g_emergency_stop_triggered) {
    Motor_Stop_Safe();  // Now safe to call functions
    LED_ErrorCode(0xFF);  // Solid LED = emergency stop
    g_car_state = STATE_ERROR;
    // Stay in error state until physical button released + manual reset
    while (GPIO_ReadInputDataBit(EMERGENCY_STOP_PORT, EMERGENCY_STOP_PIN) == Bit_RESET) {
        IWDG_ReloadCounter();  // Feed watchdog while waiting
    }
    // Button released → wait for explicit restart command (long-press, or power cycle)
}
```

**Verification rules:**
- A dedicated physical emergency-stop pin must exist in hardware. Minimum viable: SPST-NC button between EN and GND (works even with MCU dead).
- The EXTI ISR for emergency stop must have the **highest preemption priority** (0) — higher than sensor, motor, or communication ISRs.
- The ISR must do exactly 3 things: (1) set all PWM CCR registers to 0, (2) set EN pin LOW, (3) set a volatile flag. No function calls, no delays, no IWDG refresh.
- Recovery from emergency stop must require **explicit human action**: button release + long-press confirmation, or full power cycle. Never auto-recover.
- The emergency stop button is an **exception to Item 37** (which forbids bare EXTI for buttons). Here, false trigger = safe stop, not dangerous start, so EXTI is acceptable and preferred for minimal latency.

**Corrective action if FAIL:**
```
🔴 CRITICAL — Add physical emergency stop

  Minimum hardware (works without any code):
  1. SPST normally-closed button between TB6612 STBY pin and GND.
  2. 10kΩ pull-up from STBY to 3.3V.
  3. Button pressed → STBY = GND → motors off, regardless of MCU state.

  Software backup (required for competition):
  1. Add EXTI on a free GPIO with highest priority (0).
  2. ISR: clear PWM → pull EN → set flag → LED on. Nothing else.
  3. Recovery: requires button release + long-press or power cycle.
```

---

### Item 59: Black-Box Error Log (RAM Buffer + LED Post-Mortem)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Competition run fails → car stops in middle of field → no serial cable, no debugger, no OLED (battery dying) → zero diagnostic data → you don't know if it was I2C hang, PID NaN, encoder slip, or state machine timeout → fix is pure guesswork → next run fails the same way |
| **AI Common Mistake** | Either (a) no logging at all, or (b) `printf()` every sensor value at 100Hz (Item 26). Neither helps when the car is dead on the field with no UART connected. |

**What to check:**

```c
// ❌ CRITICAL FAIL — zero error logging
void Algorithm_RunMission(void) {
    if (state_timeout) {
        Motor_Stop();
        g_car_state = STATE_ERROR;
        // No record of WHAT caused the error or WHEN
    }
}

// ✅ CORRECT — RAM error log + LED blink playback
#define ERROR_LOG_SIZE   16

typedef struct {
    uint32_t timestamp_ms;   // When the error occurred
    uint8_t  error_code;     // What went wrong
    uint8_t  state_at_error; // Which state the car was in
    uint16_t spare_data;     // Context-specific (e.g., sensor value at error time)
} ErrorLogEntry_t;

static ErrorLogEntry_t g_error_log[ERROR_LOG_SIZE];
static volatile uint8_t g_error_log_idx = 0;

// Error codes (keep under 256 for uint8_t)
#define ERR_I2C_TIMEOUT      0x01
#define ERR_STATE_TIMEOUT    0x02
#define ERR_PID_NAN          0x03
#define ERR_VOLTAGE_SAG      0x04
#define ERR_ENCODER_SLIP     0x05
#define ERR_WIRELESS_LOST    0x06
#define ERR_IMU_STALE        0x07
#define ERR_EMERGENCY_STOP   0x08
#define ERR_WATCHDOG_RESET   0x09   // Check RCC_CSR register for IWDG reset flag
#define ERR_HARDFAULT        0x0A   // Only if we somehow recover from HardFault
#define ERR_SENSOR_CONFIDENCE 0x0B

void ErrorLog_Write(uint8_t code, uint16_t spare) {
    if (g_error_log_idx >= ERROR_LOG_SIZE) {
        // Buffer full — overwrite oldest entry (circular buffer)
        // Shift all entries left by 1, lose oldest
        for (int i = 0; i < ERROR_LOG_SIZE - 1; i++) {
            g_error_log[i] = g_error_log[i + 1];
        }
        g_error_log_idx = ERROR_LOG_SIZE - 1;
    }

    g_error_log[g_error_log_idx].timestamp_ms = GetTick();
    g_error_log[g_error_log_idx].error_code = code;
    g_error_log[g_error_log_idx].state_at_error = (uint8_t)g_car_state;
    g_error_log[g_error_log_idx].spare_data = spare;
    g_error_log_idx++;
}

// === POST-MORTEM PLAYBACK via LED ===
// Trigger: long-press button (2 seconds) while motors stopped
void ErrorLog_PlaybackLED(void) {
    if (g_error_log_idx == 0) {
        // No errors → single slow blink
        LED_BlinkPattern(0x01);  // 1 blink, long pause, repeat
        return;
    }

    // Blink error count first: N fast blinks = number of errors
    for (int i = 0; i < g_error_log_idx; i++) {
        GPIO_ResetBits(GPIOC, GPIO_Pin_13);  // LED ON
        Delay_ms(150);
        GPIO_SetBits(GPIOC, GPIO_Pin_13);    // LED OFF
        Delay_ms(150);
    }
    Delay_ms(1000);  // 1-second gap between count and data

    // Then blink each error code in binary (8 bits, MSB first)
    for (int i = 0; i < g_error_log_idx; i++) {
        uint8_t code = g_error_log[i].error_code;
        for (int bit = 7; bit >= 0; bit--) {
            if (code & (1 << bit)) {
                GPIO_ResetBits(GPIOC, GPIO_Pin_13);  // LED ON = 1
            } else {
                GPIO_SetBits(GPIOC, GPIO_Pin_13);    // LED OFF = 0
            }
            Delay_ms(300);  // 300ms per bit — human-readable
        }
        // Gap between error entries
        GPIO_SetBits(GPIOC, GPIO_Pin_13);
        Delay_ms(1500);
    }
}

// Watchdog reset detection (check at startup):
void ErrorLog_CheckWDReset(void) {
    if (RCC_GetFlagStatus(RCC_FLAG_IWDGRST) != RESET) {
        RCC_ClearFlag();  // Clear all reset flags
        // We just recovered from a watchdog reset — log it!
        ErrorLog_Write(ERR_WATCHDOG_RESET, 0);
        // Don't clear g_error_log_idx — preserve the log across reset!
        // (This requires the error log to be in a no-init RAM section)
    }
}
```

**Verification rules:**
- A RAM-based error log buffer (≥ 10 entries, circular) must exist, storing at minimum: timestamp, error code, and state at error time.
- The error log must be written to on: I2C timeout, state machine timeout, PID NaN detection (Item 13), voltage sag (Item 50), encoder slip (Item 45), wireless timeout (Item 33/52), and emergency stop (Item 58).
- Post-mortem playback must be triggerable WITHOUT a serial console: long-press button → LED blinks error codes in a human-readable pattern (binary, Morse, or blink count). The pattern must be documented in code comments or a README.
- For watchdog reset survival: the error log buffer should be placed in a **no-init RAM section** (not zeroed at startup) so the pre-reset log survives. Alternatively, write critical errors to a dedicated Flash page (with Item 53's caveat — only when motors are stopped by the error condition itself).
- The LED playback speed must be slow enough for a human to count: ≥ 200ms per bit. A smartphone video recording of the LED can be reviewed frame-by-frame later.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — Black-box error log

  Add to your K题小车:
  1. ErrorLogEntry_t g_error_log[16] in RAM.
  2. ErrorLog_Write() calls at every error detection point:
     - algorithm.c: state timeout → ERR_STATE_TIMEOUT
     - algorithm.c: PID NaN → ERR_PID_NAN
     - mpu6050.c: I2C timeout → ERR_I2C_TIMEOUT
     - sensors.c: confidence invalid → ERR_SENSOR_CONFIDENCE
  3. Post-mortem trigger: hold button 2s while motors stopped.
  4. LED playback: blink error count → pause → blink codes in binary.
  5. Document the blink pattern so you can decode it at competition.
```

---

### Item 60: Dual-Config Fallback (Safe Mode Hardware Pin)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | You tuned aggressive PID gains for maximum speed. On competition day, the floor is slightly different → car oscillates → you have 2 minutes between runs → you can't re-tune 6 PID parameters in 2 minutes → you either run the aggressive tune (risk crash) or forfeit the run |
| **AI Common Mistake** | Stores one set of PID parameters. AI assumes the tuned values are correct and final. The concept of "revert to known-good defaults via hardware pin" is alien to AI. |

**What to check:**

```c
// ❌ FAIL — single parameter set, no fallback
#define SPEED_KP   2.0f    // Only one version exists
#define SPEED_KI   0.5f    // If this is wrong, you MUST reflash

// ✅ CORRECT — dual config with hardware pin selection
#define SAFE_MODE_PIN    GPIO_Pin_15   // PA15 (freed by JTAG disable — Item 42)
#define SAFE_MODE_PORT   GPIOA

typedef struct {
    float Kp, Ki, Kd;
    float output_limit, integral_limit;
    uint32_t crc32;
} PID_ConfigSet_t;

typedef struct {
    PID_ConfigSet_t conservative;  // Version A: guaranteed to finish (slow but stable)
    PID_ConfigSet_t aggressive;    // Version B: fast but risky
    uint8_t active_set;            // Which set is currently loaded? (0=A, 1=B)
    uint32_t config_crc32;         // CRC over entire struct
} PID_DualConfig_t;

static PID_DualConfig_t g_pid_dual_config;

// Must be called VERY EARLY in main(), before ANY peripheral init
uint8_t Boot_CheckSafeMode(void) {
    GPIO_InitTypeDef g;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    g.GPIO_Pin = SAFE_MODE_PIN;
    g.GPIO_Mode = GPIO_Mode_IPU;  // Pull-up: normally HIGH = aggressive mode
    g.GPIO_Speed = GPIO_Speed_2MHz;  // Slow is fine — read once at boot
    GPIO_Init(SAFE_MODE_PORT, &g);

    // Read pin level (before any other GPIO config might affect it)
    uint8_t pin_level = GPIO_ReadInputDataBit(SAFE_MODE_PORT, SAFE_MODE_PIN);

    if (pin_level == Bit_RESET) {
        // Pin is grounded → SAFE MODE → load conservative config
        return 1;  // SAFE MODE active
    }
    return 0;  // Normal mode — use aggressive config
}

void PID_Config_LoadDual(void) {
    Flash_Read(PID_FLASH_ADDR, (uint32_t*)&g_pid_dual_config, sizeof(g_pid_dual_config));

    // Verify CRC
    uint32_t calc_crc = CRC32_Calc((uint32_t*)&g_pid_dual_config,
                                    sizeof(g_pid_dual_config) - 4);
    if (calc_crc != g_pid_dual_config.config_crc32) {
        // Flash empty/corrupt → load hardcoded factory defaults
        PID_Config_LoadFactoryDefaults();
        return;
    }

    uint8_t safe_mode = Boot_CheckSafeMode();

    PID_ConfigSet_t *selected;
    if (safe_mode) {
        selected = &g_pid_dual_config.conservative;
        g_pid_dual_config.active_set = 0;
        // Signal safe mode: LED slow blink (different from normal boot)
        LED_BlinkPattern(0x03);  // 3 slow blinks = safe mode
    } else {
        selected = &g_pid_dual_config.aggressive;
        g_pid_dual_config.active_set = 1;
    }

    // Apply selected config
    PID_Init(&g_speed_pid,   selected[0].Kp, selected[0].Ki, selected[0].Kd, ...);
    PID_Init(&g_heading_pid, selected[1].Kp, selected[1].Ki, selected[1].Kd, ...);
    // (In practice: store all controllers' params in the config struct)

    // If safe mode: LOCK Flash writes (Item 53) — don't allow saving over conservative set
    if (safe_mode) {
        g_flash_write_locked = 1;  // Prevent PID_Config_MaybeSave() from writing
    }
}

// Factory defaults — hardcoded conservative values that ARE tested and WILL finish
static void PID_Config_LoadFactoryDefaults(void) {
    // Speed PID: very conservative
    g_pid_dual_config.conservative.Kp = 1.5f;   // Low gain → won't oscillate
    g_pid_dual_config.conservative.Ki = 0.3f;
    g_pid_dual_config.conservative.Kd = 0.05f;
    g_pid_dual_config.conservative.output_limit = 50.0f;   // 50% max speed
    g_pid_dual_config.conservative.integral_limit = 10.0f;

    // Heading PID: very conservative
    g_pid_dual_config.conservative.Kp = 2.0f;
    // ... etc

    // Aggressive: copied from conservative initially (user tunes later)
    g_pid_dual_config.aggressive = g_pid_dual_config.conservative;
    g_pid_dual_config.active_set = 0;

    PID_Config_SaveDual();  // Write to Flash for persistence
}
```

**Verification rules:**
- Two complete PID parameter sets must exist in persistent storage: "conservative" (factory-tested, guaranteed to finish) and "aggressive" (user-tuned for speed).
- A hardware pin with internal pull-up must be read ONCE at boot, BEFORE any other GPIO configuration. If the pin is LOW (externally grounded via jumper): force-load conservative set, lock Flash writes, signal via LED pattern.
- The factory defaults must be **physically tested** — not guessed. "Conservative" means: this exact config has completed the full course at least once without incident. Document the test date and conditions in a comment.
- The hardware safe-mode pin should be clearly labeled on the PCB silkscreen: "SAFE MODE — SHORT TO GND FOR RECOVERY."
- This is NOT the same as Item 25 (single-set Flash storage) or Item 37 (button handling). It's a **hardware-level configuration selector** that works before any software initialization. If the aggressive config causes an instant HardFault on boot, the user can ground the pin, power-cycle, and the firmware will boot into safe mode before any PID code runs.

**Corrective action if FAIL:**
```
⚠️ RECOMMENDATION — Dual-config safe mode

  For your K题小车:
  1. Reserve one Flash page for PID_DualConfig_t (2 complete PID sets + CRC).
  2. Choose a free GPIO with internal pull-up (e.g., PA15, freed if JTAG disabled).
  3. At boot, check pin level BEFORE any PID init.
  4. If LOW: load conservative set, lock Flash writes, blink LED 3× slow.
  5. Factory defaults: set speed PID output limit to 50%, heading PID Kp low.
     Test this conservative config on the full course at least once.
  6. Label the safe-mode pin on your PCB/enclosure.

  Competition recovery procedure:
  1. Aggressive tune fails → car crashes.
  2. Pull battery, short SAFE_MODE pin to GND with jumper.
  3. Reconnect battery → car boots in safe mode (LED 3 slow blinks).
  4. Run the course at reduced speed → at least you finish and get a score.
```

---

