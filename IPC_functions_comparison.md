# IPC 函数区别与命令类型详解

## 1. IPC_TxHidData 函数

### 功能
**发送 HID（人机接口设备）数据**，主要用于传输键盘按键信息、鼠标数据等用户输入数据。

### 目标
- **最终目标**：主机设备（如电脑、手机等）
- **直接目标**：根据当前连接状态，发送到不同的模块：
  - 蓝牙 BR/EDR 模块（经典蓝牙）
  - 蓝牙 BLE 模块（低功耗蓝牙）
  - 2.4G 无线模块

### 实现原理
```c
void IPC_TxHidData(byte* dt, byte len)
{
    xbyte tx_hid_buff[10];
    byte i=0;

    for(i==0; i< len; i++)
    {
        tx_hid_buff[i] = dt[i];
    }
    
    if(!YC_check_need_reconnected()) 
        return;
    if (g_variable.release_data)
        return;

    if (g_variable.br_currentState == CHANGE_TO_BR_CONNECTED)
    {
        // 发送到经典蓝牙模块
        IPC_TxBREDRHidData(tx_hid_buff,len);
    }
    if ((g_variable.ble_currentState == CHANGE_TO_BLE_CONNECTED) ||
    ((g_variable.ble_currentState == CHANGE_TO_BLE_CONNECTING) && ble_fast_connect_flag))
    {
        // 发送到BLE模块
        IPC_TxBleData(tx_hid_buff,len);
    }
    if (g_variable.g24_currentState == CHANGE_TO_24G_CONNECTED)
    {
        // 发送到2.4G模块
        IPC_Tx24GData(tx_hid_buff,len);
    }
}
```

### 应用场景
- 按键按下/释放事件
- 多媒体按键操作
- 鼠标移动/点击事件
- 系统控制按键（如电源、睡眠等）

### 数据格式
- 标准 HID 报告格式
- 报告 ID + 数据
- 长度通常为 3-9 字节

## 2. IPC_TxControlCmd 函数

### 功能
**发送控制命令**，用于控制蓝牙模块、2.4G模块的行为和状态。

### 目标
- **直接目标**：蓝牙模块或2.4G模块
- **间接目标**：影响模块的工作状态和行为

### 实现原理
- 发送单字节命令码
- 通过 IPC 通信机制传递给目标模块
- 模块根据命令执行相应的操作

### 应用场景
- 开始/停止设备发现
- 建立/断开连接
- 进入/退出低功耗模式
- 启动/停止2.4G模式
- 设备配对操作
- 设备切换操作

## 3. 主要区别

| 特性 | `IPC_TxHidData` | `IPC_TxControlCmd` |
|------|----------------|-------------------|
| **发送内容** | HID数据（按键、鼠标等） | 控制命令 |
| **目标** | 主机设备 | 蓝牙/2.4G模块 |
| **用途** | 用户输入操作 | 模块状态控制 |
| **数据格式** | 标准HID报告格式 | 单字节命令码 |
| **调用时机** | 用户操作时 | 系统状态变化时 |
| **参数** | 数据缓冲区和长度 | 命令码 |
| **返回值** | 无 | 无 |

## 4. IPC_TxControlCmd 命令类型

### 4.1 发现与连接命令

| 命令 | 代码 | 功能 |
|------|------|------|
| `IPC_CMD_START_DISCOVERY` | 0x01 | 开始设备发现 |
| `IPC_CMD_STOP_DISCOVERY` | 0x02 | 停止设备发现 |
| `IPC_CMD_RECONNECT` | 0x03 | 重新连接设备 |
| `IPC_CMD_DISCONNECT` | 0x04 | 断开连接 |
| `IPC_CMD_START_INQUIRY` | 0x0b | 开始查询 |
| `IPC_CMD_STOP_INQUIRY` | 0x0c | 停止查询 |

### 4.2 低功耗模式命令

| 命令 | 代码 | 功能 |
|------|------|------|
| `IPC_CMD_ENTER_SNIFF` | 0x05 | 进入嗅探模式（低功耗） |
| `IPC_CMD_EXIT_SNIFF` | 0x06 | 退出嗅探模式 |
| `IPC_CMD_ENTER_SNIFF_SUBRATING` | 0x07 | 进入子速率嗅探模式 |
| `IPC_CMD_EXIT_SNIFF_SUBRATING` | 0x08 | 退出子速率嗅探模式 |
| `IPC_CMD_ENTER_HIBERNATE` | 0x19 | 进入休眠模式 |

