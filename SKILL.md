---
name: embedded-hardware-safety-review
description: >-
  60-point hardware safety review for AI-generated embedded firmware code (STM32, DC/BLDC motor control, MPU6050 attitude sensing, PID control, servo, balancing robot, encoder, mecanum, Ackermann, I2C/SPI/UART/DMA comms, wireless remote, state machine, camera/vision fusion, competition logic, pre-race lockdown, defense-in-depth, emergency stop, post-mortem). Use this whenever the user submits embedded code for safety review, asks for code review of motor driver / STM32 / MPU6050 / PWM / PID / encoder / servo / I2C / balancing / mecanum / Ackermann / state-machine / competition code, mentions hardware safety concerns (烧毁、冒烟、失控、炸驱动、卡死、飞车、变砖、HardFault、推头、甩尾、冲线误判、虚电猝死、同频干扰、重心偏移、无日志、无法回退), or says keywords like 嵌入式安全审查、电机驱动审查、STM32安全、PWM安全、MPU6050安全、姿态解算审查、舵机安全、平衡车安全、麦轮安全、竞赛安全、封车安全. Also trigger proactively when you generate embedded motor control, sensor fusion, state-machine, or competition-strategy code — review your own output before presenting it to the user. Covers 20 dimensions across 60 items.
---

# Embedded Hardware Safety Review — 60-Point Checklist

## Purpose

AI-generated embedded code is syntactically correct but **physically dangerous**. LLMs do not understand voltage, current, or thermal dissipation — they treat GPIO configuration, PWM timers, and PID loops as abstract text with no physical consequence. This skill provides a systematic **60-point safety review across 20 dimensions**.

When you apply this skill, you are acting as a **hardware safety auditor**, not a code-style reviewer. Every item must be checked. If an item cannot be verified from the code alone, flag it as **NEEDS MANUAL VERIFICATION** and explain what physical measurement the developer must perform.

---

## Review Process

### Step 1 — Identify Scope

Determine which subsystems are present to know which items are applicable:

| Subsystem | Applicable Items |
|-----------|-----------------|
| Motor driver (PWM, H-bridge) | 1, 2, 3, 4, 9, 11, 18, 30, 43 |
| Battery-powered (LiPo 2S/3S/4S) | 5, 19, 43, 50 |
| MPU6050 / IMU / attitude | 6, 7, 8, 13, 31, 44 |
| Encoder odometry | 9, 45 |
| I2C/SPI sensors | 19, 21, 55 |
| Servo | 20 [conditional] |
| Two-wheel balancing | 23 [conditional] |
| UART/Bluetooth/WiFi | 22, 26, 33, 52 |
| Line-following / vision | 24, 35, 51 |
| Mecanum / omni / swerve | 29, 46 [conditional] |
| Ackermann | 46 [conditional] |
| DMA-based ADC/sensor | 32, 39 |
| Camera + IMU fusion | 47 [conditional] |
| Competition start/finish | 48, 49 [conditional] |
| Wireless remote | 33, 52 [conditional] |
| Encoder present | 9, 45 [conditional] |
| Flash storage for params | 25, 53, 60 |
| Full system | 10, 11, 12, 13, 27, 28, 36, 37, 38, 40, 41, 42, 54, 56, 57, 58, 59 |

### Step 2 — Read the Relevant Modules

For each applicable item, read the corresponding reference file in `references/`. Each file contains:
- ❌ What AI commonly gets wrong
- ✅ The correct implementation with code examples
- 📋 Verification rules (what to check, line by line)
- 🔧 Corrective action if the item fails

### Step 3 — Produce the Report

After checking all applicable items, output a structured report using the template in [`references/output-template.md`](references/output-template.md). Rate overall severity as **CRITICAL / HIGH / MEDIUM / LOW**.

---

## Dimension Map

