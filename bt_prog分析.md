# bt.prog 代码分析

## 概述

`bt.prog` 是一个**蓝牙基带固件**，运行在专用蓝牙 SOC 上。该文件包含了蓝牙协议栈底层、射频控制、低功耗管理等核心功能。

---

## 一、系统初始化相关

| 函数 | 作用 | 行号 |
|------|------|------|
| `start` | 系统入口，检查 LPM 状态 | 39 |
| `soft_reset` | 软复位，初始化堆栈、无线电、UI 等 | 42 |
| `initialize_radio` | 射频初始化（RF PLL、ADC 校准等） | 843 |
| `init_param` | 初始化蓝牙协议栈参数 | 1226 |

### 初始化流程

```assembly
start:
    call lpmstate              ; 检查低功耗状态

soft_reset:
    bpatch patch00_0,mem_patch00
    clear_stack                ; 清空堆栈
    call app_param_init        ; 应用参数初始化
    call initialize_radio      ; 射频初始化
    call iic_init_360khz       ; I2C 初始化
    call init_param            ; 协议栈参数初始化
    call l2cap_init              ; L2CAP 层初始化
    call ui_init               ; UI 初始化
    call app_init              ; 应用初始化
```

---

## 二、主循环调度（核心）

### main_loop 结构 (行 70-83)

```assembly
main_loop:
    bpatch patch00_2,mem_patch00
    call le_advertising_dispatch   ; BLE 广播调度
    call idle_dispatch             ; 空闲调度
    call app_process_idle          ; 应用空闲处理
    call connection_dispatch       ; 连接调度
    call g24_dispatch              ; 2.4G 调度
    call lpm_dispatch              ; 低功耗调度
    call kscan_dispatch            ; 键盘扫描调度
    branch main_loop
```

### 调度函数说明

| 函数 | 功能描述 |
|------|----------|
| `le_advertising_dispatch` | 处理 BLE 广播事件 |
| `idle_dispatch` | 处理空闲状态事件（如创建连接） |
| `app_process_idle` | 应用层空闲处理（UI、基带事件等） |
| `connection_dispatch` | 管理蓝牙连接上下文 |
| `g24_dispatch` | 2.4G 私有协议处理 |
| `lpm_dispatch` | 低功耗模式决策 |
| `kscan_dispatch` | 键盘扫描处理 |

---

## 三、低功耗管理（LPM）- 关键！

### LPM 模式类型

| 模式 | 描述 |
|------|------|
| `lpm_doze` | 打盹模式，无保持内存 |
| `lpm_sleep` | 睡眠模式，保持内存 |
| `lpm_hibernate` | 休眠模式，无保持内存 |

### 核心函数

| 函数 | 作用 | 行号 |
|------|------|------|
| `lpm_dispatch` | 低功耗模式调度 | 1616 |
| `lpm_check_wake_lock` | 检查唤醒锁（阻止睡眠） | 1751 |
| `lpm_sleep` | 进入睡眠模式 | 1487 |
| `lpm_hibernate` | 进入休眠（无保持内存） | 1478 |
| `lpm_get_wake_lock` | 获取唤醒锁 | 1737 |
| `lpm_put_wake_lock` | 释放唤醒锁 | 1743 |
| `lpm_recover_clk` | 从 LPM 恢复时钟 | 1545 |
| `lpm_save_context` | 保存 LPM 上下文 | 1429 |
| `lpm_load_context` | 加载 LPM 上下文 | 1413 |

### 唤醒锁机制

```assembly
lpm_check_wake_lock:
    call app_check_wake_lock      ; 应用层唤醒锁检查
    fetch 2,mem_lpm_wake_lock
    ; 检查以下唤醒源:
    ; - wake_lock_ble_tx    : BLE 发送
    ; - wake_lock_ipc_bt2c51: IPC 队列 (BT→C51)
    ; - wake_lock_ipc_c512bt: IPC 队列 (C51→BT)
    ; - wake_lock_cmd       : HCI 命令
    ; - wake_lock_uart_rx/tx: UART 收发
```

---

## 四、连接管理

### 连接上下文调度

