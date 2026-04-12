# Module Hierarchy Tracker

> 记录每个symbol的内部结构，随讲解逐步填充。
> 状态标记：✅ 已理解 | 🔲 待介绍 | ❓ 存疑

---

## Top Level: `data_inputV2` (I96) ✅

**接口：**
- IN: CLK, DATA, d_EN
- OUT: Channel_number<7:0>, Switch_number<7:0>, Stim_mode, IDAC<4:0>

**内部子模块：**

```
data_inputV2
├── SPI_V2                 ✅  SPI移位寄存器，输出解码后信号
├── DEL1BWP7T_Z (I94)     ✅  d_EN延迟cell → EN_SPI（channel decoder用），延迟≈1ns
├── DEL1BWP7T_Z (I95)     ✅  d_EN延迟cell → EN_SPI（switch decoder用），延迟≈1ns
├── 3bit_decoder_EN (I80) ✅  CH_address<2:0> + EN_SPI → Channel_number<7:0>
└── 3bit_decoder_EN (I85) ✅  SW_address<2:0> + EN_SPI → Switch_number<7:0>
```

**信号流：**
1. CLK + DATA + d_EN → **SPI_V2** → CH_address<2:0>, SW_address<2:0>, Stim_mode, IDAC<4:0>
2. d_EN → **DEL1BWP7T_Z** → EN_SPI（延迟后使能decoder，确保地址稳定）
3. CH_address + EN_SPI → **3bit_decoder_EN(I80)** → Channel_number<7:0>（one-hot）
4. SW_address + EN_SPI → **3bit_decoder_EN(I85)** → Switch_number<7:0>（one-hot）

---

## `SPI_V2` ✅
> SPI-like 12-bit shift register with gated CLK and gated output

**内部结构：**
- 12× DFD0BWP7T (D flip-flop)：DATA串行移入，形成12位移位寄存器
- CLK gating (每级)：INVD0BWP7IZN(d_EN) → AN2D0BWP7T(CLK, /d_EN) → DFF.CP
- Output gating (每bit)：AN2D0BWP7T(Q, d_EN) → BUFF → 输出

**标准单元：**
- `INVD0BWP7IZN` (I42): INV，对d_EN取反
- `AN2D0BWP7T` (I41): AND，CLK × /d_EN → DFF.CP
- `DFD0BWP7T` (I1): D flip-flop
- `AN2D0BWP7T` (I15): AND，Q × d_EN → output buffer

**工作逻辑：**

| d_EN | CLK到DFF | 输出AND门 | 状态 |
|------|---------|---------|------|
| 0 | 通过（移位有效） | 屏蔽（输出=0） | READ：逐位移入DATA |
| 1 | 封锁（DFF冻结） | 打开（输出=Q） | LOCK：输出当前12位值 |

**d_EN上升沿同时完成：**
1. 封锁CLK → 停止移位寄存器
2. 打开输出门 → 当前12位值送至decoder

---

## `DEL1BWP7T_Z` ✅
> 标准单元延迟buffer，延迟≈1ns

**用途：** d_EN上升沿时，SPI_V2的输出门立即打开，12位数据需要经过BUFF传播至decoder输入。DEL1BWP7T_Z将EN_SPI延迟~1ns，确保decoder的地址输入稳定后再使能，避免误译码。

**时序关系：**
```
d_EN ↑ ──→ SPI_V2 output gate opens ──→ address bits propagate via BUFF ──┐
d_EN ↑ ──→ DEL1BWP7T_Z (~1ns) ──→ EN_SPI ──────────────────────────────→ 3bit_decoder_EN
                                                                  地址已稳定 ↑
```

---

## `3bit_decoder_EN` ✅
> 3-to-8 decoder with enable，4-input AND实现

**标准单元：**
- `ZNNVD0BWP7T` × 3：每个地址位生成true + complement两路
- `AN4D0BWP7T` × 8：4-input AND gate，一个per输出位

**逻辑（EN_SPI为4th输入）：**
```
CH<0> = EN_SPI & /A2 & /A1 & /A0  (addr 000)
CH<1> = EN_SPI & /A2 & /A1 &  A0  (addr 001)
CH<2> = EN_SPI & /A2 &  A1 & /A0  (addr 010)
CH<3> = EN_SPI & /A2 &  A1 &  A0  (addr 011)
CH<4> = EN_SPI &  A2 & /A1 & /A0  (addr 100)
CH<5> = EN_SPI &  A2 & /A1 &  A0  (addr 101)
CH<6> = EN_SPI &  A2 &  A1 & /A0  (addr 110)
CH<7> = EN_SPI &  A2 &  A1 &  A0  (addr 111)
```
EN_SPI=0时所有输出为0（decoder disabled）

