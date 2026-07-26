<p align="center">
  <img src="https://img.shields.io/badge/Claude%20Code-Skill-8A2BE2?style=for-the-badge&logo=anthropic&logoColor=white">
  <img src="https://img.shields.io/badge/Items-60-red?style=for-the-badge">
  <img src="https://img.shields.io/badge/Dimensions-20-blue?style=for-the-badge">
  <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-STM32-03234B?style=flat-square&logo=stmicroelectronics&logoColor=white">
  <img src="https://img.shields.io/badge/Arch-Cortex%20M3%20%7C%20M4%20%7C%20M7-0091BD?style=flat-square&logo=arm&logoColor=white">
  <img src="https://img.shields.io/badge/Motor-TB6612%20%7C%20DRV8833%20%7C%20L298N-FF6B00?style=flat-square">
  <img src="https://img.shields.io/badge/IMU-MPU6050%20%7C%20ICM--20948-00BFFF?style=flat-square">
  <img src="https://img.shields.io/badge/Language-C%20%7C%20C%2B%2B-555555?style=flat-square&logo=c&logoColor=white">
</p>

<p align="center">
  <img src="https://img.shields.io/github/stars/016darling610/embedded-hardware-safety-review?style=social">
  <img src="https://img.shields.io/github/forks/016darling610/embedded-hardware-safety-review?style=social">
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

| AI 生成的代码 | 物理后果 | Item |
|-------------|---------|------|
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
├── SKILL.md                 ← 137 行导航中枢 (Claude 加载入口)
├── README.md                ← 你正在看的
├── LICENSE                  ← MIT
│
└── references/              ← 22 个模块 (按需渐进式加载)
    │
    ├── 01-physical-safety.md         D1:  PWM · 占空比 · 引脚 · 死区 · 电池
    ├── 02-sensors-algorithms.md      D2:  Yaw · PID · 时间戳 · 编码器
    ├── 03-logic-exceptions.md        D3:  看门狗 · 上电 · 中断 · NaN
    ├── 04-power-integrity.md         D5:  反电动势 · 共地干扰 · 舵机
    ├── 05-bus-deadlock.md            D6:  I2C死锁 · 串口溢出
    ├── 06-control-strategy.md        D7:  平衡车 · 速度耦合
    ├── 07-data-debugging.md          D8:  Flash存储 · 日志
    ├── 08-code-structure.md          D9:  volatile · 类型转换
    ├── 09-kinematics.md              D10: 麦轮 · 软启动
    ├── 10-calibration-drift.md       D11: 陀螺漂移 · 置信度
    ├── 11-cpu-dma.md                 D12: ISR超时 · DMA一致性
    ├── 12-wireless-modes.md          D13: 心跳 · 无冲击切换
    ├── 13-state-machine.md           D14: 状态超时 · 按键消抖
    ├── 14-system-startup.md          D15: 栈溢出 · 晶振 · 延时 · SWD
    ├── 15-battery-motion.md          D16: 电压前馈 · 俯仰 · 打滑 · 底盘
    ├── 16-fusion-competition.md      D17: 时间对齐 · 起跑冲线 · 摩擦
    ├── 17-field-survival.md          D18: 负载检测 · OTSU · 干扰 · Flash · 重心
    ├── 18-defense-depth.md           D19: RCC复位 · 时钟验算 · 寄存器守卫
    ├── 19-emergency-postmortem.md    D20: 急停 · 黑匣子 · 双配置
    ├── output-template.md            审查报告模板
    ├── quick-reference.md            60项快速检查表
    └── prompt-prefix.md              AI生成代码前置约束
