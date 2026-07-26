## Dimension 9: AI Code Structural Traps (Programmer Self-Defense)

These are bugs that survive compilation and only manifest intermittently. Hardest to debug.

---

### Item 27: Volatile Qualification on Shared Variables

| Aspect | Detail |
|--------|--------|
| **Consequence** | Variable modified in ISR, read in main loop → compiler caches the value in a register → main loop reads stale register value forever → encoder count "stuck", sensor flag "never set", PWM value "frozen" |
| **AI Common Mistake** | Declares global variables without `volatile`, or marks `volatile` only on simple types but not on structs accessed across ISR/main boundaries |

**What to check:**

```c
// ❌ CRITICAL FAIL — shared variables without volatile
// In ISR:
void SysTick_Handler(void) {
    g_sys_tick_ms++;  // Modified in ISR
}

// In main loop:
uint32_t now = g_sys_tick_ms;  // Compiler may cache g_sys_tick_ms → never sees updates!

// ❌ ALSO WRONG — struct without volatile
SensorData_t g_sensors;  // Modified in multiple functions, read across contexts

// ✅ CORRECT — volatile on every ISR-shared variable
volatile uint32_t g_sys_tick_ms = 0;       // ISR writes, main loop reads
volatile CarState_t g_car_state;            // Modified across state machine paths
volatile SensorData_t g_sensors;            // Multiple writers, multiple readers
volatile uint8_t sensor_data_ready = 0;     // ISR sets, main loop clears
```

**Verification rules:**
- **Every global variable** must be audited: is it written in an ISR? Is it written in one function and read in another (especially across different `.c` files)? If yes → must be `volatile`.
- Struct members don't inherit `volatile` from the struct — if `g_sensors` is `volatile SensorData_t`, all members are volatile. But if `g_sensors` is NOT volatile and a member `g_sensors.yaw` is accessed in both ISR and main, the compiler can cache the member separately. The entire struct must be volatile.
- Stale data is especially dangerous for: encoder counts, sensor data ready flags, system tick, car state, PID setpoints changed via UART commands.
- Read-modify-write on volatile variables in main loop must be protected from ISR preemption during the RMW sequence. If the variable is > 32 bits (e.g., `uint64_t` on Cortex-M3), a single load/store is not atomic — use critical sections (`__disable_irq()` / `__enable_irq()`) or atomic access functions.

**Corrective action if FAIL:**

For your specific K题小车 code, the following variables need `volatile`:
```c
// main.c
volatile uint32_t g_sys_tick_ms = 0;    // ← already has volatile ✓
volatile CarState_t g_car_state;         // ← MISSING volatile!
// g_car_state is modified in Algorithm_RunMission() and Algorithm_StartMission()
// and read in main()'s while loop → MUST be volatile

// sensors.c
volatile SensorData_t g_sensors;         // ← MISSING volatile!
// g_sensors is written by Sensors_UpdateAll(), MPU6050_Update()
// and read by Algorithm_Navigate(), OLED_ShowDebug() → MUST be volatile

// algorithm.c
// CarOdom_t g_odom is declared extern in main.h but its struct members
// are written across different algorithm states → SHOULD be volatile
```

---

### Item 28: Implicit Type Conversion Overflow

| Aspect | Detail |
|--------|--------|
| **Consequence** | `int16_t` encoder count × `float` KP → intermediate int16 overflow before float conversion → garbage PID output → motor jump |
| **AI Common Mistake** | Mixes integer and float types without explicit casts in PID and sensor math |

**What to check:**

```c
// ❌ FAIL — integer overflow before float conversion
int16_t encoder_cnt = 32000;  // Near int16_t max
float speed = encoder_cnt * SPEED_SCALE;  // int16 × float → int16 multiply first!
// 32000 × 100 = 3,200,000 → overflow int16_t → -31072 → garbage!

// ❌ ALSO WRONG — PID output from float to int16 without range check
int16_t pwm_out = (int16_t)PID_Compute(&pid, input, dt);  // Float truncation!

// ✅ CORRECT — explicit cast BEFORE multiply
float speed = (float)encoder_cnt * SPEED_SCALE;  // Cast first → safe float multiply

// ✅ CORRECT — clamp before integer conversion
float pid_out = PID_Compute(&pid, input, dt);
if (pid_out > 32767.0f)  pid_out = 32767.0f;
if (pid_out < -32768.0f) pid_out = -32768.0f;
int16_t pwm_out = (int16_t)pid_out;  // Safe: float was clamped to int16 range
```

**Verification rules:**
- Search for all lines where an integer type (`int16_t`, `int32_t`, `uint8_t`) is multiplied by or divided by a float. The integer must be explicitly cast to `(float)` before the operation.
- Search for all `(int16_t)float_expression` or `(int32_t)float_expression` casts — these are the PID→PWM and sensor→int conversion points. The float must be range-checked before casting.
- **Specific danger zone**: `atan2f()` returns `float` in `[-π, π]`. This value converted to degrees (`180/π`) is `[-180, 180]` — safe for int16. But intermediate calculations (gyro integration, angle wrapping) can produce values outside this range.
- **PID accumulator**: `pid->integral` is a float. If integral_limit is not set, it can grow to ±1e38 and overflow even float precision. This is why integral_limit clamping (Item 7) matters for type safety too.

**Corrective action if FAIL:**
```c
// Safe type conversion macros
#define TO_FLOAT(x)      ((float)(x))   // Always cast before float ops
#define TO_INT16_CLAMP(x) ((int16_t)( \
    ((x) > 32767.0f) ? 32767 : ((x) < -32768.0f) ? -32768 : (int16_t)(x) \
))

// Usage:
float speed = TO_FLOAT(encoder_count) * SPEED_SCALE;    // Safe
int16_t pwm = TO_INT16_CLAMP(pid_output);                // Safe

// In your existing code, fix these patterns:
// algorithm.c:100
// OLD: angular_speed = (int16_t)PID_Compute(&g_heading_pid, g_sensors.yaw, dt);
// NEW:
float heading_raw = PID_Compute(&g_heading_pid, g_sensors.yaw, dt);
angular_speed = TO_INT16_CLAMP(heading_raw);

// algorithm.c:68 (and everywhere you assign float to int16_t)
// OLD: int16_t linear_speed = (int16_t)speed_pct;
// This is relatively safe (speed_pct ≤ 100) but still fragile:
int16_t linear_speed = TO_INT16_CLAMP(speed_pct);
```

---

