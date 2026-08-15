# 24G 私有协议知识库

## 概述

本 Skill 包含对 `mouse_public_Single_Mode` 项目中 24G 私有无线协议的完整理解，包括对码（配对）、重连、正常通讯的全部流程和数据包结构。24G 协议使用自定义 FSK 收发器，不是标准 BLE。

---

## 一、文件结构

### 核心协议文件

| 文件 | 内容 |
|------|------|
| `program/g24_protocol_stack/24g_pair.prog` (425行) | 对码状态机、四步握手全部实现（发射端+接收端） |
| `program/g24_protocol_stack/24g.prog` (567行) | 底层 RF 收发、PDU 组装、ACK 机制、地址/同步字/HEC 计算 |
| `program/g24_protocol_stack/24g_reconn.prog` (178行) | 重连全部逻辑 |
| `program/g24_protocol_stack/24g_transmitter.prog` (400行) | 发射调度、ACK 等待、重传跳频 |
| `program/g24_protocol_stack/24g_receiver.prog` (572行) | 接收调度、模式切换（BIND→SEARCH→WORK）、多地址轮询 |
| `program/app_mouse.prog` | 鼠标端应用层：触发对码、重连、数据打包 |
| `program/app_dongle.prog` | Dongle 端应用层：主循环、回调注册、ACK payload 准备 |

### 数据格式定义文件

| 文件 | 内容 |
|------|------|
| `format/g24_protocol_stack/24g_pair.format` | 对码状态常量、数据类型常量、对码内存分配 |
| `format/g24_protocol_stack/24g.format` | 基础 RF 数据结构（地址、通道映射） |
| `format/g24_protocol_stack/24g_transmitter.format` | 连接状态机、重试计数 |
| `format/g24_protocol_stack/24g_receiver.format` | 工作模式、绑定状态、ACK payload |
| `format/g24_protocol_stack/24g_reconn.format` | 重连数据结构 |

---

## 二、地址体系

### 地址列表

| 地址变量 | 持有端 | 大小 | 说明 |
|----------|--------|------|------|
| `mem_24g_device_addr` | 发射端 | 4B | 鼠标/键盘的唯一地址（来自 LAP 前 4 字节） |
| `mem_24g_lap` | Dongle | 6B | Dongle 的唯一标识，前 4 字节用于 RF Access Address |
| `mem_24g_receiver_addr` | 发射端 | 4B | 对码 Step2 从 Dongle ACK 学到的 Dongle 地址（LAP 前 4B） |
| `mem_24g_transmitter_addr` | Dongle | 4B | 对码 Step1 从 ATTEMP 包保存的发射端地址 |
| `mem_24g_device1_addr` | Dongle | 4B | 鼠标地址（transmitter_addr 的分类存档） |
| `mem_24g_device2_addr` | Dongle | 4B | 键盘地址（transmitter_addr 的分类存档） |
| `mem_24g_pair_addr` | 双方 | 4B | 对码广播地址 = 0x101520（公共频道） |
| `mem_24g_addr` | 双方 | 4B | RF 硬件当前使用的 Access Address（运行时可变） |
| `mem_24g_fast_conn_addr` | 发射端 | 4B | 快速连接备用通道地址 |

### 地址的角色本质

真正的"身份"只有两个：**发射端的 device_addr** 和 **Dongle 的 LAP**。其余地址都是这两个身份的副本或临时值。`mem_24g_addr` 是操作寄存器，不同场景下被不同来源覆盖。

### LAP（Lower Address Part）

Dongle 的出厂唯一标识，6 字节。默认值示例（`sched/mouse.dat` 行 184）：`01 12 26 55 51 61`。只有前 4 字节用于 RF 的 Access Address 对比。对码 Step1 中发射端发送 4 字节地址，Dongle 用 LAP 前 4 字节与其对比。

---

## 三、RF 帧结构

### 通用空中 RF 帧

所有 24G 数据包（对码、ACK、正常通讯、重连）共享相同结构：

