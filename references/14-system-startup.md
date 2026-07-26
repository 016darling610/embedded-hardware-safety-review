## Dimension 15: System Startup & Debug Integrity

These items prevent the worst-case scenario: a bricked board or a HardFault with no debugger access, at the competition venue with 30 minutes left on the clock.

---

### Item 38: Stack Overflow Prevention (HardFault Hard Fall)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | `float FilterBuffer[256]` declared as local variable inside function → 1KB on stack → function called from ISR with already-partial stack → stack pointer crosses into heap/global data → registers corrupted → HardFault → motors freeze at last PWM value (Item 10 not implemented → no IWDG → permanent freeze) |
| **AI Common Mistake** | Declares large local arrays, deep function call chains, recursive patterns — all on the Cortex-M's tiny default stack (often 512 bytes) |

**What to check:**

```c
// ❌ CRITICAL FAIL — large local array in function
void ProcessSensorData(void) {
    float temp_buffer[256];   // 256 × 4 = 1024 bytes on stack!
    // Most STM32F103 startup files default Stack_Size = 0x00000400 (1024 bytes)
    // This single array consumes the ENTIRE stack → next function call overwrites heap
    for (int i = 0; i < 256; i++) {
        temp_buffer[i] = ComputeSomething(i);
    }
    // ...
}

// ✅ CORRECT — static or global for large arrays
static float g_sensor_buffer[256];  // Global: allocated in .bss, NOT on stack

void ProcessSensorData(void) {
    for (int i = 0; i < 256; i++) {
        g_sensor_buffer[i] = ComputeSomething(i);  // Safe: no stack consumption
    }
}
```

**Verification rules:**
- **Check startup file**: `startup_stm32f10x_md.s` (or equivalent). `Stack_Size` must be ≥ `0x00000800` (**2KB minimum**) for any motor control project. 8KB (`0x00002000`) recommended if using MPU6050 DMP or floating-point math.
- **Zero-tolerance**: NO local arrays larger than 64 bytes inside any function. Large buffers must be `static` or global.
- **Zero-tolerance for ISRs**: NO local arrays of ANY size in interrupt service routines. The ISR stack is even more constrained (often the Main stack pointer vs Process stack pointer).
- Search for `float ...[...];` or `uint8_t ...[...];` declarations inside function bodies. Flag any with size > 64 as FAIL.
- Recursive functions: NONE allowed on Cortex-M without an RTOS with stack overflow detection. AI sometimes writes recursive pathfinding or tree traversal for obstacle maps → immediate HardFault.
- Stack overflow detection: recommend using the MPU (Memory Protection Unit) to set a guard region below the stack. On overflow → MemManage fault → safe shutdown (better than HardFault corruption). Alternatively, fill the stack with a known pattern at startup and periodically check the high-water mark.

**Corrective action if FAIL:**
```
⚠️ CRITICAL FIX — Stack size check

  1. Open startup_stm32f10x_md.s (your project's startup file).
  2. Find: Stack_Size  EQU  0x00000400 (or similar).
  3. Change to: Stack_Size  EQU  0x00002000 (8KB).
  4. In ALL .c files: move arrays larger than 64 bytes from local scope
     to static or global scope.
  5. In ALL ISR handlers: verify no local arrays exist.
  6. Optional: add stack high-water mark check:
     - Fill stack with 0xDEADBEEF pattern at startup.
     - Periodically scan from stack top → find first non-pattern word.
     - Report "stack used: X / Y bytes" over UART.
```

---

### Item 40: HSE Crystal Frequency Verification

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Board has 12MHz crystal but AI defaults to `HSE_VALUE = 8000000` → PLL configured for 8MHz × 9 = 72MHz → actual frequency = 12MHz × 9 = 108MHz (overclock!) → MCU overheats, PWM frequencies wrong, UART baud rates wrong, timer periods wrong, `Delay_ms(1000)` actually ≈ 667ms |
| **AI Common Mistake** | Never checks the actual board crystal; always assumes 8MHz for STM32F1, 25MHz for STM32F4 |

