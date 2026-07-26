## Dimension 12: Real-Time CPU & DMA Integrity

AI generates code that compiles but violates real-time deadlines. These bugs survive compilation and only appear as intermittent glitches.

---

### Item 32: ISR Timing Overrun Verification

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Control ISR designed for 1ms period takes 1.5ms to execute → ISR re-enters before previous invocation finishes → stack grows without bound → HardFault OR main loop starved → PWM frozen → motor whines at fixed frequency |
| **AI Common Mistake** | Runs `double`-precision Kalman filter, `atan2()`, full PID computation, and OLED frame buffer update ALL inside a 1kHz timer ISR |

**What to check:**

```c
// ❌ CRITICAL FAIL — heavy computation in ISR with no timing verification
void TIM7_IRQHandler(void) {
    // 1kHz timer ISR — supposed to take <500µs
    MPU6050_ReadRaw(&raw);           // ~50µs (I2C)
    ComplementaryFilter(&raw, dt);   // ~200µs (float trigonometry)
    PID_Compute(&pid, input, dt);    // ~100µs
    Motor_Drive(speed, steer);       // ~50µs
    OLED_DrawGraph(&sensors);        // ~2000µs!!! OLED I2C in ISR!
    // Total: ~2400µs > 1000µs period → ISR overrun!
}

// ✅ CORRECT — ISR timing measurement + split heavy work to main loop
// Step 1: Toggle a GPIO at ISR entry/exit for oscilloscope measurement
#define ISR_TIMING_PIN    GPIO_Pin_15   // Use any free GPIO for timing probe
#define ISR_TIMING_PORT   GPIOA

void TIM7_IRQHandler(void) {
    GPIO_SetBits(ISR_TIMING_PORT, ISR_TIMING_PIN);  // Timing start

    // MINIMAL work in ISR:
    MPU6050_ReadRaw_DMA(&raw);        // DMA reads I2C → ~10µs CPU time
    sensor_data_ready = 1;            // Signal main loop

    GPIO_ResetBits(ISR_TIMING_PORT, ISR_TIMING_PIN);  // Timing end
    // Measure high-time with oscilloscope → MUST be < 50% of ISR period
}

// Step 2: Heavy math goes in main loop
while (1) {
    if (sensor_data_ready) {
        sensor_data_ready = 0;
        ComplementaryFilter(&raw, dt);    // Float math in main loop
        PID_Compute(&pid, input, dt);
        Motor_Drive(speed, steer);
    }
    OLED_DrawGraph(&sensors);             // Slow I2C in main loop (only when idle)
    IWDG_ReloadCounter();
}

// Step 3: Use float (not double) throughout — Cortex-M3/4 has single-precision FPU only
// ❌ double → software emulation, 10x slower
// ✅ float  → hardware FPU instruction, 1 cycle
```

**Verification rules:**
- Every timer ISR must have a **GPIO toggle** at entry and exit for oscilloscope measurement. The ISR execution time must be **measurable**, not inferred.
- ISR execution time must be **< 50% of the ISR period**. For a 1kHz timer (1000µs period), the ISR must complete in ≤ 500µs. The remaining 500µs are for the main loop and other ISRs.
- Search for blocking operations inside timer ISRs: I2C reads, SPI transfers, UART printf, OLED updates, Flash writes. Any of these in an ISR → FAIL.
- All floating-point operations must use `float`, not `double`. On Cortex-M3 (STM32F103), `double` is software-emulated and ~10× slower than `float`. On Cortex-M4F (STM32F407), `float` is hardware-accelerated but `double` is still software.
- If CMSIS-DSP functions are used (`arm_sqrt_f32`, `arm_mat_mult_f32`), verify they are the `f32` variants, not `f64`.
- For complex filters: if execution time is borderline, reduce iteration count (e.g., Mahony filter with 1 iteration instead of 3; complementary filter with 0.98/0.02 weights instead of Kalman).

**Corrective action if FAIL:**
```
⚠️ NEEDS MANUAL VERIFICATION — ISR timing measurement

  1. Add GPIO toggle at ISR entry/exit (free GPIO pin → oscilloscope probe).
  2. Measure high-time at full control rate (worst case: all sensors active, max PID iterations).
  3. If high-time > 50% of ISR period:
     a. Move heavy computation (filter, PID, OLED) from ISR to main loop.
     b. Change all 'double' to 'float' (especially in atan2, sqrt, sin/cos calls).
     c. Replace blocking I2C reads with DMA-based reads.
     d. If still over budget: reduce ISR frequency (500Hz is acceptable for most ground vehicles).
  4. Document the measured time in a code comment:
     // Measured ISR high-time: 340µs, period: 1000µs (34%) @ STM32F103 72MHz
```

---

### Item 39: DMA-CPU Cache Coherency (Torn Reads)

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | DMA writes ADC data to array while CPU reads the same array → CPU sees half-old half-new data ("torn read") → battery voltage reads 25.3V for one cycle → false low-voltage trigger → motors cut out mid-race |
| **AI Common Mistake** | Configures DMA in circular mode to a single buffer, CPU reads directly from that buffer with no synchronization |