```
┌──────────┬──────────────┬─────┬──────┬─────────┬──────────┬─────┐
│ Preamble │ Access Addr  │ HEC │ Type │ Control │ Payload  │ CRC │
│  5 字节   │    4 字节    │ 1B  │  1B  │   1B    │ N 字节   │ 3B  │
└──────────┴──────────────┴─────┴──────┴─────────┴──────────┴─────┘
                                        └──────── PDU ────────┘
```

Preamble = 5 字节（40 bits）。因为使用自定义 FSK 收发器而非标准 BLE 硬件，需要更长前导码做比特同步。

### PDU（`g24_transmit_prep`，24g.prog 行 282-322）

```
PDU偏移     │ 来源                       │ 说明
────────────┼────────────────────────────┼──────
txpayload[0]│ mem_24g_syncword_crc8      │ HEC = 同步字低字节 + 高字节
txpayload[1]│ mem_24g_data_type 低3位     │ Type（设备/数据类型标识）
txpayload[2]│ datalen<<3|PID<<1|NO_ACK    │ Control字节
txpayload[3]│ mem_24g_txbuf[0..N-1]       │ 应用层 payload
```

### Control 字节结构

```
bit 7:3 = payload_len  (5 bits，payload 字节数)
bit 2:1 = PID          (2 bits，包序号 0-3，用于去重)
bit 0   = NO_ACK       (1 bit，=1 不要求 ACK)
```

各种场景的 Control 值：

| 场景 | datalen | Control |
|------|---------|---------|
| 对码包 | 7 | 0x38 |
| 对码 ACK | 8 | 0x40 |
| 重连 ATTEMP | 6 | 0x30 |
| 正常鼠标包 | 8 | 0x40 |
| 正常键盘包 | 10 | 0x50 |

### HEC（Header Error Check）

`g24_update_addr_and_synccrc8`（24g.prog 行 392-410）：4 字节地址之和 → syncword（2B），syncword 低字节 + 高字节 → crc8（1B），即 HEC。用于接收方验证 Access Address 是否正确收到。

### 帧组装完整调用链

```
mouse_24g_package_data（填充 txbuf）
  → g24_transmit_prep（组装 PDU：HEC+Type+Control+payload）
    → g24_transmit（加 Preamble+Access Addr，RF 发送）
```

---

## 四、对码（配对）流程

### 状态机

**对码状态机** `mem_24g_pair_sm`（定义于 24g_pair.format 行 28-38）：

```
NULL(0x00) → STATE_1(0x01, ATTEMP) → STATE_1_WAITING_ACK(0x11)
           → STATE_2(0x02, BIND)    → STATE_2_WAITING_ACK(0x12)
           → STATE_3(0x03, CONFIG)   → STATE_3_WAITING_ACK(0x13)
           → STATE_4(0x04, OK)       → STATE_4_WAITING_ACK(0x14)
           → SUCCESS(0xFF)
```

**连接状态机** `mem_24g_conn_sm`（定义于 24g_transmitter.format 行 40-45）：

```
bit0 = STATE_24G_PAIR（对码模式）
bit1 = STATE_24G_RECONN（重连模式）
均为 0 = 停止/正常工作
```

### 触发入口

**发射端**：`mouse_24g_start_pair_mode`（app_mouse.prog 行 4448）→ `g24_pair_start`（24g_pair.prog 行 21）

**接收端**：`dongle_pc_bind`（app_dongle.prog 行 146）→ `g24_bind_mode_auto`（24g_receiver.prog 行 294）

### 对码包数据结构

每个对码包 datalen=7，结构如下：

```
txbuf[0] = 对码数据类型（0xFF/0xAA/0x55/0x22）
txbuf[1] = 设备类型（TYPE_MS=1 / TYPE_KB=2，取低 3 位 bits_data）
txbuf[2-5] = 地址字段（根据步骤不同含义不同）
txbuf[6] = 0x00（填充）
```

### 四步握手

#### Step 1: ATTEMP（发射端 → Dongle）

- txbuf: `[0xFF, 设备类型, device_addr(4B), 0x00]`
- 含义：**"我是鼠标/键盘，这是我的地址"**
- Dongle：保存发射端地址到 `mem_24g_transmitter_addr` 和 `mem_24g_deviceX_addr`
- Dongle ACK：普通 ACK（无有效 payload，ackpayload_enable 此时为 0）