**What to check:**

```c
// ❌ CRITICAL FAIL — mismatched HSE_VALUE
// In stm32f10x.h (or stm32f4xx_hal_conf.h):
#define HSE_VALUE    ((uint32_t)8000000)   // Assumes 8MHz — is your board 8MHz or 12MHz?

// If your board has a 12MHz crystal:
// - PLL configured as 8MHz × 9 = 72MHz → actually 12 × 9 = 108MHz
// - APB1 = 108/2 = 54MHz (rated for 36MHz max!) → peripheral damage possible
// - TIM3 PWM: 108MHz / 72 / 100 = 15kHz → actually 108MHz / 72 / 100 = 15kHz
//   Wait, this is independent of HSE if PCLK is correct... but PCLK is wrong
// - UART: 108MHz / 115200 = wrong divider → garbage on serial

// ✅ CORRECT — verify and hard-code the actual frequency
// Step 1: Find the crystal marking on your board (look at the metal can near the MCU)
// Step 2: Hard-code in system_stm32f10x.c or stm32f10x.h:
#define HSE_VALUE    ((uint32_t)12000000)   // CONFIRMED: board has 12MHz crystal!

// Step 3: VERIFY with hardware — toggle a GPIO, measure with logic analyzer
void Clock_Verify_Test(void) {
    GPIO_InitTypeDef g;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    g.GPIO_Pin = GPIO_Pin_15;
    g.GPIO_Mode = GPIO_Mode_Out_PP;
    g.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &g);

    while (1) {
        // Toggle at exactly 1kHz if clock is correct
        GPIO_SetBits(GPIOA, GPIO_Pin_15);
        Delay_ms(500);   // 500ms HIGH
        GPIO_ResetBits(GPIOA, GPIO_Pin_15);
        Delay_ms(500);   // 500ms LOW → 1Hz square wave on scope
    }
    // Measure with logic analyzer/oscilloscope:
    //   Period = 1.000s → clock is correct
    //   Period = 0.667s → HSE is 12MHz but HSE_VALUE is 8MHz → fix HSE_VALUE!
}
```

**Verification rules:**
- `HSE_VALUE` must match the **physical crystal on the PCB**, not the chip datasheet default. Read the crystal can marking or measure with an oscilloscope.
- Common traps: STM32F103C8 "Blue Pill" clones often use 12MHz (genuine: 8MHz). STM32F407 Discovery: 8MHz (not 25MHz — the 25MHz is for the ST-Link MCU). ESP32: 40MHz crystal, but most modules have it onboard.
- Always verify with a GPIO toggle test: 500ms HIGH, 500ms LOW → measure period with logic analyzer. If period ≠ 1.000s, the clock config is wrong.
- This also affects the **independent watchdog** (IWDG uses LSI, which is independent — PHEW) but **window watchdog** (WWDG) uses APB1 clock and would be affected.
- On STM32F1 Standard Peripheral Library: the startup file calls `SystemInit()` which sets `SystemCoreClock = 72000000` based on `HSE_VALUE`. If `HSE_VALUE` is wrong, `SystemCoreClock` is wrong, and `SysTick_Config(SystemCoreClock / 1000)` produces wrong 1ms ticks.

**Corrective action if FAIL:**
```
⚠️ CRITICAL FIX — Verify HSE crystal frequency

  1. Read the metal can marking on the crystal next to the MCU.
     Common markings:
     - "8.000" → 8MHz
     - "12.000" → 12MHz
     - "25.000" → 25MHz (STM32F4xx common)

  2. Set HSE_VALUE to the actual frequency in the HAL/StdPeriph config header.

  3. If you've been running with the wrong HSE_VALUE:
     - ALL your PID gains are wrong (dt in I-term is based on wrong tick rate)
     - ALL your encoder speed calculations are wrong
     - ALL your ultrasonic pulse timing is wrong
     → You must re-tune everything after fixing the clock!

  4. Flash the GPIO toggle test firmware and VERIFY with oscilloscope/logic analyzer
     before trusting any timing-critical code.
```

