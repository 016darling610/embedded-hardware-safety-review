<p align="center">
  <img src="https://img.shields.io/badge/Claude%20Code-Skill-8A2BE2?style=for-the-badge&logo=anthropic&logoColor=white" alt="Claude Code Skill">
  <img src="https://img.shields.io/badge/Items-60-red?style=for-the-badge" alt="60 Items">
  <img src="https://img.shields.io/badge/Dimensions-20-blue?style=for-the-badge" alt="20 Dimensions">
  <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="MIT License">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-STM32-03234B?style=flat-square&logo=stmicroelectronics&logoColor=white">
  <img src="https://img.shields.io/badge/Arch-Cortex%20M3%20%7C%20M4%20%7C%20M7-0091BD?style=flat-square&logo=arm&logoColor=white">
  <img src="https://img.shields.io/badge/Motor-TB6612%20%7C%20DRV8833%20%7C%20L298N-FF6B00?style=flat-square">
  <img src="https://img.shields.io/badge/IMU-MPU6050%20%7C%20ICM--20948-00BFFF?style=flat-square">
  <img src="https://img.shields.io/badge/Language-C%20%7C%20C%2B%2B-555555?style=flat-square&logo=c&logoColor=white">
</p>

<p align="center">
  <img src="https://img.shields.io/github/stars/016darling610/embedded-hardware-safety-review?style=social" alt="Stars">
  <img src="https://img.shields.io/github/forks/016darling610/embedded-hardware-safety-review?style=social" alt="Forks">
</p>

---

```
╔══════════════════════════════════════════════════════════════╗
║  EMBEDDED HARDWARE SAFETY REVIEW — AI 代码烧板子审查器        ║
║  60 项检查 · 20 个维度 · 5000+ 行结构化规则                    ║
║  实战驱动：每一个规则背后都是一块烧毁的板子或一次比赛失利          ║
╚══════════════════════════════════════════════════════════════╝
```

> **AI 写的代码语法正确，但物理上危险。** LLM 不懂电流、电压、热量——它把 GPIO、PWM、PID 当作抽象文本来处理。这个 Skill 在 AI 生成代码之后、**上电之前**，强制执行 60 项物理安全检查。

---

## Why This Exists