#### Step 2: BIND（发射端 → Dongle）

- txbuf: `[0xAA, 设备类型, device_addr(4B), 0x00]`
- 含义：**"请绑定我"**
- 若 `pair_switch=1`：发送前将 RF 地址从广播切到 device_addr
- Dongle ACK payload（8字节）：`[设备类型(0x01/0x02), 0x80, LAP(6B)]`
- 发射端从 ACK 学 Dongle 的 LAP → `mem_24g_receiver_addr`
- Dongle 含义：**"我是 Dongle，这是我的地址"**

#### Step 3: CONFIG（发射端 → Dongle）

- `pair_switch=1`：txbuf `[0x55, 设备类型, receiver_addr(4B), 0x00]` — 回传 Dongle 地址
- `pair_switch=0`：txbuf `[0x55, 设备类型, device_addr(4B), 0x00]` — 发自己地址
- 含义：**"我确认你的地址是 XXX"**
- Dongle 校验（pair_switch=1）：回传地址 == LAP？不匹配则拒绝
- Dongle ACK payload（pair_switch=1）：`[设备类型, 0x80, transmitter_addr(4B)]`
- 发射端校验：ACK 中地址 == device_addr？不匹配则拒绝
- 至此**双向确认完成**

#### Step 4: OK（发射端 → Dongle）

- txbuf: `[0x22, 设备类型, device_addr(4B), 0x00]`
- 含义：**"绑定完成"**
- 两端最终校验地址 → `STATE_24G_PAIRING_SUCCESS`
- 发射端通知 UI（`BT_EVT_24G_PAIRING_COMPLETE`），退出对码模式
- Dongle 标记 `bind_disable=1`（关闭槽位），进入 DONGLE_SEARCH 模式

### pair_switch 标志

`mem_24g_pair_switch` = 1（完整模式）：从 Step1 开始，走全部地址校验（6 处检查点全在 24g_pair.prog 中），Step2 起切换到设备专属地址。

`mem_24g_pair_switch` = 0（简化模式）：从 Step2 开始，跳过全部地址校验，四步握手全程在广播地址 0x101520 上进行。安全性更低、更快。

---

## 五、ACK 数据包

### ACK 触发机制

`g24_transmit_ack`（24g_receiver.prog 行 111-126）：收到包后检查 Control 字节 bit0（NO_ACK）。NO_ACK=1 则跳过；NO_ACK=0 则回复 ACK。

### ACK 帧结构

与发射端帧结构完全相同，区别：datalen=8, txlen=11, Control=0x40。

### ACK Payload（8字节）

| 步骤 | [0] | [1] | [2-5] | [6-7] |
|------|-----|-----|-------|-------|
| Step 1 | 残留值 | 残留值 | 残留值 | 残留值 |
| Step 2 | 0x01/0x02 | 0x80 | LAP[0-3] | LAP[4-5] |
| Step 3(pair=1) | 0x01/0x02 | 0x80 | transmitter_addr | 残留值 |
| Step 3(pair=0) | 0x01/0x02 | 0x80 | LAP[0-3] | LAP[4-5] |
| Step 4 | 0x01/0x02 | 0x80 | LAP[0-3] | LAP[4-5] |

### 发射端解析 ACK

`g24_ackpayload_parse`（24g_transmitter.prog 行 176-184）：从 rxbuf 偏移 2 开始读 8 字节到 `mem_24g_rxpayload`。`rxpayload+2` 起 4 字节即为地址字段。

---

## 六、对码完成后

### Dongle 模式切换

对码成功后 `g24_bind_mode_auto` 检测到 `bind_device_status` 非零 → `g24_switch_work_mode` 切到 DONGLE_SEARCH 模式 → 启动多地址轮询监听。

### SEARCH 模式地址轮询

`g24_auto_addr_ch_search`（24g_receiver.prog 行 396-407）：用 `mem_24g_time_slice` 低 2 位在 4 个槽位间轮流：

```
槽 0 → device2 地址
槽 1 → 自己的 LAP
槽 2 → device1 地址
槽 3 → 自己的 LAP
```