| 函数 | 作用 | 行号 |
|------|------|------|
| `connection_dispatch` | 连接上下文调度 | 85 |
| `connection_incontext` | 检查连接是否有效 | 92 |
| `context_load` | 加载连接上下文 | 135 |
| `context_save` | 保存连接上下文 | 148 |
| `context_new` | 创建新连接上下文 | 178 |
| `context_check_idle` | 检查是否空闲（无连接） | 190 |

### 上下文搜索函数

| 函数 | 搜索条件 | 行号 |
|------|----------|------|
| `context_search_empty` | 空上下文 | 231 |
| `context_search_conn_handle` | 连接句柄 | 197 |
| `context_search_plap` | PLAP(Periodic Advertising) | 204 |
| `context_search_insniff` | Sniff 锚点 | 211 |
| `context_search_sniff_window` | Sniff 窗口 | 349 |

### Sniff 模式相关

```assembly
context_search_sniff:
    ; 检查 Sniff 锚点是否匹配
    ; 计算下一个 Sniff 事件
    ; 处理 Sniff 窗口冲突
```

---

## 五、射频（RF）相关

### 射频初始化

| 函数 | 作用 | 行号 |
|------|------|------|
| `initialize_radio` | 射频总初始化 | 843 |
| `rfpll_aac_ghpc` | RF PLL 校准 | 908 |
| `rx_dcoc` | RX DC 偏移校准 | 499 |
| `dpll_ring_ibias_calc` | DPLL 偏置电流计算 | 990 |

### 频率设置

| 函数 | 作用 | 行号 |
|------|------|------|
| `set_freq_rx` | 设置接收频率 | 567 |
| `set_freq_tx` | 设置发射频率 | 747 |
| `calc_freq` | 计算频率合成器参数 | 694 |
| `set_lemode` | 设置 BLE 模式 (1M/2M/Coded) | 612 |

### 功率控制

| 函数 | 作用 | 行号 |
|------|------|------|
| `set_tx_power` | 设置发射功率 | 770 |
| `gain_control` | 自动增益控制（AGC） | 1105 |
| `save_rssi` | 保存 RSSI 值 | 1055 |
| `rf_rx_enable` | 启用接收器 | 639 |
| `txon` | 开启发射 | 752 |

### 发射功率档位

```assembly
set_tx_power:
    ; 支持的功率档位:
    ; TX_POWER_10DB   : +10dBm
    ; TX_POWER_7DB    : +7dBm
    ; TX_POWER_5DB    : +5dBm
    ; TX_POWER_3DB    : +3dBm
    ; TX_POWER_0DB    : 0dBm
    ; TX_POWER_F3DB   : -3dBm
    ; TX_POWER_F5DB   : -5dBm
    ; TX_POWER_F10DB  : -10dBm
    ; TX_POWER_F20DB  : -20dBm
```

---

## 六、与 app.prog 的接口（IPC）

### 回调函数

| 回调 | 作用 | 调用位置 |
|------|------|----------|
| `mem_cb_idle_process` | 空闲回调 | `app_process_idle` |
| `mem_cb_bb_event_process` | 基带事件处理回调 | `app_process_bb_event_priority` |
| `mem_cb_before_lpm` | 进入 LPM 前回调 | `lpm_dispatch_lpo` |
| `mem_cb_check_wakelock` | 唤醒锁检查回调 | `lpm_check_wake_lock` |
| `mem_cb_event_timer` | 事件定时器回调 | `app_evt_timer` |
| `mem_cb_le_process` | LE 处理回调 | `app_process_ble` |

### IPC 通信队列

| 队列 | 方向 | 用途 |
|------|------|------|
| `mem_ipc_fifo_bt2c51` | BT → C51 | 基带事件通知 |
| `mem_ipc_fifo_c512bt` | C51 → BT | HCI 命令发送 |

### 共享内存结构

```assembly
; mem_power_param struct
{
    unsigned char  mem_power_state
    unsigned char  mem_power_timer
    unsigned char  mem_power_off_timeout
    unsigned char  mem_power_starting_timeout
    unsigned long  mem_power_off_cb
    unsigned long  mem_power_starting_cb
    unsigned long  mem_power_standby_cb
    unsigned long  mem_ui_button_up_cb
}
```

---

## 七、Patch 机制

### 硬件 Patch Controller

