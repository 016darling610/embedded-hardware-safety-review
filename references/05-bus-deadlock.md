## Dimension 6: Communication Bus Deadlock & Timeout (The Silent Freeze)

These bugs don't crash the MCU — they freeze it silently. Without a watchdog, the car keeps running on the last valid PWM command.

---

### Item 21: I2C/SPI Bus Hang Recovery

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | I2C slave (MPU6050) holds SDA low → `HAL_I2C_Master_Transmit()` waits forever (default timeout = HAL_MAX_DELAY = 0xFFFFFFFF) → main loop dead → motors keep running with no sensor updates → car crashes with no obstacle avoidance |
| **AI Common Mistake** | Uses HAL default infinite timeout, or soft-resets the entire MCU on I2C hang |

**What to check:**

```c
// ❌ CRITICAL FAIL — blocking I2C with no or huge timeout
HAL_I2C_Master_Transmit(&hi2c1, MPU_ADDR, buf, len, 1000);  // 1000ms! Blocked for 1 second.
// or worse:
HAL_I2C_Master_Transmit(&hi2c1, MPU_ADDR, buf, len, HAL_MAX_DELAY);  // INFINITE wait

// ❌ WRONG RECOVERY — reset entire MCU on I2C hang
NVIC_SystemReset();  // MCU reboots → GPIOs float → motors may kick!

// ✅ CORRECT — 10ms timeout + hardware power-cycle the slave device
#define I2C_TIMEOUT_MS      10    // Maximum 10ms per I2C transaction
#define I2C_HANG_RECOVERY_COUNT  5  // After 5 consecutive hangs, power-cycle slave

// Software I2C timeout pattern (your existing code should extend this):
static uint8_t I2C_WaitAck_Timeout(uint32_t timeout_us) {
    uint32_t start = TIM_GetCounter(I2C_TIMEOUT_TIMER);
    I2C_SDA_H(); I2C_Delay();
    I2C_SCL_H(); I2C_Delay();
    while (I2C_SDA_READ()) {
        if ((uint32_t)(TIM_GetCounter(I2C_TIMEOUT_TIMER) - start) > timeout_us) {
            I2C_SCL_L();
            return 1;  // ACK timeout — slave not responding
        }
    }
    I2C_SCL_L();
    return 0;  // ACK received
}

// Power-cycle slave via GPIO-controlled P-MOSFET:
static void MPU6050_PowerCycle(void) {
    GPIO_ResetBits(MPU_PWR_PORT, MPU_PWR_PIN);  // P-MOSFET gate LOW = OFF
    Delay_ms(50);  // Let capacitors discharge
    GPIO_SetBits(MPU_PWR_PORT, MPU_PWR_PIN);    // P-MOSFET gate HIGH = ON
    Delay_ms(100);  // Wait for MPU6050 startup (including PLL lock)
    MPU6050_Init();  // Re-initialize registers
}
```

**Verification rules:**
- Every I2C transaction must have a timeout ≤ **10 ms**. This applies to both HAL and software I2C.
- After N consecutive I2C failures (N ≤ 5), the slave device must be **power-cycled** via GPIO. Soft reset (writing 0x80 to PWR_MGMT_1) is insufficient — a stuck-low SDA can't be cleared by I2C commands.
- During I2C failure and recovery, the **motor control must NOT stall**. Use last-known-good sensor values (marked with a `stale` flag) to decide whether to continue or stop.
- Slave power control requires **hardware support**: the MPU6050 VCC must be switchable via a GPIO-controlled P-MOSFET or load switch. If the PCB does not have this, flag it as ⚠️ NEEDS HARDWARE MODIFICATION.
- SP: same principle — use `HAL_SPI_TransmitReceive()` with a finite timeout, and provide a GPIO-based CS (chip select) reset sequence on timeout.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — I2C timeout + hardware slave reset

  Your existing code (mpu6050.c) already has software I2C + 3 retries + bus recovery
  (9 SCL pulses). This is a good baseline. Three things to add:

  1. Hardware: Add a P-MOSFET on MPU6050 VCC, gate controlled by a free GPIO.
     This lets you power-cycle the MPU6050 without rebooting the MCU.

  2. Software: On 3 consecutive I2C failures, call MPU6050_PowerCycle()
     instead of just retrying.

  3. During I2C recovery window (~150ms), set a flag 'sensors_stale = 1'.
     Main loop reads this flag and either slows down or stops motors.