---

### Item 41: AI-Deleted Power-Up Delays (Restore Them)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | AI "optimizes" by deleting `HAL_Delay(100)` after MPU6050 init → MPU6050 hasn't finished internal PLL lock → first I2C read returns garbage → attitude initialized with garbage values → yaw offset = random angle → car drives in wrong direction from frame 1 |
| **AI Common Mistake** | Suggests removing "unnecessary delays" to improve loop speed, not understanding that sensors need power-up settling time |

**What to check:**

```c
// ❌ CRITICAL FAIL — delays deleted by AI "optimization"
void System_Init(void) {
    MPU6050_Init();    // AI deleted the Delay_ms(50) after PWR_MGMT_1 write
    OLED_Init();        // AI deleted the Delay_ms(100) after display init
    Motor_Init();
    // Immediately starts reading sensors → reads garbage → car misbehaves
}

// ✅ CORRECT — mandatory delays with retry-on-fail
void System_Init(void) {
    MPU6050_Init();         // Internal: writes 0x80 to PWR_MGMT_1 → 100ms delay → writes 0x01
    Delay_ms(50);           // MPU6050 PLL lock time (datasheet: 30ms typ, 50ms safe)

    OLED_Init();            // Internal: charge pump needs 100ms to reach 9V for OLED panel
    Delay_ms(100);

    Motor_Init();
    // ADD: verify that initialization actually succeeded
    if (!MPU6050_VerifyInit()) {   // Read WHO_AM_I register (0x75, should be 0x68)
        // Retry up to 3 times
        for (int retry = 0; retry < 3; retry++) {
            Delay_ms(200);
            MPU6050_Init();
            if (MPU6050_VerifyInit()) break;
        }
        if (!MPU6050_VerifyInit()) {
            // MPU6050 dead → enter safe mode, don't run motors with no attitude
            LED_Alarm_Blink();
            while (1) { IWDG_ReloadCounter(); }
        }
    }
}
```

**Verification rules:**
- Power-up delays after peripheral initialization are NOT optional. They exist because the peripherals have physical settling times.
- Key delays that AI likes to delete:
  - MPU6050: 100ms after reset (PWR_MGMT_1[7]=1), 50ms after wake (PWR_MGMT_1=0x01), 30ms after PLL select (PWR_MGMT_1=0x03 for gyro-Z reference)
  - OLED SSD1306: 100ms after VCC stable, 100ms after charge pump enable (0x8D → 0x14)
  - TB6612FNG: 100µs after STBY HIGH before PWM input (AI often confuses this with a "useless delay")
  - HC-SR04: 2ms between triggers (takes time for echo to die out from previous measurement)
- After every peripheral init, add an **init verification read** (read WHO_AM_I register, read a known-status register). If verification fails, retry up to 3 times. If still failing, enter safe mode — do not continue with uninitialized sensors.
- AI-generated code almost never has init verification. This is 100% on you to add.

**Corrective action if FAIL:**
```
⚠️ REQUIRED — Restore deleted delays + add init verification

  Your existing code (mpu6050.c:172-178) already has the correct delays:
  Delay_ms(100) after reset (line 174)
  Delay_ms(50) after wake (line 178)
  → KEEP THESE. Do NOT let AI remove them.

  Add after each init function:
  uint8_t whoami = MPU6050_ReadReg(0x75);
  if (whoami != 0x68) { /* retry or error */ }

  Same for OLED: write a test pixel, read back display status, verify it's on.
```

---

### Item 42: SWD Pin Protection — Don't Let AI Brick Your Board

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | AI configures PA13 (SWDIO) or PA14 (SWCLK) as GPIO_Out_PP to get "one more LED pin" → next flash attempt fails → "No Target Connected" in Keil/IAR → board is BRICKED → must use BOOT0 jumper + reset timing trick to erase Flash |
| **AI Common Mistake** | Treats PA13/PA14 as "free GPIOs" without understanding they're the debug interface |

