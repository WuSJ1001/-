# YiChip BLE 芯片笔记

本文档记录了 YiChip BLE 蓝牙芯片固件中的关键变量、常量定义及其使用方式。

---

## 目录

1. [UI 状态映射变量 (mem_ui_state_map)](#1-ui-状态映射变量-mem_ui_state_map)
2. [蓝牙常量定义](#2-蓝牙常量定义)
3. [IPC 通信机制](#3-ipc-通信机制)
4. [BLE 设备地址 (mem_le_lap)](#4-ble-设备地址-mem_le_lap)

---

## 1. UI 状态映射变量 (mem_ui_state_map)

### 1.1 定义

```format
// format/ui.format:23
2 mem_ui_state_map    ; 2 字节（16 位）内存空间
```

### 1.2 状态位定义

```format
// format/ui.format:35-41
(
9  UI_STATE_BLE_CONNECTED    ; BLE 已连接状态
10 UI_STATE_BLE_WRITE_RCV    ; BLE 写入接收状态
11 UI_STATE_BLE_ADV          ; BLE 广播中状态
12 UI_STATE_BTN_DOWN         ; 按钮按下状态
)
```

### 1.3 设计模式

这是一种**位图状态机**设计模式：
- 使用单个内存变量的不同 bit 位来表示多个独立的布尔状态
- 节省内存空间，便于状态的快速检查和修改
- 在嵌入式系统中常见，可高效管理和传输多个状态标志

### 1.4 使用示例

**设置状态位**（设置按钮按下）：
```asm
fetch 2,mem_ui_state_map           ; 读取 2 字节状态到 pdata
set1 UI_STATE_BTN_DOWN,pdata       ; 将第 12 位置 1 (1<<12 = 0x1000)
store 2,mem_ui_state_map           ; 存回内存
```

**清除状态位**（清除按钮按下）：
```asm
fetch 2,mem_ui_state_map
set0 UI_STATE_BTN_DOWN,pdata       ; 将第 12 位置 0
store 2,mem_ui_state_map
```

**检查状态位**：
```asm
; 检查按钮是否按下
ui_check_paring_button:
    fetch 1,mem_ui_state_map
    rtnbit0 UI_STATE_BTN_DOWN      ; 测试第 12 位，如果为 0 则返回
    rtn

; 检查 BLE 连接状态
bbit1 UI_STATE_BLE_CONNECTED,app_led_off  ; 如果第 9 位为 1，跳转
```

### 1.5 相关位操作指令

| 指令 | 作用 |
|------|------|
| `set1 bit_pos, reg` | 将寄存器中指定位置 1 |
| `set0 bit_pos, reg` | 将寄存器中指定位置 0 |
| `rtnbit bit_pos, reg` | 测试指定位，根据结果返回 |
| `bbit1 bit_pos, label` | 如果位为 1 则跳转 |
| `bbit0 bit_pos, label` | 如果位为 0 则跳转 |
| `isolate1 bit_pos, reg` | 隔离指定位到另一个寄存器 |

---

## 2. 蓝牙常量定义

### 2.1 蓝牙命令 (BT_CMD_*)

用于 51 单片机与蓝牙芯片之间的 IPC 通信命令：

```format
// format/ui.format:45-66
(
0  BT_CMD_STANDBY
13 BT_CMD_START_ADV           ; 开始广播
14 BT_CMD_STOP_ADV            ; 停止广播
15 BT_CMD_START_DIRECT_ADV    ; 开始定向广播
16 BT_CMD_STOP_DIRECT_ADV     ; 停止定向广播
17 BT_CMD_LE_DISCONNECT       ; LE 断开连接
18 BT_CMD_LE_UPDATE_CONN      ; LE 更新连接参数
19 BT_CMD_LED_OFF             ; LED 关闭
20 BT_CMD_LED_ON              ; LED 开启
21 BT_CMD_LED_BLINK           ; LED 闪烁
22 BT_CMD_LE_START_CONN       ; LE 开始连接
23 BT_CMD_LE_START_SCAN       ; LE 开始扫描
24 BT_CMD_LE_STOP_SCAN        ; LE 停止扫描
25 BT_CMD_ENTER_HIBERNATE     ; 进入休眠
27 BT_CMD_LE_SMP_SECURITY_REQUEST
31 BT_CMD_STORE_RECONN_INFO_LE
34 BT_CMD_START_24G           ; 启动 2.4G
35 BT_CMD_STOP_24G            ; 停止 2.4G
36 BT_CMD_PAIR_24G            ; 2.4G 配对
)
```

### 2.2 蓝牙事件 (BT_EVT_*)

用于蓝牙芯片向 51 单片机上报事件：

```format
// format/ui.format:68-100
(
0x00 BT_EVT_NULL
0x0F BT_EVT_DISCOVERY_STOPED
0x10 BT_EVT_BUTTON_LONG_PRESSED   ; 按钮长按
0x14 BT_EVT_LE_CONNECTED          ; LE 已连接
0x15 BT_EVT_LE_DISCONNECTED       ; LE 已断开
0x17 BT_EVT_BUTTON_ENTER_HIBERNATE
0x25 BT_EVT_BUTTON_ADJUST_DPI
0x28 BT_EVT_LE_WRITE_REQUEST
0x29 BT_EVT_LE_ENC_INFO
0x2c BT_EVT_BUTTON_DOWN           ; 按钮按下
0x2d BT_EVT_BUTTON_UP             ; 按钮释放
0x2e BT_EVT_REMOTE_UNSNIFF
0x30 BT_EVT_LE_PAIRING_FAIL       ; LE 配对失败
0x31 BT_EVT_LE_PAIRING_SUCCESS    ; LE 配对成功
0x32 BT_EVT_LE_START_ENC          ; LE 开始加密
0x33 BT_EVT_LE_PAUSE_ENC
0x34 BT_EVT_LE_TK_GENERATE
0x35 BT_EVT_BT_GKEY_GENERATE
0x36 BT_EVT_BT_GET_PASSKEY
0x39 BT_EVT_24G_PAIRING_COMPLETE  ; 2.4G 配对完成
0x3a BT_EVT_24G_ATTEMPT_FAIL
0x3b BT_EVT_LE_GKEY_GENERATE
0x3c BT_EVT_24G_ATTEMPT_SUCCESS
0x3d BT_EVT_STORE_NVRAM
0x3e BT_EVT_LE_PAIRING_COMPLETE
0x3F BT_EVT_LE_RECONNECT_COMPLETE
0x40 BT_EVT_LE_PARSE_CONN_PAPA_UPDATE_RSP
0x41 BT_EVT_LE_LTK_LOST
0x42 BT_EVT_LE_UPDATE_PHY
0x43 BT_EVT_LE_GET_PASSKEY
0x44 BT_EVT_LE_PARSE_CONN_PARAM_ACCEPTED
)
```

### 2.3 广播类型 (ADV_*)

```format
// format/ble_protocol_stack/le_advertising.format:82-93
(
0 ADV_IND            ; 可连接可扫描广播
1 ADV_DIRECT_IND     ; 定向广播
2 ADV_NONCONN_IND    ; 不可连接广播
3 SCAN_REQ           ; 扫描请求
4 SCAN_RSP           ; 扫描响应
5 CONNECT_REQ        ; 连接请求
6 ADV_SCAN_IND       ; 可扫描广播
)
```

### 2.4 为什么值是这些？

| 常量类型 | 值的来源 |
|---------|---------|
| `UI_STATE_*` | 位序号，用于 `set1/set0/rtnbit` 等位操作指令 |
| `BT_CMD_*` | 命令 opcode，用于 IPC 命令分发（switch-case 风格） |
| `BT_EVT_*` | 事件类型码，用于 IPC 事件上报 |
| `ADV_*` | 蓝牙规范定义的广播类型值 |
| `ON/OFF` | 布尔值 1/0 |
| `LED_OFFSET_*` | LED 结构体成员偏移量（字节） |

**注意**：命令值不连续（如 0, 13, 14...25, 27, 31...）的原因：
1. 历史原因 - 早期版本可能定义过其他命令
2. 分类预留 - 不同范围预留给不同类别的命令

---

## 3. IPC 通信机制

### 3.1 FIFO 定义

```format
// format/ui.format:17-18
8 mem_ipc_fifo_bt2c51    ; 蓝牙→51 的 FIFO，8 字节（容量）
8 mem_ipc_fifo_c512bt    ; 51→蓝牙的 FIFO，8 字节（容量）
```

### 3.2 FIFO 结构

```
mem_ipc_fifo_bt2c51 的内存布局（8 字节 = 8 个数据项）：
┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
│ byte 0 │ byte 1 │ byte 2 │ byte 3 │ byte 4 │ byte 5 │ byte 6 │ byte 7 │
│ event1 │ event2 │ event3 │ event4 │ event5 │ event6 │ event7 │ event8 │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
   ↑                                                        ↑
 读指针                                                     写指针
```

**关键点**：
- `UTIL_FIFO_LEN = 8` 表示 FIFO 可以容纳 **8 个独立的数据项**
- 每个数据项是 **1 字节（8 位）**
- 每次 `fifo_in` 只写入 **1 字节** 的数据

### 3.3 事件上报流程（蓝牙 → 51）

```asm
; 1. 准备事件数据
jam BT_EVT_BUTTON_DOWN,mem_fifo_temp    ; mem_fifo_temp = 0x2c

; 2. 发送到 IPC FIFO
ui_ipc_send_event:
    call ui_ipc_get_lock               ; 获取 IPC 锁
    arg mem_ipc_fifo_bt2c51,rega       ; 选择蓝牙→51 的 FIFO
    call fifo_in                       ; 写入 FIFO（1 字节）
    branch ui_ipc_put_lock
```

### 3.4 命令接收流程（51 → 蓝牙）

```asm
check_51cmd_once:
    call ui_ipc_get_lock
    arg mem_ipc_fifo_c512bt,rega       ; 选择 51→蓝牙的 FIFO
    call fifo_out                      ; 从 FIFO 取出数据
    copy pdata,temp                    ; temp = BT_CMD_* 值
    call ui_ipc_put_lock

    ; 根据命令值分发处理
    beq BT_CMD_START_ADV,check_51cmd_adv
    beq BT_CMD_STOP_ADV,check_51cmd_stop_adv
    ...
```

### 3.5 数据流向图

```
┌─────────────┐    BT_EVT_BUTTON_DOWN(0x2c)   ┌─────────────┐
│  蓝牙芯片   │ ────────────────────────────→ │  51 单片机  │
│  (BT)       │      mem_ipc_fifo_bt2c51      │  (C51)      │
│             │         FIFO (8 字节)          │             │
│  jam 0x2c   │ ────────────────────────────→ │  beq 0x2c   │
│  fifo_in    │                               │  分支处理   │
└─────────────┘                               └─────────────┘
```

### 3.6 BT_EVT_* 值的理解

`BT_EVT_*` 是**FIFO 中数据的类型索引/操作码**：
- 每个事件值是 1 字节（0x00-0xFF），最多支持 256 种事件
- 51 单片机读取后，根据值进行 `switch-case` 风格的分支处理
- 类似于操作系统中的**消息队列**或**信号量**机制

---

## 4. BLE 设备地址 (mem_le_lap)

### 4.1 定义

```format
// format/ble_protocol_stack/le.format:153
3 mem_le_lap    ; 实际使用为 6 字节
```

### 4.2 含义

`mem_le_lap` 存储的是 **蓝牙设备的 6 字节 MAC 地址**（BD_ADDR / LE Address）。

虽然变量名是 `lap`（来自经典蓝牙的 Lower Address Part 命名习惯），但在这里它实际上存储的是完整的 **6 字节 BLE 设备地址**。

### 4.3 蓝牙地址结构

标准蓝牙地址是 6 字节（48 位）：
```
字节：  [5]    [4]    [3]    [2]    [1]    [0]
       NAP(高 2 字节)   UAP(1 字节)    LAP(低 3 字节)
```

### 4.4 使用场景

**设置设备地址**：
```asm
module_hci_cmd_set_le_addr:
    fetch 1,mem_module_uart_len
    bne 6,module_hci_event_receive_invalid_cmd  ; 必须是 6 字节
    ifetch 6,contru
    store 6,mem_le_lap    ; 存储 6 字节蓝牙地址
```

**广播时发送自身地址**：
```asm
fetch 6,mem_le_lap
istore 6,contw  ; 将自身地址填入广播包
```

**生成随机地址**：
```asm
storet 2,mem_le_lap+1   ; 设置随机地址的高位
store 1,mem_le_lap      ; 设置低位
```

### 4.5 sched 文件中的预设地址

sched 文件夹下的 `.dat` 文件为不同产品预设了不同的蓝牙地址：

| 文件 | 设备类型 | 蓝牙地址示例 |
|------|---------|-------------|
| `shutter.dat` | 快门 | `20 31 27 98 07 2a` |
| `mouse.dat` | 鼠标 | `71 81 91 a1 b1 c1` |
| `keyboard.dat` | 键盘 | `73 83 92 a1 b1 c1` |
| `car.dat` | 车载设备 | `34 75 07 99 07 2b` |

这些地址在生产时会被烧录到设备的 OTP 或 Flash 中，作为设备的唯一标识。

### 4.6 总结

| 项目 | 值 |
|------|-----|
| **变量名** | `mem_le_lap` |
| **大小** | 6 字节 |
| **含义** | BLE 设备 MAC 地址 |
| **用途** | 广播、扫描、连接时标识设备身份 |
| **命名来源** | 历史原因（经典蓝牙 LAP 命名习惯） |

---

## 附录：相关文件路径

```
D:\YiChip_workspace\BLE\
├── format/
│   ├── ui.format                    ; UI 相关定义
│   ├── bt.format                    ; 蓝牙基础定义
│   └── ble_protocol_stack/
│       ├── le.format                ; BLE 协议栈定义
│       └── le_advertising.format    ; BLE 广播定义
├── program/
│   ├── ui.prog                      ; UI 处理逻辑
│   ├── app_module.prog              ; 应用模块逻辑
│   └── ble_protocol_stack/
│       └── le_advertising.prog      ; BLE 广播逻辑
└── sched/
    ├── shutter.dat                  ; 快门调度配置
    ├── mouse.dat                    ; 鼠标调度配置
    └── keyboard.dat                 ; 键盘调度配置
```

---

*文档生成日期：2026-04-02*