**What to check:**

```c
// ❌ CRITICAL FAIL — shared buffer, no synchronization
static uint16_t adc_dma_buffer[8];  // DMA writes, CPU reads — same buffer!

void ADC_DMA_Init(void) {
    DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;
    DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)adc_dma_buffer;  // Single buffer
    // ...
}

float Battery_ReadVoltage(void) {
    // CPU reads adc_dma_buffer[0] while DMA might be writing it!
    uint16_t raw = adc_dma_buffer[0];  // TORN READ: high byte old, low byte new
    return (raw / 4095.0f) * 3.3f * VOLTAGE_DIVIDER_RATIO;
}

// ✅ CORRECT — double buffering with completion flag
#define ADC_BUF_SIZE  8
static uint16_t adc_buffer_a[ADC_BUF_SIZE];  // Buffer A
static uint16_t adc_buffer_b[ADC_BUF_SIZE];  // Buffer B
static volatile uint8_t active_read_buffer = 0;  // 0=CPU reads B (DMA fills A), 1=CPU reads A (DMA fills B)
static volatile uint8_t buffer_updated = 0;

// DMA HT (Half Transfer) ISR: Buffer A full → switch DMA to Buffer B, signal CPU to read A
void DMA1_Channel1_IRQHandler(void) {
    if (DMA_GetITStatus(DMA1_IT_HT1)) {   // Half-transfer = Buffer A full
        DMA_ClearITPendingBit(DMA1_IT_HT1);
        active_read_buffer = 0;            // CPU should read Buffer B? No — HT means first half done
        // Actually for double-buffering, use TC (Transfer Complete) with M0/M1 addressing
    }
}

// Better approach: DMA with M0/M1 double-buffer mode (STM32F4/F7/H7)
void ADC_DMA_DoubleBuffer_Init(void) {
    DMA_DoubleBufferModeConfig(DMA2_Stream0, (uint32_t)adc_buffer_b, DMA_Memory_0);
    DMA_DoubleBufferModeCmd(DMA2_Stream0, ENABLE);
    // DMA fills Buffer A → interrupt → CPU reads Buffer A, DMA switches to Buffer B → etc.
}

// Simplest approach for STM32F1 (no double-buffer hardware):
static volatile uint8_t dma_complete = 0;
static uint16_t adc_safe_copy[ADC_BUF_SIZE];  // CPU-only buffer

void DMA1_Channel1_IRQHandler(void) {
    if (DMA_GetITStatus(DMA1_IT_TC1)) {
        DMA_ClearITPendingBit(DMA1_IT_TC1);
        // Copy DMA buffer to safe buffer INSIDE the ISR (DMA is stopped after TC in non-circular mode)
        memcpy(adc_safe_copy, adc_dma_buffer, sizeof(adc_safe_copy));
        dma_complete = 1;
    }
}

float Battery_ReadVoltage_Safe(void) {
    if (!dma_complete) return last_valid_voltage;  // Use previous value if DMA not ready
    uint16_t raw = adc_safe_copy[0];  // Safe: this buffer is only written in ISR
    return (raw / 4095.0f) * 3.3f * VOLTAGE_DIVIDER_RATIO;
}
```

**Verification rules:**
- DMA destination buffer and CPU read buffer must be **different memory regions**, or DMA writes must be synchronized with a completion flag/interrupt.
- For circular DMA: CPU must either (a) read only inside the DMA TC/HT interrupt handler, or (b) use hardware double-buffering (DMA M0/M1 mode on STM32F4+).
- Memory barriers: on Cortex-M4/M7 with cache (e.g., STM32F407 with ART accelerator), DMA writes may land in SRAM but the CPU's cache still holds the old value. Use `SCB_CleanInvalidateDCache()` or `__DSB()` before reading DMA buffers.
- Search for all `__HAL_DMA_GET_COUNTER()` or `DMA_GetCurrDataCounter()` calls — these are used to compute how much data DMA has transferred. The counter changes asynchronously; the result is only valid inside the DMA ISR.
- Array alignment: DMA buffers must be aligned to the data width (4-byte alignment for 32-bit, cache-line alignment for cached systems). Use `__attribute__((aligned(32)))` or `ALIGN_32BYTES()`.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — DMA double-buffering

  For STM32F103 (no hardware double-buffer):
  1. Use DMA TC interrupt to copy data to a CPU-safe buffer.
  2. CPU reads ONLY the safe copy, never the DMA target buffer.
  3. Alternatively: disable the DMA stream, read, re-enable — but this
     causes data gaps (~few µs). Acceptable for slow-changing signals
     like battery voltage (read every 100ms).

  For STM32F4/F7/H7:
  1. Use DMA M0/M1 double-buffer mode (hardware auto-switching).
  2. CPU reads the inactive buffer while DMA fills the active one.

  Minimum fix for your K题小车:
  - Battery voltage ADC (Item 5): sample 8× → average in ISR → store result
    in volatile float. Main loop reads the float directly (atomic on Cortex-M3
    for 32-bit values). This avoids DMA entirely for low-rate signals.
```

---

