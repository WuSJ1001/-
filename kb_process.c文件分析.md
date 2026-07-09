## YC_key_action_handle 函数执行逻辑
### 函数概述
这是一个 按键动作处理分发函数 ，根据不同的 key_num （按键编号）执行相应的蓝牙/2.4G 控制操作。

### 整体结构
```
YC_key_action_handle(key_num)
├── 检查当前连接状态
│   ├── 2.4G 已连接/连接中 → 先停止 2.4G
│   ├── 有活动连接/重连 → 保存动作，等连接断开
│   └── 无活动连接 → 直接执行
└── switch(key_num) 分发处理
```
### 各按键处理逻辑
按键 功能 核心操作 KEY_RECON_0/1/2 重连指定索引设备 断开 → 触发重连 (索引 0/1/2) KEY_STOP_DISCOVERY 停止搜索 停止 BLE 广播和搜索 KEY_DISCOVERY 开始设备搜索 清标志 → 根据模式启动 Discovery KEY_DISCONNECT_ALL 断开所有连接 断开 BR/BLE/2.4G KEY_CLEAR_RECORD 清除配对记录 清除 EEPROM 配对列表 KEY_START_24G 启动 2.4G 读取地址 → 连接/快速连接 KEY_OPEN_24G 打开 2.4G 同上 KEY_PAIRING_24G 2.4G 配对模式 发送配对命令，设置 3 分钟超时

### 关键设计模式
1. 状态检查优先 ：大多数操作前会检查是否存在活动连接
2. 动作保存机制 ：如果当前有连接在处理，先保存按键动作，等处理完再执行
3. 超时保护 ：配对和 Discovery 都有超时机制
4. 多模式支持 ：BT/BLE/BLE+BT 综合模式，通过编译宏区分
### 简单总结 这个函数是 蓝牙/2.4G 控制中枢 ，根据用户按下的按键：

- 发起连接/断开
- 搜索/配对设备
- 清除记录
协调 BR（经典蓝牙）、BLE（低功耗蓝牙）、2.4G（无线适配器）三种通信模式。

---

## IPC_TxHidData 函数执行逻辑
### 函数概述
这是一个 HID 数据发送分发函数 ，将键盘按键数据通过当前连接的通道（BR/BLE/2.4G）发送出去。

### 整体流程图
```
IPC_TxHidData(dt, len)
│
├─ 1. 复制数据到本地缓冲区
│
├─ 2. 前置检查
│   ├─ YC_check_need_reconnected() 返回 false → 直接 return
│   └─ release_data == 1 → 直接 return（有待释放数据）
│
├─ 3. 分通道发送
│   ├─ BR 已连接 ──→ 转换 SYSTEM → 3 ──→ IPC_TxBREDRHidData()
│   ├─ BLE 已连接 ──→ 转换 SYSTEM → 3 ──→ IPC_TxBleData()
│   └─ 2.4G 已连接 ──→ 根据 Report ID 转换格式
│                   │   ReportID_1 → 4 (并判断释放)
│                   │   ReportID_2 → 5 (并判断释放)
│                   │   ReportID_3 → 7
│                   └─→ IPC_Tx24GData()
```
### 各通道处理逻辑
| 通道   | 连接条件                                                   | Report ID        | 特殊处理            |
| ---- | ------------------------------------------------------ | ---------------- | --------------- |
| BR   | br_currentState == CONNECTED                           | SYSTEM(0x06) → 3 | 发送 BREDRHidData |
| BLE  | BLE_CONNECTED 或 ( BLE_CONNECTING + fast_connect_flag ) | SYSTEM(0x06) → 3 | 发送 BleData      |
| 2.4G | g24_currentState == CONNECTED                          | 见下方              | 判断按键释放          |

### 2.4G 通道 Report ID 映射
| 原始ID                  | 转换后 | 说明                |
| --------------------- | --- | ----------------- |
| HID_REPORTID_1 (0x01) | 4   | 标准键盘，自动判断释放       |
| HID_REPORTID_2 (0x02) | 5   | 媒体键，自动判断释放        |
| HID_REPORTID_3 (0x03) | 7   | Consumer 控制，无释放判断 |

### 按键释放判断（2.4G）
```
// ReportID_1 释放判断
if (buff[1]==0 && buff[3]==0 && buff[4]==0 && buff[5]==0)
    repeat_send_24g = 0;  // 全部为0，是释放
else
    repeat_send_24g = 1;  // 有按键按下

// ReportID_2 释放判断
if (buff[1]==0 && buff[2]==0)
    repeat_send_24g = 0;  // 释放
else
    repeat_send_24g = 1;  // 有按键按下
```
### 简单总结 数据分发逻辑 ：

1. 先检查是否能发送（是否需要重连、是否有待释放数据）
2. 根据当前连接的通道（BR/BLE/2.4G），选择对应的发送函数
3. BR 和 BLE 对系统键有 Report ID 转换
4. 2.4G 有独立的协议格式，需要转换 Report ID
5. 2.4G 模式下还会判断是否为按键释放，设置 repeat_send_24g 标志
特点 ：支持多通道并发发送（三个 if 是独立的），数据会同时发送到所有已连接的通道。

---

