## Quick Reference: AI's Top 60 Embedded Safety Sins

**D1 — Physical (1-5):**
1. PWM=15kHz? 2. PID≤85%? 3. IN pins IN/IPU/AF_OD? 4. Dead zone measured? 5. ADC+MotorDisable()?

**D2 — Sensors (6-9):**
6. ResetYaw all statics+FIFO? 7. Integral separation+clamp? 8. No delay(), timer ISR? 9. TI12+MA filter?

**D3 — Logic (10-13):**
10. IWDG main loop only? 11. PWM=0%→100ms→EN=HIGH? 12. ISR prio: sensor 0, UART 14? 13. safe_asin/safe_sqrt+isnan()?

**D4 — [RESERVED 14-17]:** ⏳

**D5 — Power (18-20):**
18. Soft ramp 50ms? 19. 100nF+10µF MPU6050? 20. [SERVO] 10°-170°, >2s→center?

**D6 — Bus (21-22):**
21. I2C≤10ms, >5 fail→RCC clock-gate+power-cycle? 22. UART DMA+IDLE?

**D7 — Control (23-24):**
23. [BALANCE] Pitch/Roll>30°→disable? 24. Speed×(1-|steer|×coeff)?

**D8 — Data (25-26):**
25. PID Flash+CRC? 26. printf≤10Hz, #ifdef DEBUG?

**D9 — Code (27-28):**
27. ISR globals volatile? 28. (float) cast, clamp before (int16_t)?

**D10 — Kinematics (29-30):**
29. [MECANUM] Single-wheel test? 30. EMA/ramp on speed setpoint?

**D11 — Calibration (31, 35):**
31. Gyro bias dynamic? 35. [VISION] Confidence+fallback?

**D12 — CPU & DMA (32, 39):**
32. ISR GPIO toggle<50%? 39. DMA double-buffer?

**D13 — Wireless (33-34):**
33. [WIRELESS] 500ms+lockout? 34. PID bumpless transfer?

**D14 — State Machine (36-37):**
36. State timeout→STATE_ERROR? 37. Button poll 10ms+3 reads?

**D15 — System (38, 40-42):**
38. Stack≥0x2000? 40. HSE=xtal+GPIO verified? 41. Delays+WHO_AM_I? 42. PA13/14 untouched?

**D16 — Battery & Motion (43-46):**
43. cm/s, V feedforward? 44. Accel weight dynamic? 45. [ENCODER] Slip? 46. Chassis type?

**D17 — Fusion & Comp (47-49):**
47. [CAM+IMU] Timestamps? 48. [COMP] Start triple+finish triple? 49. Friction adaptive?

**D18 — Field Survival (50-54):**
50. V sag+coulomb? 51. [VISION] OTSU adaptive? 52. [WIRELESS] Co-channel? 53. Flash only stopped? 54. L/R calibrated?

**D19 — Defense-in-Depth (55-57):**
55. I2C→RCC clock off→re-init? 56. Clock tree math in comments? 57. Inline if(pwm>MAX) before TIM_SetCompare?

**D20 — Emergency & Post-Mortem (58-60):**
58. NC button EN→GND + EXTI highest prio? ISR: PWM=0 + EN=LOW only?
59. RAM error log (≥16 entries)? Long-press→LED blink playback? Error codes documented?
60. Flash dual config (conservative+aggressive)? Safe-mode pin at boot? Factory defaults tested?