**What to check:**

```c
// ❌ CRITICAL FAIL — SWD pins repurposed as GPIO
// In any GPIO init function, AI writes:
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13 | GPIO_Pin_14 | GPIO_Pin_15;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
GPIO_Init(GPIOA, &GPIO_InitStructure);
// PA13 and PA14 are now GPIO outputs → debugger disconnected → BRICK!

// ✅ CORRECT — conditional compilation protects SWD during development
// NEVER configure PA13 or PA14 unless explicitly unguarded for production

// Method 1: Exclude SWD pins from GPIO config entirely
void GPIO_Config(void) {
    // PC13: onboard LED (NOT PA13!)
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;  // This is PC13, not PA13!
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOC, &GPIO_InitStructure);
    // PA13 and PA14 are NEVER touched → they remain SWD by default
}

// Method 2: If you MUST reuse SWD pins (only in production build):
#if defined(PRODUCTION_BUILD)
    // Reuse PA13/PA14 as GPIO ONLY if you accept the risk
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);
    GPIO_PinRemapConfig(GPIO_Remap_SWJ_Disable, ENABLE);  // Disable SWJ → full SWJ disable
    // or GPIO_Remap_SWJ_JTAGDisable → disable JTAG, keep SWD
    // Now PA13=GPIO, PA14=GPIO, PA15=GPIO, PB3=GPIO, PB4=GPIO
    // WARNING: from this point, SWD is GONE. Only system bootloader can recover.
#endif

// Method 3: Better debug safety — disable JTAG, keep SWD
// (Frees PB3/PB4/PA15, but keeps PA13/PA14 as SWD)
GPIO_PinRemapConfig(GPIO_Remap_SWJ_JTAGDisable, ENABLE);
// Now you have 3 extra GPIOs (PB3, PB4, PA15) AND still have SWD!
```

**Verification rules:**
- **Absolute rule**: NEVER configure PA13 or PA14 in development firmware. If the code touches these pins, it's a FAIL unless `#ifdef PRODUCTION_BUILD` guards them.
- Search for `GPIO_Pin_13` in GPIOA context (NOT GPIOC). PA13 = SWDIO, PA14 = SWCLK. Many boards have an LED on PC13 (GPIOC Pin 13) — this is fine and completely unrelated to PA13.
- `GPIO_Remap_SWJ_JTAGDisable` is the safe option: it disables only JTAG (frees PB3/PB4/PA15), leaving SWD (PA13/PA14) functional. This gives you 3 extra GPIOs without losing debug access.
- `GPIO_Remap_SWJ_Disable`: disables BOTH JTAG and SWD. Only use in production build, annotated with `// WARNING: SWD permanently disabled`.
- **Recovery procedure** if you accidentally brick a board:
  1. Set BOOT0 jumper to HIGH (connect to 3.3V).
  2. Press and release RESET.
  3. The MCU boots into system bootloader (no user code runs → SWD pins stay as SWD).
  4. Use STM32CubeProgrammer or `st-flash erase` to mass-erase the Flash.
  5. Set BOOT0 back to LOW, press RESET → board is unbricked.

**Corrective action if FAIL:**
```
🔴 CRITICAL — SWD pins being repurposed

  Your K題小车 code (main.c:99, line 154) uses:
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_8 | GPIO_Pin_9 | GPIO_Pin_10 | GPIO_Pin_11;
    → These are PA8-PA11 — SAFE. Not SWD pins. ✓

  BUT verify in the FULL project: search ALL .c files for GPIO_Pin_13 and GPIO_Pin_14
  used with GPIOA (not GPIOC or GPIOB). If found and not guarded by #ifdef → remove.
  
  PC13 (GPIOC Pin 13) ≠ PA13 (GPIOA Pin 13, SWDIO).
  This is the #1 source of confusion: your LED on PC13 is fine,
  but AI may confuse it with PA13 when suggesting "add another LED."
```

---