```

---

## Quick Start

```bash
# 安装到 Claude Code
git clone https://github.com/016darling610/embedded-hardware-safety-review.git
cp -r embedded-hardware-safety-review ~/.claude/skills/
# 重启 Claude Code — Skill 自动注册
```

### 使用

```
用 embedded-hardware-safety-review 审查这段代码
```

触发词：`STM32安全` `PWM安全` `电机驱动审查` `姿态解算审查` `竞赛安全` `封车安全`

### 预防：AI Prompt 前缀

每次让 AI 生成嵌入式代码前，将 [`references/prompt-prefix.md`](references/prompt-prefix.md) 粘贴到 Prompt 开头。46 条约束从源头拦截。

---

## 维度总览

| # | 维度 | Items | 等级 |
|---|------|-------|:----:|
| D1 | Physical Safety | 1–5 | 🔴 |
| D2 | Sensors & Algorithms | 6–9 | 🔴 |
| D3 | Logic & Exceptions | 10–13 | 🔴 |
| D5 | Power Integrity | 18–20 | 🟡 |
| D6 | Bus Deadlock | 21–22 | 🟡 |
| D7 | Control Limits | 23–24 | 🟡 |
| D8 | Data & Debugging | 25–26 | 🟢 |
| D9 | Code Structure | 27–28 | 🟢 |
| D10 | Kinematics | 29–30 | 🟡 |
| D11 | Calibration & Drift | 31, 35 | 🟡 |
| D12 | CPU & DMA | 32, 39 | 🟡 |
| D13 | Wireless & Modes | 33–34 | 🟡 |
| D14 | State Machine | 36–37 | 🟡 |
| D15 | System Startup | 38, 40–42 | 🔴 |
| D16 | Battery & Motion | 43–46 | 🟡 |
| D17 | Fusion & Competition | 47–49 | 🟢 |
| D18 | Field Survival | 50–54 | 🟢 |
| D19 | Defense-in-Depth | 55–57 | 🟡 |
| D20 | Emergency & Post-Mortem | 58–60 | 🔴 |

---

## 条件项

| 标记 | 条件 |
|------|------|
| `[SERVO]` | 舵机 |
| `[BALANCE]` | 两轮平衡车 |
| `[MECANUM]` | 麦轮/全向轮 |
| `[VISION/LINE]` | 摄像头/灰度循迹 |
| `[WIRELESS]` | 蓝牙/WiFi/NRF24 |
| `[CAMERA+IMU]` | 摄像头+IMU 融合 |
| `[COMPETITION]` | 竞赛起跑/冲线 |
| `[ENCODER]` | 轮式编码器 |

---

## 实测验证

**K题自动避障小车** (STM32F103C8T6 + TB6612FNG + MPU6050):

| | 审查前 | 修复后 |
|---|:---:|:---:|
| CRITICAL | 3 | **0** |
| FAIL | 8 | 16 |
| PASS | 6 | **18** |
| 关键修复 | — | IWDG · 电池ADC · 15kHz · 85%限幅 · NaN防线 · 积分分离 · 软启动 |

## 适用赛题

| 赛题 | 适用度 | 说明 |
|------|:---:|------|
| 🚗 小车类 | ⭐⭐⭐⭐⭐ | 运动控制/电机/姿态全覆盖 |
| 🚁 无人机/倒立摆/风力摆 | ⭐⭐⭐⭐⭐ | 姿态/PID/IMU 完全复用 |
| 🔌 电源类 | ⭐⭐⭐⭐ | 死区/软启动/限幅/超时规则平移 |
| 📊 仪器仪表类 | ⭐⭐⭐⭐ | 栈/DMA/晶振/ADC 规则通用 |
| 📡 高频/通信类 | ⭐⭐ | 消抖/I2C/SPI 超时可用 |

---

## 贡献

每个规则来自真实的硬件烧毁或比赛失利。贡献新规则：

1. Fork → 2. 按模板添加 → 3. PR

模板：AI盲区 → 物理后果 → ❌错误代码 → ✅正确代码 → 📋验证规则 → 🔧修复动作

---

MIT © [016darling610](https://github.com/016darling610)

<p align="center"><sub>Built with pain. Tested with fire. Deployed with confidence.</sub></p>