| # | Dimension | Items | Reference |
|---|-----------|-------|-----------|
| D1 | Hardware Physical Safety | 1–5 | [`01-physical-safety.md`](references/01-physical-safety.md) |
| D2 | Sensors & Attitude Algorithms | 6–9 | [`02-sensors-algorithms.md`](references/02-sensors-algorithms.md) |
| D3 | Logic Flow & Exception Reset | 10–13 | [`03-logic-exceptions.md`](references/03-logic-exceptions.md) |
| D4 | ⏳ RESERVED | 14–17 | — |
| D5 | Power & Electrical Integrity | 18–20 | [`04-power-integrity.md`](references/04-power-integrity.md) |
| D6 | Communication Bus Deadlock | 21–22 | [`05-bus-deadlock.md`](references/05-bus-deadlock.md) |
| D7 | Control Strategy & Physical Limits | 23–24 | [`06-control-strategy.md`](references/06-control-strategy.md) |
| D8 | Data Persistence & Debugging | 25–26 | [`07-data-debugging.md`](references/07-data-debugging.md) |
| D9 | Code Structural Traps | 27–28 | [`08-code-structure.md`](references/08-code-structure.md) |
| D10 | Kinematics & Motion Smoothing | 29–30 | [`09-kinematics.md`](references/09-kinematics.md) |
| D11 | Sensor Calibration & Drift | 31, 35 | [`10-calibration-drift.md`](references/10-calibration-drift.md) |
| D12 | Real-Time CPU & DMA Integrity | 32, 39 | [`11-cpu-dma.md`](references/11-cpu-dma.md) |
| D13 | Wireless Control & Mode Transition | 33–34 | [`12-wireless-modes.md`](references/12-wireless-modes.md) |
| D14 | State Machine & Input Robustness | 36–37 | [`13-state-machine.md`](references/13-state-machine.md) |
| D15 | System Startup & Debug Integrity | 38, 40–42 | [`14-system-startup.md`](references/14-system-startup.md) |
| D16 | Battery-Aware Control & Motion | 43–46 | [`15-battery-motion.md`](references/15-battery-motion.md) |
| D17 | Multi-Sensor Fusion & Competition | 47–49 | [`16-fusion-competition.md`](references/16-fusion-competition.md) |
| D18 | Competition Field Survival | 50–54 | [`17-field-survival.md`](references/17-field-survival.md) |
| D19 | Defense-in-Depth Hardware Guards | 55–57 | [`18-defense-depth.md`](references/18-defense-depth.md) |
| D20 | Emergency Stop & Post-Mortem | 58–60 | [`19-emergency-postmortem.md`](references/19-emergency-postmortem.md) |

---

## Templates & Tools

| Resource | File |
|----------|------|
| Report output template | [`references/output-template.md`](references/output-template.md) |
| 60-item quick reference | [`references/quick-reference.md`](references/quick-reference.md) |
| AI prompt prefix (copy-paste before generating code) | [`references/prompt-prefix.md`](references/prompt-prefix.md) |

---

## Conditional Item Tags

Items marked with these tags are checked only when the corresponding hardware is present:

| Tag | Condition |
|-----|-----------|
| `[SERVO]` | System includes servo motors |
| `[BALANCE]` | Two-wheel self-balancing robot |
| `[MECANUM]` | Mecanum / omni / swerve drive |
| `[VISION/LINE]` | Camera or grayscale line tracking |
| `[WIRELESS]` | Bluetooth / WiFi / NRF24 remote control |
| `[CAMERA+IMU]` | Fusing camera with IMU data |
| `[COMPETITION]` | Competition with start/finish lines |
| `[ENCODER]` | Wheel encoders present |

Items without tags apply to all projects.

---

## Output Format

Use the template in [`references/output-template.md`](references/output-template.md). The report must include:

1. **Header** — project name, MCU model, date
2. **Overall Severity** — CRITICAL / HIGH / MEDIUM / LOW
3. **Dimension tables** — one row per item: check name, result (✅/❌/⚠️/⏭️), finding
4. **CRITICAL FIXES REQUIRED** — numbered list with file:line, exact fix, physical consequence
5. **NEEDS MANUAL VERIFICATION** — measurement procedure, required tool, acceptable range
6. **CORRECTIVE CODE SNIPPETS** — complete replacement code for each FAIL item

---

## Quick Start

**To review a project:**

```
用 embedded-hardware-safety-review 审查这段代码
```

**To prevent issues before AI generates code**, copy the entire contents of [`references/prompt-prefix.md`](references/prompt-prefix.md) into your AI prompt as a system prefix.

**To quickly audit your own code**, use the checklist in [`references/quick-reference.md`](references/quick-reference.md).