```assembly
; Patch 触发机制:
; 1. Monitor: 硬件监控 PC 指针
; 2. Check: 当 PC 到达 Patch Hook 点，查询 Patch Mask RAM
; 3. Trigger: 如果掩码位为 1:
;    - Action A: 强制 PC 跳转到 RAM 基地址 0x0000
;    - Action B: 硬件自动将 Patch ID 写入 pdata 累加器
```

### Patch 点分布

| Patch 点 | 用途 |
|----------|------|
| `patch00_x` | 系统初始化、主循环 |
| `patch01_x` | 上下文搜索 |
| `patch02_x` | 射频设置 |
| `patch03_x` | 射频控制、校准 |
| `patch04_x` | LPM 相关 |
| `patch05_x` | LPM 调度 |

---

## 八、开发注意事项

### 1. Patch 机制使用

```assembly
; 正确使用 bpatch 指令
bpatch patch00_0,mem_patch00    ; 设置补丁映射

; mem_patchXX 需要在初始化时正确配置
```

### 2. 唤醒锁管理

```assembly
; 睡眠前必须检查唤醒锁
call lpm_check_wake_lock
nbranch lpm_dispatch_sleep,blank

; 需要保持活跃时获取唤醒锁
call lpm_get_wake_lock

; 操作完成后释放
call lpm_put_wake_lock
```

### 3. 上下文切换

```assembly
; 多连接时切换上下文
call context_search_conn_handle
rtn zero                        ; 未找到连接
call context_load               ; 加载上下文

; 保存当前上下文
call context_save
```

### 4. 时序关键操作

- **Sniff 锚点计算**: `context_search_sniff`
- **时钟偏移校准**: `calc_clke` / `calc_clke2`
- **LPM 时钟恢复**: `lpm_recover_clk`

### 5. 内存对齐和边界

```assembly
; 注意 32 位操作对齐
iadd temp,timeup                ; 28 位时钟包装
deposit timeup
```

### 6. 射频校准时机

- 首次启动：`initialize_radio`
- LPM 唤醒后：`lpm_recover_clk`
- 定期校准：`lpo_calibration`

---

## 九、关键数据结构

### 连接上下文 (context)

| 偏移 | 字段 | 说明 |
|------|------|------|
| `coffset_mode` | mode | 连接模式 (LE/Master/Slave) |
| `coffset_plap` | plap | Periodic Advertising LAP |
| `coffset_conn_handle` | handle | 连接句柄 |
| `coffset_sniff_anchor` | anchor | Sniff 锚点 (4 字节) |
| `coffset_tsniff` | t_sniff | Sniff 间隔 |
| `coffset_rx_window` | rx_window | 接收窗口 |
| `coffset_clk_offset` | clk_offset | 时钟偏移 (6 字节) |

### LPM 相关

| 变量 | 说明 |
|------|------|
| `mem_lpm_wake_lock` | 唤醒锁位图 |
| `mem_lpm_mode` | LPM 模式选择 |
| `mem_lpm_interval` | LPM 间隔 |
| `mem_lpm_mult` | 多事件倍数 |
| `mem_sleep_counter` | 睡眠计数器 (LPO 周期) |
| `mem_clks_per_lpo` | LPO 校准值 |

---

## 十、调试建议

1. **监控唤醒锁**: 检查 `mem_lpm_wake_lock` 位图
2. **跟踪上下文**: 监控 `mem_current_context`
3. **RF 校准验证**: 检查 `mem_aac_res_table` / `mem_ghpc_table`
4. **Sniff 同步**: 监控 `mem_sniff_rcv` / `mem_sniff_lost`
5. **LPM 决策**: 跟踪 `lpm_dispatch` 执行路径

---

## 附录：编译选项

```assembly
define SECURE_CONNECTION     ; 安全连接支持
define NEC                   ; NEC 遥控
define COMPILE_SHUTTER       ; 快门功能
define COMPILE_MOUSE         ; 鼠标功能
define COMPILE_MODULE        ; 模块功能
define COMPILE_USB           ; USB 功能
define COMPILE_DONGLE        ; 加密狗功能
define COMPILE_LE            ; 低功耗蓝牙
define COMPILE_24G           ; 2.4G 私有协议
define COMPILE_CAR           ; 车控功能
define COMPILE_REMOTE_CAR    ; 远程车控
define COMPLIE_ADPCM         ; ADPCM 音频
```
