## Dimension 8: Data Persistence & Debugging Ergonomics (How Many Nights You Spend Tuning)

These aren't physical safety items, but they determine how efficiently you can tune the system. AI never generates these proactively.

---

### Item 25: PID Parameter Persistent Storage

| Aspect | Detail |
|--------|--------|
| **Consequence** | Every PID re-tune requires editing `#define KP 1.2`, recompiling, reflashing. 100 tuning iterations = 100 flash cycles = hours of wasted time. |
| **AI Common Mistake** | Hardcodes PID constants as `#define` or `const float`. Never mentions EEPROM or Flash storage. |

**What to check:**

```c
// ❌ FAIL — hardcoded constants, reflash for every change
#define SPEED_KP   2.0f
#define SPEED_KI   0.5f
#define SPEED_KD   0.1f

// ✅ CORRECT — runtime-configurable PID with Flash storage
typedef struct {
    float Kp, Ki, Kd;
    float output_limit, integral_limit;
    uint32_t crc32;  // Checksum to detect corruption
} PID_Config_Flash_t;

static PID_Config_Flash_t g_pid_config;

// Load from Flash on boot, use defaults if Flash is empty
void PID_Config_Load(void) {
    Flash_Read(PID_FLASH_ADDR, (uint32_t*)&g_pid_config, sizeof(g_pid_config));
    uint32_t calc_crc = CRC32_Calc((uint32_t*)&g_pid_config, sizeof(g_pid_config) - 4);
    if (calc_crc != g_pid_config.crc32) {
        // Flash empty or corrupted → use factory defaults
        g_pid_config.Kp = 2.0f;
        g_pid_config.Ki = 0.5f;
        g_pid_config.Kd = 0.1f;
        g_pid_config.output_limit = 85.0f;
        g_pid_config.integral_limit = 20.0f;
    }
}

// Save to Flash
void PID_Config_Save(void) {
    g_pid_config.crc32 = CRC32_Calc((uint32_t*)&g_pid_config, sizeof(g_pid_config) - 4);
    Flash_ErasePage(PID_FLASH_ADDR);
    Flash_Write(PID_FLASH_ADDR, (uint32_t*)&g_pid_config, sizeof(g_pid_config));
}
```

**Verification rules:**
- PID constants must be runtime-writable, not compile-time constants.
- Storage target: internal Flash (last page reserved for config), external EEPROM (AT24C02 via I2C), or emulated EEPROM in Flash (STM32F1 has no true EEPROM).
- Include a **CRC32 or checksum** to detect corrupted/uninitialized Flash. Factory defaults must be used when CRC fails.
- Provide a **calibration mode**: long-press a button → enter calibration → use UART/Bluetooth/knobs to adjust Kp/Ki/Kd → short-press to save → LED confirms save.
- This item is not a safety-critical FAIL — flag it as ⚠️ RECOMMENDATION rather than ❌ FAIL.

**Corrective action if FAIL:**
```
⚠️ RECOMMENDATION — Add PID persistent storage

  For STM32F103C8 (no EEPROM):
  - Reserve the last 1KB Flash page for config storage.
  - Use the STM32 Flash API: FLASH_ErasePage(), FLASH_ProgramWord().
  - Add a calibration mode triggered by a button long-press.
  - Store a CRC32 alongside the config for corruption detection.
  - Format: struct { float Kp, Ki, Kd; float out_limit, i_limit; uint32_t crc; }

  For chips with EEPROM (STM32L0, ESP32):
  - Use the EEPROM emulation library or Preferences (ESP32).
  - Much simpler — no page erase required.

  This is NOT a safety-critical item, but it will save you hours of reflashing.
```

---

### Item 26: Non-Blocking Debug Logging

| Aspect | Detail |
|--------|--------|
| **Consequence** | `printf()` in main loop → 115200 baud UART transmits ~11.5 KB/s → each `printf("yaw=%.2f pitch=%.2f roll=%.2f\n", ...)` is ~40 bytes → 3.5 ms blocked → at 200 Hz main loop, that's 70% of CPU time wasted → PWM update jitter → motor vibration |
| **AI Common Mistake** | Puts full `printf()` with floating-point formatting in the main control loop |

**What to check:**

```c
// ❌ FAIL — printf in every control loop iteration
while (1) {
    PID_Compute(...);
    Motor_Drive(...);
    printf("yaw=%.2f pitch=%.2f speed=%d\n", g_sensors.yaw, g_sensors.pitch, speed);
    // This printf takes 3.5 ms → at 200 Hz loop, 70% of time is blocked on UART!
}

// ✅ CORRECT — sparse logging, integer-only
#define LOG_INTERVAL_MS     100    // Print once every 100ms, NOT every loop cycle
#define LOG_INTERVAL_CYCLES 20     // Or once every 20 main loop cycles

static uint32_t last_log_tick = 0;
static uint32_t log_cycle_count = 0;

while (1) {
    PID_Compute(...);
    Motor_Drive(...);
    log_cycle_count++;

    if ((GetTick() - last_log_tick) >= LOG_INTERVAL_MS) {
        last_log_tick = GetTick();
        // Integer-only formatting — much faster than float printf
        int16_t yaw_x10 = (int16_t)(g_sensors.yaw * 10.0f);
        int16_t pitch_x10 = (int16_t)(g_sensors.pitch * 10.0f);
        int16_t speed_int = (int16_t)Motor_GetLeftSpeed();
        printf("Y%d P%d S%d\n", yaw_x10, pitch_x10, speed_int);
        // This printf is ~15 bytes → ~1.3 ms → still heavy but 10x less frequent
    }

    Delay_ms(5);
    IWDG_ReloadCounter();
}
```

**Verification rules:**
- Search for `printf()` inside `while(1)` main loops. If present, verify it fires no more than **10 times per second** (every 100ms minimum interval).
- Float formatting (`%f`, `%.2f`) in printf is **very slow** on Cortex-M without hardware FPU. Prefer integer representations (multiply by 10/100/1000, print as integer, parse on the PC side).
- Better: use a **binary logging protocol** (send raw structs over UART, parse on PC) or **ITM/SWO** (SWO pin to debug probe — zero CPU overhead).
- If UART is also used for command reception, the debug prints must yield priority — commands take precedence over logs.
- Production/competition firmware should have logging behind `#ifdef DEBUG_ENABLED` so it can be compiled out entirely for the race.

**Corrective action if FAIL:**
```c
// Non-blocking sparse log pattern
#define DEBUG_ENABLED  1   // Set to 0 for competition build

#if DEBUG_ENABLED
  #define LOG_INTERVAL_MS  100
  static uint32_t _log_timer = 0;
  #define DEBUG_LOG(fmt, ...) \
      do { \
          if ((uint32_t)(GetTick() - _log_timer) >= LOG_INTERVAL_MS) { \
              _log_timer = GetTick(); \
              printf(fmt, ##__VA_ARGS__); \
          } \
      } while(0)
#else
  #define DEBUG_LOG(fmt, ...)  ((void)0)
#endif

// Usage in main loop:
DEBUG_LOG("Y%d P%d S%d\n", yaw_x10, pitch_x10, speed_int);
```

---