```

---

### Item 22: UART Receive Overflow Protection

| Aspect | Detail |
|--------|--------|
| **Fatal Consequence** | Host/Bluetooth floods UART RX → Overrun Error (ORE) flag set → UART stops receiving → no more remote commands → car keeps executing last command forever |
| **AI Common Mistake** | Parses commands with `strcmp()` inside the UART RX ISR, or uses a single-byte receive buffer with no overflow handling |

**What to check:**

```c
// ❌ CRITICAL FAIL — UART ISR does string parsing, no overflow protection
void USART1_IRQHandler(void) {
    uint8_t ch = USART_ReceiveData(USART1);
    rx_buf[rx_idx++] = ch;
    if (ch == '\n') {
        rx_buf[rx_idx] = '\0';
        if (strcmp(rx_buf, "FORWARD\n") == 0) { ... }  // strcmp in ISR!
        rx_idx = 0;
    }
    // No ORE check — if overrun occurs, USART stops receiving silently
}

// ✅ CORRECT — DMA circular buffer + IDLE line detection
#define UART_RX_BUF_SIZE  64
static volatile uint8_t uart_rx_dma_buf[UART_RX_BUF_SIZE];
static volatile uint8_t uart_rx_data_ready = 0;

// Init: DMA circular mode
void UART_DMA_Init(void) {
    DMA_InitTypeDef DMA_InitStructure;
    DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&USART1->DR;
    DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)uart_rx_dma_buf;
    DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;
    DMA_InitStructure.DMA_BufferSize = UART_RX_BUF_SIZE;
    DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;  // Circular buffer
    // ... other DMA config ...
    DMA_Init(DMA1_Channel5, &DMA_InitStructure);

    USART_DMACmd(USART1, USART_DMAReq_Rx, ENABLE);
    DMA_Cmd(DMA1_Channel5, ENABLE);

    // Enable IDLE line interrupt
    USART_ITConfig(USART1, USART_IT_IDLE, ENABLE);
}

// IDLE ISR: fires when RX line goes idle after a frame
void USART1_IRQHandler(void) {
    if (USART_GetITStatus(USART1, USART_IT_IDLE) != RESET) {
        USART_ReceiveData(USART1);  // Clear IDLE flag (must read DR + SR)
        DMA_Cmd(DMA1_Channel5, DISABLE);
        uint16_t received_len = UART_RX_BUF_SIZE - DMA_GetCurrDataCounter(DMA1_Channel5);
        DMA_SetCurrDataCounter(DMA1_Channel5, UART_RX_BUF_SIZE);
        DMA_Cmd(DMA1_Channel5, ENABLE);
        uart_rx_data_ready = 1;    // Signal main loop to process
    }
    // Handle Overrun Error
    if (USART_GetFlagStatus(USART1, USART_FLAG_ORE) != RESET) {
        USART_ReceiveData(USART1);  // Read DR to clear ORE
    }
}
```

**Verification rules:**
- Search for `USART_IT_ORE` or `USART_FLAG_ORE` — overrun error must be explicitly handled. Reading the DR register clears ORE.
- UART ISR must NOT call `strcmp()`, `strstr()`, `sscanf()`, `printf()`, or any blocking function. String parsing belongs in the main loop.
- Must use either: (a) DMA circular buffer + IDLE line interrupt, or (b) a ring buffer with head/tail pointers and explicit overflow detection.
- Single-byte interrupt receive is acceptable for low-bandwidth remote control (e.g., 9600 baud joystick), but must still check for ORE and protect against buffer overflow.
- If using an RTOS, the UART ISR should post to a queue/semaphore, not do the parsing itself.

**Corrective action if FAIL:**
```
⚠️ NEEDS IMPLEMENTATION — UART overflow protection

  If UART is currently unused in your project: skip this item.
  If UART is used for Bluetooth/WiFi remote control:

  1. Replace single-byte RX ISR with DMA circular buffer + IDLE interrupt.
  2. Move all command parsing out of ISR into main loop.
  3. Add ORE flag handling in the ISR.
  4. If no DMA channels are free, implement a software ring buffer with:
     - rx_head (ISR writes), rx_tail (main loop reads)
     - If (rx_head + 1) % BUF_SIZE == rx_tail → drop byte (overflow)
     - Check ORE flag each ISR entry
```

---