每个槽位调用 `g24_update_addr_and_synccrc8` 切换 RF 硬件地址。收到数据后切到 DONGLE_WORK 工作模式。

---

## 七、正常通讯帧

### 鼠标包

datalen=8, txlen=11, Control=0x40。

```
txbuf[0] = 0x00（累积标志）
txbuf[1] = mem_mouse_key（bit0=左键 bit1=右键 bit2=中键）
txbuf[2-3] = X 位移
txbuf[4-5] = Y 位移
txbuf[6] = 滚轮 Z（有符号）
txbuf[7] = 倾斜滚轮 TZ（有符号）
```

### 键盘包

datalen=10, txlen=13, Control=0x50。

```
txbuf[0] = 0x00
txbuf[1-9] = 键值（9字节，mem_customer_key_press/release）
```

### 与对码包关键区别

| | 对码包 | 正常通讯包 |
|--|--------|-----------|
| Access Address | 0x101520（广播） | 设备专属地址 |
| datalen | 7 | 8（鼠标）/10（键盘） |
| txbuf[0] | 对码类型 0xFF/AA/55/22 | 累积标志 0x00 |
| txbuf[1-5] | 设备类型+地址 | 按键+传感器数据 |
| txbuf[6] | 0x00（填充） | 滚轮数据 |

---

## 八、重连协议

### 概述

对码完成后发射端从睡眠唤醒或断线后恢复通信时，只发一个轻量版 ATTEMP 包宣告身份，不重复四步握手。

### 重连包结构

datalen=6, txlen=9, Control=0x30（比对码 ATTEMP 少 1 字节填充）。

```
txbuf[0] = 0xFF（DATATYPE_ATTEMP）
txbuf[1] = 设备类型
txbuf[2-5] = device_addr（自身地址）
（无填充字节）
```

### 重连流程

1. `g24_reconn_start`（24g_reconn.prog 行 10-32）：设置 STATE_24G_RECONN，根据 reconn_type 选地址
2. `g24_reconn_dispatch`（行 61-109）：速率控制 — 每 512 周期密集发 8 次，中间 504 周期静默
3. 发送 ATTEMP → 等待 ACK → 两级地址校验
4. 成功后通知 UI + 退出重连模式；失败则 `g24_reconn_device_fail` 地址轮换重试

### 重连地址选择（mem_24g_reconn_type）

| 值 | 常量 | 策略 |
|----|------|------|
| 0 | DEFAULT_24G_DEVICE | 只用 receiver_addr |
| 1 | FAST_CONN_AND_RECEIVER | fast_conn_addr ↔ receiver_addr |
| 2 | FAST_CONN_AND_3_0_ADDR | fast_conn_addr ↔ device_addr |
| 3 | RECEIVER_AND_3_0_ADDR | receiver_addr ↔ device_addr |
| 4 | PAIR_AND_3_0_ADDR | pair_addr ↔ device_addr |

### 重连时 Dongle 行为

Dongle 在工作模式收到 ATTEMP 包后：`g24_receive_packet_parse` → `g24_data_receive_attemp` → `g24_data_attemp`。重连包**不触发 ACK 回复**（NO_ACK=1），Dongle 只更新地址记录。

### 对码 ATTEMP vs 重连 ATTEMP

| | 对码 ATTEMP | 重连 ATTEMP |
|--|-----------|------------|
| txbuf 长度 | 7B | 6B |
| PDU(txlen) | 10B | 9B |
| Control | 0x38 | 0x30 |
| 填充字节 | 有 | 无 |
| Access Addr | 0x101520 | 设备专属地址 |
| Dongle 模式 | DONGLE_BIND | DONGLE_SEARCH/WORK |
| Dongle 响应 | 回 ACK | 不回 ACK，只更新地址 |

---

## 九、关键函数速查

### 对码（24g_pair.prog）