**输出级：**
- `pre_ch<7:0>` → `BUFFD3BWP7T`（drive strength 3）→ `CH<7:0>`
- 原因：one-hot信号需扇出至8个channel的local register，需要更强的驱动能力

---

## 其他模块（待引入）

| Module | 所属层级 | 状态 |
|--------|---------|------|
| Local register (per channel×switch) | channel level | ✅ 见下方两个assigner |

---

## `Switch_assigner_V3` (I119) ✅
> 每个channel的Switch + Stim_mode local寄存器，机制与IDAC_assigner_V3相同

**接口：**
| Pin | 方向 | 描述 |
|-----|------|------|
| SW_number<7:0> | IN | 全局switch one-hot数据（来自decoder） |
| Stim_mode | IN | 刺激方向（来自SPI_V2） |
| CH_NUMBER | IN | 本channel使能位 |
| D_EN | IN | 锁存触发 |
| EN_SWP<7:0> | OUT | 输出至本channel的switch matrix（电极选择） |
| STIM_MODE | OUT | 输出至H-bridge（正向/反向刺激，模拟部分详述） |

**内部结构（已确认，与IDAC_assigner_V3相同）：**
```
D_EN → DEL4BWP7T(I19) → delayed
DEL + D_EN → A2XOR2D0BWP7T(I16) → 双边沿脉冲
脉冲 + D_EN → AN2D0BWP7T(I17) → 上升沿脉冲
脉冲 + CH_NUMBER → AN2D0BWP7T(I20) → gated pulse → DFF.CP
STIM_MODE_PRE → DFOFD0BWP7T(I12) → STIM_MODE        （1个DFF）
EN_SW_PRE<7:0> → CPDFD0BWP7TN(I10<7:0>) → EN_SWP<7:0>  （8个DFF）
```
- 写使能：D_EN上升沿脉冲 & CH_NUMBER=1

---

## `IDAC_assigner_V3` (I118) ✅
> 每个channel的IDAC local寄存器，CH_number作为写使能

**接口：**
| Pin | 方向 | 描述 |
|-----|------|------|
| IDAC<4:0> | IN | 全局IDAC数据（来自SPI_V2，所有channel共享） |
| CH_number | IN | 本channel的使能位（来自CH<N>，one-hot中的1位） |
| D_EN | IN | 锁存触发（全局d_EN） |
| IDAC_PULSE<4:0> | OUT | 输出至本channel的IDAC模拟电路 |

**内部结构（脉冲发生器 + 门控DFF）：**
```
D_EN ──→ DEL4BWP7T(I19) ──→ D_EN_delayed
D_EN + D_EN_delayed ──→ A2XOR2D0BWP7T(I23) ──→ 双边沿窄脉冲
双边沿脉冲 + D_EN ──→ AN2D0BWP7T(I24) ──→ 仅上升沿脉冲 (assign_pulse)
assign_pulse + CHANNEL_NUMBER ──→ AN2D0BWP7T(I20) ──→ gated_pulse → DFF.CP
IDAC<4:0> ──────────────────────────────────────────────────────→ DFF.D (直接连接)
```

**工作逻辑：**
- D_EN上升沿 & CH_number=1 → gated_pulse触发 → 5个DFF锁存IDAC<4:0>
- D_EN上升沿 & CH_number=0 → 脉冲被屏蔽 → DFF保持不变
- 使用脉冲发生器而非直接用D_EN作CLK：避免D_EN=HIGH期间DFF透明导致glitch

**寻址机制：**
- 8个channel各有一个IDAC_assigner_V3实例
- 所有实例共享同一组IDAC<4:0>和D_EN输入
- 每个实例的CH_number接CH<N>（N=0~7），one-hot保证每次只有一个channel被写入

---

## 待介绍模块

| Module | 所属层级 | 状态 |
|--------|---------|------|
| TDM control logic | top level | 🔲 |
| Idle Controller | channel level | 🔲 |
| PI controller | channel level | 🔲 |
| Active feedback IDAC | channel level | 🔲 |
| CMLS Level Shifter | channel level | 🔲 |
| Complementary NMOS rectifier | channel level | 🔲 |
| VCDL | channel level | 🔲 |

---
