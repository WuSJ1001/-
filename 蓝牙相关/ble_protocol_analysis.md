# BLE 协议栈变量分析笔记

本文档记录了 BLE 协议栈核心变量、广播数据结构及功能分析。

---

## 目录

1. [内存布局概览](#1-内存布局概览)
2. [BLE 协议栈核心变量](#2-ble-协议栈核心变量)
3. [蓝牙广播数据变量](#3-蓝牙广播数据变量)
4. [广播数据格式解析示例](#4-广播数据格式解析示例)
5. [关键函数分析](#5-关键函数分析)

---

## 1. 内存布局概览

BLE 协议栈核心变量位于 RAM 起始位置（0x0000 开始）：

| 地址   | 变量名                  |
|--------|-------------------------|
| 0x0000 | mem_le_adv_transmit     |
| 0x0001 | mem_le_adv_waitcnt      |
| 0x0002 | mem_le_adv_rcv          |
| 0x0003 | mem_le_req_rcv          |
| 0x0004 | mem_le_scanrsp_rcv      |
| 0x0005 | mem_le_conn_rcv         |
| 0x0006 | mem_current_context     |
| 0x0007 | mem_le_ch_mapped        |
| 0x0008 | mem_last_freq           |
| 0x0009 | mem_rssi                |
| 0x000a | mem_context_ptr         |
| 0x000c | mem_rx_window           |
| 0x000e | mem_lpm_adjust          |
| 0x000f | mem_sync_clke           |

---

## 2. BLE 协议栈核心变量

### 2.1 广播/扫描统计变量

| 变量名 | 作用 |
|--------|------|
| `mem_le_adv_transmit` | **广告发送计数器** - 记录已发送的广告包数量，每次发送 ADV/SCAN_REQ 后递增 |
| `mem_le_adv_waitcnt` | **广告等待计数器** - 记录等待广告的尝试次数 |
| `mem_le_adv_rcv` | **广告接收计数器** - 收到广告包时递增，用于统计扫描到的设备数量 |
| `mem_le_req_rcv` | **请求接收计数器** - 收到 SCAN_REQ 或 CONNECT_REQ 时递增 |
| `mem_le_scanrsp_rcv` | **扫描响应接收计数器** - 收到 SCAN_RSP 包时递增 |
| `mem_le_conn_rcv` | **连接请求接收计数器** - 收到 CONNECT_REQ 或连接数据包时递增 |

### 2.2 连接管理变量

| 变量名 | 作用 |
|--------|------|
| `mem_current_context` | **当前上下文索引** - 用于多连接管理，指向当前活动的连接上下文 |
| `mem_context_ptr` | **上下文指针** - 指向当前连接上下文数据结构 |
| `mem_rx_window` | **接收窗口大小** - 定义接收数据的时间窗口长度 |

### 2.3 射频/物理层变量

| 变量名 | 作用 |
|--------|------|
| `mem_le_ch_mapped` | **LE 信道映射** - 当前使用的 BLE 信道号（37/38/39 用于广告，0-36 用于数据传输） |
| `mem_last_freq` | **最后使用的频率** - 记录上次通信的频率值 |
| `mem_rssi` | **接收信号强度** - 存储收到数据包的 RSSI 值 |

### 2.4 低功耗管理变量

| 变量名 | 作用 |
|--------|------|
| `mem_lpm_adjust` | **低功耗模式时间调整值** - 用于校准 LPM 唤醒时间 |
| `mem_sync_clke` | **同步时钟扩展值** - 保存连接同步时的钟点值，用于低功耗唤醒时重新同步 |

---

## 3. 蓝牙广播数据变量

### 3.1 核心广播数据

| 变量名 | 作用 | 最大长度 |
|--------|------|----------|
| `mem_le_adv_data` | **广播数据包 (Advertising Data)** | 31 字节 |
| `mem_le_adv_data_len` | 广播数据长度 | 1 字节 |
| `mem_le_scan_data` | **扫描响应数据 (Scan Response Data)** | 31 字节 |
| `mem_le_scan_data_len` | 扫描响应数据长度 | 1 字节 |

### 3.2 广播地址相关

| 变量名 | 作用 |
|--------|------|
| `mem_le_lap` | 本地蓝牙地址 (Local Address)，6 字节 |
| `mem_le_plap` | 对等方/目标蓝牙地址 (Peer Address)，6 字节 |
| `mem_le_adv_own_addr_type` | 本地地址类型（公共/随机） |
| `mem_le_adv_channel_map` | 广播信道映射（37/38/39） |

### 3.3 广播发送相关

| 变量名 | 作用 |
|--------|------|
| `mem_le_adv_type` | 广播类型（ADV_IND、ADV_DIRECT_IND 等） |
| `mem_le_txheader` | 发送数据包头 |
| `mem_le_txlen` | 发送数据长度 |
| `mem_le_txpayload` | 发送数据载荷（实际发送时存放 LAP+ 广播数据） |
| `mem_le_adv_transmit` | 已发送的广播包计数器 |

---

## 4. 广播数据格式解析示例

### 4.1 BLE AD 结构

BLE 广播数据采用 **AD 结构**（Advertising Data Structure）：
- **Byte 1**: AD 长度（Length）
- **Byte 2**: AD 类型（AD Type）
- **Byte 3~N**: AD 数据（Data）

### 4.2 常见 AD 类型

| AD 类型 | 值 | 说明 |
|---------|-----|------|
| Flags | 0x01 | 发现模式标志 |
| Complete Local Name | 0x09 | 完整设备名称 |
| Shortened Local Name | 0x08 | 缩短的设备名称 |
| Complete 16-bit UUID | 0x03 | 完整 16 位服务 UUID |
| Manufacturer Specific | 0xFF | 制造商自定义数据 |

### 4.3 实例分析

**原始数据：**
```
地址     数据
4340:                               0c 09 41 42 20 53 68
4350: 75 74 74 65 72 33 00 00 00 00 00 00 00 00 00 00
4360: 00 00 00 00 00 00 00 00 0d
```

**解析结果：**

| 偏移 | 数据 | 说明 |
|------|------|------|
| 0x434E | `0C` | AD1 长度 = 12 |
| 0x434F | `09` | AD1 类型 = 完整设备名称 |
| 0x4350-0x435A | `41 42 20 53 68 75 74 74 65 72 33` | ASCII: "AB Shutter3" |

**结论：**
- 设备名称：**"AB Shutter3"**
- 第一条 AD 结构完整且格式正确
- 第二条 AD 记录（`0x0D`）可能不完整或被截断

---

## 5. 关键函数分析

### 5.1 timer_single_step_2B

**位置：** `app.prog:130`

**功能：** 2 字节（16 位）定时器递减函数

**参数：**
- `regc`: 定时器变量的内存地址（输入）
- `regb`: 超时后要调用的回调函数地址（输入）

**流程：**
```
timer_single_step_2B:
    ifetch 2,regc        ; 从 regc 指向的地址读取 2 字节数据
    rtn blank            ; 如果值为 0，直接返回
    pincrease -1         ; 值减 1
    istore 2,regc        ; 将减 1 后的值存回 regc 指向的地址
    nrtn blank           ; 如果结果不为 0，返回
    copy regb,pdata      ; 如果减 1 后为 0，将 regb（回调函数地址）传给 pdata
    branch callback_func ; 调用回调函数
```

### 5.2 initialize_radio_wait

**位置：** `bt.prog:835`

**功能：** RF（射频）初始化等待函数

**作用：**
1. 检查 `mem_rf_init_ptr` 的 bit14 标志位
2. 如果 bit14=0，循环等待直到置位
3. 读取 RF 状态寄存器 `0x8a00`
4. 等待寄存器值变为 `0xff`（RF 就绪状态）
5. 就绪后继续后续初始化（DPLL、DCOC、AAC 等）

### 5.3 ble_shutter_reconn_timer

**位置：** `app_shutter.prog:203`

**功能：** 蓝牙快门重连定时器

**流程：**
```
ble_shutter_reconn_timer:
    fetch 1,mem_shutter_soft_switch_power_state
    rtnne SS_STATE_POWER_ON
    arg mem_ble_shutter_reconn_timer,regc    ; 定时器地址
    arg shutter_start_bluetooth_discovery,regb ; 超时回调函数
    branch timer_single_step_2B              ; 递减定时器，超时后启动广播
```

**工作流程：**
1. 设备断电且蓝牙断开时，定时器递减
2. 超时后触发 `shutter_start_bluetooth_discovery`
3. 开始蓝牙发现/广播模式，LED 闪烁
4. 连接成功时定时器清零

---

## 6. 广播数据发送流程

### 6.1 le_send_adv_ind（发送广播包）

**位置：** `le_advertising.prog:222`

```assembly
le_send_adv_ind:
    ; 1. 设置广播头和地址类型
    fetch 1,mem_le_adv_type
    fetcht 1,mem_le_adv_own_addr_type
    storet 1,mem_le_txheader

    ; 2. 计算发送长度 = 广播数据长度 + 6 字节头
    fetcht 1,mem_le_adv_data_len
    add temp,6,pdata
    store 1,mem_le_txlen

    ; 3. 复制本地地址到发送载荷
    fetch 6,mem_le_lap
    store 6,mem_le_txpayload

    ; 4. 复制广播数据到发送缓冲区
    copy temp,loopcnt
    arg mem_le_adv_data,contr      ; ← 广播内容源地址
    call memcpy_fast

    ; 5. 发送广播包
    branch le_send_adv_transmit
```

### 6.2 le_send_scan_response（发送扫描响应）

**位置：** `le_advertising.prog:277`

```assembly
le_send_scan_response:
    ; 设置 SCAN_RSP 类型
    arg SCAN_RSP,temp
    storet 1,mem_le_txheader

    ; 计算长度 = 扫描响应数据长度 + 6 字节
    fetcht 1,mem_le_scan_data_len
    add temp,6,pdata
    store 1,mem_le_txlen

    ; 复制本地地址
    fetch 6,mem_le_lap
    store 6,mem_le_txpayload

    ; 复制扫描响应数据
    arg mem_le_scan_data,contr     ; ← 扫描响应内容源地址
    copy temp,loopcnt
    call memcpy_fast

    ; 发送
    call le_transmit_norx
```

---

## 7. 相关文档

- 项目路径：`D:\YiChip_workspace\BLE`
- 协议栈源码：`program\ble_protocol_stack\`
- 应用示例：`program\app_shutter.prog`

---

## 8. 手机扫描不到设备问题分析

### 8.1 问题现象

- **nRF Connect**: 可以扫描到设备
- **手机系统蓝牙设置**: 扫描不到设备

### 8.2 原因分析

手机系统蓝牙只扫描带有 **Flags AD 类型 (0x01)** 且包含正确可发现标志的设备。

### 8.3 缺少 Flags 字段

**错误示例**（当前代码）：
```assembly
; app_shutter.prog:618-623
jam 0x1f,mem_le_adv_data_len
setsect 0,0x10102      ; ⚠️ 问题：缺少 Flags
setsect 1,0x80c1
setsect 2,0x18120
setsect 3,0x640c
store 9,mem_le_adv_data
```

### 8.4 正确的广播数据结构

```
字节偏移  数据              说明
─────────────────────────────────────────
0x00      02               AD1 长度 = 2
0x01      01               AD1 类型 = Flags
0x02      06               AD1 数据 = General Discoverable
0x03      0C               AD2 长度 = 12
0x04      09               AD2 类型 = Complete Local Name
0x05-0x0F 41 42 20...      "AB Shutter3"
0x10      03               AD3 长度 = 3
0x11      03               AD3 类型 = Complete 16-bit UUID
0x12-0x13 12 18            0x1812 (HID Service)
0x14-0x1E 00 00...         填充到 31 字节
```

### 8.5 Flags 值含义

| 值 | 含义 | 推荐 |
|-----|------|------|
| 0x04 | BR/EDR Not Supported | |
| 0x06 | LE General Discoverable + BR/EDR Not Supported | ✅ 推荐 |
| 0x1A | LE Limited Discoverable + BR/EDR Not Supported | |

---

## 9. shutter_default_init 函数分析

### 9.1 调用时机

| 时机 | 位置 | 说明 |
|------|------|------|
| **系统上电复位** | `bt.prog:40` | ROM 起始地址 `start:` 入口 |
| **软复位** | `bt.prog:43` | `soft_reset:` 标签处 |

### 9.2 调用链路

```
bt.prog:47 (软启动/复位)
    └── call app_param_init

app.prog:19-24
    ├── app_param_init:
    │   fetch 1,mem_device_option
    │   beq DVC_OP_SHUTTER,shutter_default_init
    │   beq DVC_OP_SHUTTER_DY,shutter_default_init
```

### 9.3 功能模块

| 模块 | 行号 | 配置内容 |
|------|------|----------|
| 按键 GPIO | 535-543 | 按键、LED、开关配置 |
| 按键映射 | 549-558 | Key0/Key1/Key2 按下/释放值 |
| 电源管理 | 559-574 | LPM 参数、重连/发现 LED 时间 |
| 连接参数 | 575-579 | 连接间隔 10ms-40ms |
| LED 配置 | 580-596 | LED 结构体、亮灯时间 |
| 广播参数 | 597-605 | 广播间隔、超时、信道 |
| 链路层 | 606-613 | 最大接收八位组数、发现超时 |
| 队列配置 | 614-617 | 队列指针、大小 |
| **广播数据** | 618-625 | ⚠️ 缺少 Flags 字段 |
| 扫描响应 | 626-634 | 扫描响应数据 |
| 按键扫描 | 635-640 | 扫描指针、按键数量 |

---

## 10. ble_shutter_start_discovery 函数分析

### 10.1 功能

启动蓝牙快门**发现模式（广播模式）**，当设备需要被手机扫描时调用。

### 10.2 配置内容

| 配置项 | 寄存器 | 值 | 说明 |
|--------|--------|-----|------|
| 超时时间 | `mem_shutter_sleep_timeout` | 0x0BB8/0x02EE | 约 3 秒后进入睡眠 |
| 广播间隔 | `mem_le_adv_interval` | 0x0140 | 200ms (320 * 0.625ms) |
| LPM 间隔 | `mem_lpm_interval` | 0x013C | 广播间隔 -4 |
| LED 亮灯时间 | `mem_shutter_led_struct_app_led_on_time` | 0x0296 | 发现模式 LED 闪烁 |
| 广播状态 | `mem_ui_state_map` | BIT_BLE_ADV | 设置广播标志 |

### 10.3 执行流程

```
ble_shutter_start_discovery
    │
    ├── [检查] 电源状态 = ON? ──NO──→ 返回
    │
    ├── [检查] 已 BLE 连接？ ──YES──→ 返回
    │
    ├── 设置睡眠超时时间
    │
    ├── 设置广播间隔 (200ms)
    │
    ├── 启动 LED 闪烁 (发现模式)
    │
    ├── [检查] 已在广播？ ──YES──→ 返回
    │
    └── 启动广播
            │
            └── 发送 HCI 命令：BT_CMD_START_ADV
```

---

## 11. LED 配置分析

### 11.1 LED 引脚配置位置

**在 `shutter_default_init` 中配置**（`app_shutter.prog:537-538`）：

```assembly
fetch 1,mem_shutter_led_struct_app_led_gpio_temp
store 1,mem_shutter_led_struct_app_led_gpio   ; LED 引脚配置
store 1,mem_shutter_power_off_led_style_gpio  ; 关机 LED 引脚配置
```

### 11.2 LED 结构体内存布局

| 偏移量常量 | 字段 | 大小 | 说明 |
|-----------|------|------|------|
| `LED_OFFSET_LED_GPIO` | GPIO 引脚 | 1 字节 | LED 连接的 GPIO 引脚号 |
| `LED_OFFSET_ON_TIME` | 点亮时间 | 2 字节 | LED 点亮持续时间 |
| `LED_OFFSET_OFF_TIME` | 熄灭时间 | 2 字节 | LED 熄灭持续时间 |
| `LED_OFFSET_BLINK_COUNT` | 闪烁次数 | 1 字节 | 闪烁次数（0xFF=无限） |
| `LED_OFFSET_CB_LEDON` | 亮灯回调 | 2 字节 | 亮灯回调函数地址 |
| `LED_OFFSET_CB_LEDOFF` | 灭灯回调 | 2 字节 | 灭灯回调函数地址 |
| `LED_OFFSET_TYPE` | LED 状态类型 | 1 字节 | 当前 LED 状态 |
| `LED_OFFSET_LENGTH` | 结构体总长度 | - | 用于计算下一个结构体地址 |

### 11.3 配置两个 LED 的方法

**不能只定义两个 GPIO 字节**，需要为每个 LED 创建完整的结构体：

```
LED 结构体 1: [GPIO][亮灯时间][灭灯时间][闪烁次数][回调...]
LED 结构体 2: [GPIO][亮灯时间][灭灯时间][闪烁次数][回调...]
```

**步骤：**
1. 修改 LED 数量：`jam 0x02,mem_ui_led_struct_num`
2. 定义两个独立结构体数据
3. 设置结构体指针数组

---

## 12. ui_ipc_send_cmd 函数分析

### 12.1 功能

**UI 处理器（C51）与蓝牙处理器（BT）之间的 IPC（处理器间通信）发送命令函数**。

### 12.2 调用链路

```
app_ble_start_adv
    │
    ├── jam BT_CMD_START_ADV,mem_fifo_temp
    │
    ▼
ui_ipc_send_cmd
    │
    ├── 获取 IPC 锁
    ├── 写入 FIFO (mem_ipc_fifo_c512bt)
    ├── 释放锁
    │
    ▼
蓝牙协议栈读取 FIFO 并执行命令
```

### 12.3 IPC 通信架构

```
┌─────────────────┐                    ┌─────────────────┐
│   C51 (MCU)     │                    │   BT (射频)     │
│   UI 处理器      │                    │   协议栈        │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         │ mem_ipc_fifo_c512bt                  │ mem_ipc_fifo_bt2c51
         │ (C51 → BT 命令)                       │ (BT → C51 事件)
         │                                      │
         ▼                                      ▼
    ┌────────┴──────────────────────────────────┴────────┐
    │              FIFO 队列 (共享内存)                    │
    └────────────────────────────────────────────────────┘
```

### 12.4 常用命令字

| 命令 | 说明 |
|------|------|
| `BT_CMD_START_ADV` | 启动广播 |
| `BT_CMD_STOP_ADV` | 停止广播 |
| `BT_CMD_LE_START_SCAN` | 启动扫描 |
| `BT_CMD_LE_STOP_SCAN` | 停止扫描 |
| `BT_CMD_LE_DISCONNECT` | 断开连接 |

### 12.5 锁机制

```assembly
ui_ipc_get_lock:
    jam 1,mem_ipc_lock_bt           ; C51 设置 BT 锁
    fetch 1,mem_ipc_lock_c51        ; 检查 C51 锁
    nbranch ui_ipc_get_lock_wait,blank ; 如果 C51 锁=1，等待
    rtn

ui_ipc_put_lock:
    jam 0,mem_ipc_lock_bt           ; 释放 BT 锁
```

---

*文档生成时间：2026-04-01 | 最后更新：2026-04-01*