| 函数 | 行号 | 说明 |
|------|------|------|
| `g24_pair_start` | 21-39 | 对码启动入口 |
| `g24_pair_dispatch` | 43-58 | 状态机分发（每 256 次调度执行一次） |
| `g24_pair_sm_1/2/3/4` | 60-82 | 四步握手发送 |
| `g24_pair_sm_common` | 86-106 | 通用流程：填充→发送→等ACK→解析→分发 |
| `g24_pair_sm_N_waiting_ack` | 115-158 | 各步骤 ACK 处理 |
| `g24_pair_exit` | 154-158 | 退出对码模式 |
| `g24_bind_data_process` | 211-224 | Dongle 接收-解析-ACK 循环 |
| `g24_bind_data_parse` | 230-241 | 按 DATATYPE 分发 |
| `g24_bind_first/second/third_step` | 244-361 | Dongle 三步处理 |
| `g24_bind_ackpayload_prep` | 203-208 | ACK payload 装入 txbuf |

### 底层 RF（24g.prog）

| 函数 | 行号 | 说明 |
|------|------|------|
| `g24_transmit` | 179-274 | RF 发射 |
| `g24_transmit_prep` | 282-322 | PDU 组装 |
| `g24_transmit_prep_pdu` | 297-306 | Control 字节组装 |
| `g24_receive_packet` | 20-112 | RF 接收 |
| `g24_read_len_pid_crc` | 345-362 | 解析收到的 Control 字节 |
| `g24_update_addr_and_synccrc8` | 392-410 | 更新地址+计算同步字+HEC |

### 发射端（24g_transmitter.prog）

| 函数 | 行号 | 说明 |
|------|------|------|
| `g24_transmit_dispatch` | 29-32 | 按 conn_sm 分发（PAIR/RECONN） |
| `g24_transmit_receive_ack` | 142-156 | 发送后切 RX 等 ACK |
| `g24_transmit_process` | 113-140 | 发射主流程 |
| `g24_ackpayload_parse` | 176-184 | 解析 ACK payload |
| `g24_retransmit` | 158-174 | 重传机制 |

### 接收端（24g_receiver.prog）

| 函数 | 行号 | 说明 |
|------|------|------|
| `g24_transmit_ack` | 111-126 | ACK 回复（含 NO_ACK 判断） |
| `g24_ackpayload_prep` | 12-18 | ACK payload 准备（回调机制） |
| `g24_receive_packet_start` | 21-23 | 接收+ACK 总入口 |
| `g24_receive_packet_parse` | 24-33 | 解析收到数据到 rxdata_temp |
| `g24_bind_mode_auto` | 294-307 | 对码模式自动循环 |
| `g24_search_mode_auto` | 341-378 | 搜索模式自动循环（多地址轮询） |
| `g24_work_init` | 250-265 | 工作模式初始化 |

### 重连（24g_reconn.prog）

| 函数 | 行号 | 说明 |
|------|------|------|
| `g24_reconn_start` | 10-50 | 重连启动 |
| `g24_reconn_dispatch` | 61-109 | 重连调度 |
| `g24_reconn_data_prep` | 102-109 | 组装重连 ATTEMP 包 |
| `g24_reconn_device_fail` | 111-141 | 失败跳频/地址轮换 |
| `g24_data_attemp` | 151-171 | ATTEMP 包处理（对码/重连共用） |

---

## 十、关键概念

### bid_disable 标志

`mem_24g_deviceX_bind_disable`：运行时槽位锁。进入对码模式时归零（g24_bind_init 行 196-197），对码成功后置 1（g24_bind_dvcX_step_success），阻止同类型设备覆盖地址。存储在 XRAM 中，掉电丢失。

### MODE 切换

Dongle 有三种工作模式（mem_24g_work_mode）：DONGLE_BIND（对码）、DONGLE_SEARCH（搜索，多地址轮询）、DONGLE_WORK（正常工作，锁定当前设备）。

### 初始化链

`dongle_init` → `usb_init`（尾调用优化，不是返回后调用，而是直接跳转不复返）。`dongle_init` 只注册回调，`usb_init` 做 USB 硬件初始化和协议变量设置。

---

## 参考文档

项目中有 4 个分拆文档：
- `24G对码-01-概述与数据结构.md`
- `24G对码-02-对码流程与数据包.md`
- `24G对码-03-重连协议机制.md`
- `24G对码-04-RF层与调用链.md`
- `24G正常通讯协议包结构.md`