### 4.3 蓝牙LE相关命令

| 命令 | 代码 | 功能 |
|------|------|------|
| `IPC_CMD_START_ADV` | 0x0d | 开始广播 |
| `IPC_CMD_STOP_ADV` | 0x0e | 停止广播 |
| `IPC_CMD_START_DIRECT_ADV` | 0x0f | 开始定向广播 |
| `IPC_CMD_STOP_DIRECT_ADV` | 0x10 | 停止定向广播 |
| `IPC_CMD_LE_DISCONNECT` | 0x11 | 断开BLE连接 |
| `IPC_CMD_LE_UPDATE_CONN` | 0x12 | 更新BLE连接参数 |
| `IPC_CMD_LE_START_CONN` | 0x16 | 开始BLE连接 |
| `IPC_CMD_LE_START_SCAN` | 0x17 | 开始BLE扫描 |
| `IPC_CMD_LE_STOP_SCAN` | 0x18 | 停止BLE扫描 |
| `IPC_CMD_LE_SMP_SECURITY_REQUEST` | 0x1b | BLE安全请求 |
| `IPC_CMD_LE_START_WRITE` | 0x1c | 开始BLE写入 |
| `IPC_CMD_LE_SET_PINCODE` | 0x29 | 设置BLE PIN码 |
| `IPC_CMD_START_ADV_REC` | 0x2b | 开始广播并准备重连 |
| `IPC_CMD_START_ADV_DISCOVERY` | 0x2c | 开始广播和发现 |

### 4.4 2.4G相关命令

| 命令 | 代码 | 功能 |
|------|------|------|
| `IPC_CMD_START_24G` | 0x21 | 启动2.4G模式 |
| `IPC_CMD_STOP_24G` | 0x22 | 停止2.4G模式 |
| `IPC_CMD_PAIR_24G` | 0x23 | 开始2.4G配对 |

### 4.5 LED控制命令

| 命令 | 代码 | 功能 |
|------|------|------|
| `IPC_CMD_LED_OFF` | 0x13 | 关闭LED |
| `IPC_CMD_LED_ON` | 0x14 | 打开LED |
| `IPC_CMD_LED_BLINK` | 0x15 | LED闪烁 |

### 4.6 其他命令

| 命令 | 代码 | 功能 |
|------|------|------|
| `IPC_CMD_STANDBY` | 0x00 | 待机模式 |
| `IPC_CMD_SET_PIN_CODE` | 0x0a | 设置PIN码 |
| `IPC_CMD_ROLE_SWITCH` | 0x1d | 角色切换 |
| `IPC_CMD_BB_RECONN_CANCEL` | 0x1e | 取消BR/EDR重连 |
| `IPC_CMD_STORE_RECONN_INFO_LE` | 0x1f | 存储BLE重连信息 |
| `IPC_CMD_STORE_RECONN_INFO_BT` | 0x20 | 存储BR/EDR重连信息 |
| `IPC_CMD_DEVICE_SWITCH` | 0x24 | 设备切换 |
| `IPC_CMD_UPDATE_SUPERVISION_TO` | 0x28 | 更新监督超时 |
| `IPC_CMD_SET_RECONNECT_INIT` | 0x2a | 初始化重连 |

## 5. 技术意义

- **分层设计**：将用户输入数据与系统控制命令分离，使代码结构更清晰
- **模块化**：不同类型的数据通过不同的函数发送，便于维护和扩展
- **效率优化**：HID数据和控制命令使用不同的传输路径，优化通信效率
- **状态管理**：通过控制命令管理模块状态，确保系统稳定运行

## 6. 总结

`IPC_TxHidData` 和 `IPC_TxControlCmd` 是两个不同功能的IPC（进程间通信）函数：
- **IPC_TxHidData** 负责将用户输入的HID数据发送给主机设备，支持多种连接模式
- **IPC_TxControlCmd** 负责向蓝牙或2.4G模块发送控制命令，管理模块的工作状态

它们共同构成了键盘系统与外部设备通信的核心机制，确保了键盘功能的正常运行。通过合理使用这两个函数，可以实现键盘的各种功能，包括按键输入、设备切换、低功耗管理等。