| AI 生成的代码 | 物理后果 | 对应 Item |
|-------------|---------|-----------|
| `PWM_Frequency = 50` | 电机啸叫 10 分钟 → MOS 管冒烟烧毁 | [#1](references/01-physical-safety.md) |
| `GPIO_Mode_Out_PP` on encoder | IO 口短路 → 主控锁死 | [#3](references/01-physical-safety.md) |
| PID 输出无 `constrain()` | 堵转电流 → 驱动芯片炸裂 | [#2](references/01-physical-safety.md) |
| 无 IWDG 看门狗 | I2C 卡死 → 小车全速冲进人群 | [#10](references/03-logic-exceptions.md) |
| `asin(x)` 无值域检查 | NaN → 100% 占空比 → 撞墙 | [#13](references/03-logic-exceptions.md) |
| 无电池电压 ADC | 低压 → MCU 复位 → GPIO 乱跳 → 倒车撞裁判 | [#5](references/01-physical-safety.md) |
| AI 删除 `HAL_Delay(100)` | 外设未就绪 → 全场比赛随机数据 | [#41](references/14-system-startup.md) |

---

## Architecture

```
embedded-hardware-safety-review/
│
├── SKILL.md                 ← 137 行导航中枢 (Claude 首先读取)
├── README.md                ← 你正在看的
├── LICENSE                  ← MIT
│
└── references/              ← 22 个模块 (按需加载)
    │
    ├── 01-physical-safety.md         D1:  PWM 频率 · 占空比 · 引脚 · 死区 · 电池
    ├── 02-sensors-algorithms.md      D2:  Yaw · PID · 时间戳 · 编码器
    ├── 03-logic-exceptions.md        D3:  看门狗 · 上电 · 中断 · NaN 
    ├── 04-power-integrity.md         D5:  反电动势 · 共地干扰 · 舵机
    ├── 05-bus-deadlock.md            D6:  I2C 死锁 · 串口溢出
    ├── 06-control-strategy.md        D7:  平衡车 · 速度耦合
    ├── 07-data-debugging.md          D8:  Flash 存储 · 日志
    ├── 08-code-structure.md          D9:  volatile · 类型转换
    ├── 09-kinematics.md              D10: 麦轮 · 软启动
    ├── 10-calibration-drift.md       D11: 陀螺漂移 · 传感器置信度
    ├── 11-cpu-dma.md                 D12: ISR 超时 · DMA 一致性
    ├── 12-wireless-modes.md          D13: 心跳超时 · PID 无冲击切换
    ├── 13-state-machine.md           D14: 状态超时 · 按键消抖
    ├── 14-system-startup.md          D15: 栈溢出 · 晶振 · 延时 · SWD
    ├── 15-battery-motion.md          D16: 电压前馈 · 俯仰解耦 · 打滑 · 底盘
    ├── 16-fusion-competition.md      D17: 时间对齐 · 起跑冲线 · 摩擦
    ├── 17-field-survival.md          D18: 电压跌落 · OTSU · 干扰 · Flash · 重心
    ├── 18-defense-depth.md           D19: RCC 复位 · 时钟验算 · 寄存器守卫
    ├── 19-emergency-postmortem.md    D20: 急停 · 黑匣子 · 双配置
    ├── output-template.md            审查报告模板
    ├── quick-reference.md            60 项快速检查表
    └── prompt-prefix.md              AI 生成代码前置约束
```

> **渐进式加载**: Claude 先读 `SKILL.md`(137 行) 了解全貌，再按被审查项目的子系统按需加载对应的 `references/` 模块。不需要加载全部 5000 行。

---

## Quick Start

### Installation

```bash
# Clone
git clone https://github.com/016darling610/embedded-hardware-safety-review.git

# Install to Claude Code
cp -r embedded-hardware-safety-review ~/.claude/skills/

# Restart Claude Code — Skill auto-registers
```

### Usage

```
用 embedded-hardware-safety-review 审查这段代码
```

Or let it trigger automatically — any mention of `STM32安全`, `PWM安全`, `电机驱动审查`, `姿态解算审查`, etc.

### Prevention (AI Prompt Prefix)

Before asking AI to generate embedded code, paste the contents of [`references/prompt-prefix.md`](references/prompt-prefix.md) as a system prefix. 46 constraints that prevent most issues before they're written.

---

## Dimension Overview

| # | Dimension | Items | Priority |
|---|-----------|-------|----------|
| D1 | Physical Safety | 1–5 | 🔴 Critical |
| D2 | Sensors & Algorithms | 6–9 | 🔴 Critical |
| D3 | Logic & Exceptions | 10–13 | 🔴 Critical |
| D5 | Power Integrity | 18–20 | 🟡 High |
| D6 | Bus Deadlock | 21–22 | 🟡 High |
| D7 | Control Limits | 23–24 | 🟡 High |
| D8 | Data & Debugging | 25–26 | 🟢 Medium |
| D9 | Code Structure | 27–28 | 🟢 Medium |
| D10 | Kinematics | 29–30 | 🟡 High |
| D11 | Calibration & Drift | 31, 35 | 🟡 High |
| D12 | CPU & DMA | 32, 39 | 🟡 High |
| D13 | Wireless & Modes | 33–34 | 🟡 High |
| D14 | State Machine | 36–37 | 🟡 High |
| D15 | System Startup | 38, 40–42 | 🔴 Critical |
| D16 | Battery & Motion | 43–46 | 🟡 High |
| D17 | Fusion & Competition | 47–49 | 🟢 Medium |
| D18 | Field Survival | 50–54 | 🟢 Medium |
| D19 | Defense-in-Depth | 55–57 | 🟡 High |
| D20 | Emergency & Post-Mortem | 58–60 | 🔴 Critical |

---

## Conditional Items

Not every project needs all 60 checks. Items tagged with these labels auto-skip when the hardware isn't present:

| Tag | Trigger |
|-----|---------|
| `[SERVO]` | Servo motors present |
| `[BALANCE]` | Two-wheel balancing robot |
| `[MECANUM]` | Mecanum / omni / swerve wheels |
| `[VISION/LINE]` | Camera or line tracking sensors |
| `[WIRELESS]` | Bluetooth / WiFi / NRF24 remote |
| `[CAMERA+IMU]` | Camera + IMU sensor fusion |
| `[COMPETITION]` | Race with start/finish lines |
| `[ENCODER]` | Wheel encoders present |

---

## Real-World Validation

Tested on **K题自动避障小车** (STM32F103C8T6 + TB6612FNG + MPU6050):

| | Before Review | After Fixes |
|---|:---:|:---:|
| CRITICAL issues | 3 | **0** |
| FAIL items | 8 | 16 (deeper issues surfaced) |
| PASS items | 6 | **18** |
| Key fixes | — | IWDG · Battery ADC · 15kHz PWM · 85% Limit · NaN Guards · Integral Separation · Soft Start |

---

## Related Competition Applicability

| Competition Type | Fit | Notes |
|:---|:---:|:---|
| 🚗 Wheeled robots (4WD/Mecanum/Balance/Ackermann) | ⭐⭐⭐⭐⭐ | Native — motor control, attitude, PID fully covered |
| 🚁 Drones / Inverted Pendulum / Wind Pendulum | ⭐⭐⭐⭐⭐ | Attitude, PID, IMU rules fully reusable |
| 🔌 Power Electronics (Inverter/DC-DC/Charger) | ⭐⭐⭐⭐ | Dead-time, soft-start, limits, bus timeout rules transfer directly |
| 📊 Instrumentation (Scope/Signal Gen/Analyzer) | ⭐⭐⭐⭐ | Stack overflow, DMA double-buffer, crystal config, ADC rules universal |
| 📡 RF/Communications (Amplifier/Transceiver) | ⭐⭐ | Button debounce, I2C/SPI timeout rules apply; rest is analog-dominant |

---

## Contributing

Every rule in this skill comes from real hardware destruction, competition failure, or late-night debugging sessions. If you have your own war stories:

1. Fork the repo
2. Add your item following the template:
   - **AI Blind Spot** — Why AI gets this wrong
   - **Real-World Consequence** — What actually happened
   - **❌ Wrong Code** — What AI writes
   - **✅ Correct Code** — The fix
   - **📋 Verification Rules** — How to audit
   - **🔧 Corrective Action** — What to do if it fails
3. Submit a PR

---

## License

MIT © [016darling610](https://github.com/016darling610)

---

<p align="center">
  <sub>Built with pain. Tested with fire. Deployed with confidence.</sub>
</p>
