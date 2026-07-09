
## 结构体概述

`c51_global_variable` 结构体（在代码中定义为 `G_VARIABLE_MAP` 类型）是项目的核心数据结构，包含了系统的各种状态和配置信息。该结构体位于 `global_variable.h` 文件中，用于管理键盘系统的所有重要状态。

## 变量详细解释

### 1. 系统配置与状态

| 变量名 | 地址 | 含义 |
|-------|------|------|
| `update_eeprom_flag` | 0x4cbf | 标记是否需要更新 EEPROM 数据，用于持久化存储配置 |
| `last_device_num` | 0x4cc0 | 上次使用的设备编号，用于快速恢复上次连接 |
| `system_mode` | 0x4cc1 | 当前系统模式（Windows、Mac、iOS、Android） |
| `power_on_action` | 0x4cc2 | 开机时的动作状态 |
| `current_device_num` | 0x4cc3 | 当前激活的设备编号 |
| `temp_device_num` | 0x4cc4 | 临时设备编号，用于设备切换过程中 |

### 2. 连接状态管理

| 变量名 | 地址 | 含义 |
|-------|------|------|
| `g24_currentState` | 0x4cc5 | 2.4G 连接的当前主状态 |
| `g24_currentSubState` | 0x4cc6 | 2.4G 连接的当前子状态 |
| `ble_currentState` | 0x4cc7 | BLE（低功耗蓝牙）连接的当前主状态 |
| `ble_currentSubState` | 0x4cc8 | BLE 连接的当前子状态 |
| `br_currentState` | 0x4cc9 | BR/EDR（经典蓝牙）连接的当前主状态 |
| `br_currentSubState` | 0x4cca | BR/EDR 连接的当前子状态 |
| `history_state` | 0x4ccb | 历史状态，用于状态恢复 |

### 3. 按钮控制

| 变量名                     | 地址            | 含义                  |
| ----------------------- | ------------- | ------------------- |
| `button_reconnect_flag` | 0x4ccc        | 按钮重连标志，用于长按键进入重连模式  |
| `button_24G_rec_flag`   | 0x4ccd        | 2.4G 设备重连按钮标志       |
| `button_flag`           | 0x4cce        | 按钮状态标志，记录各种按钮事件     |
| `button_timer[4]`       | 0x4ccf~0x4cd2 | 按钮定时器数组，用于处理按钮长按等事件 |

### 4. 电池管理

| 变量名 | 地址 | 含义 |
|-------|------|------|
| `battery_check_interval` | 0x4cd3 | 电池检查间隔时间 |
| `battery_value_index` | 0x4cd4 | 电池值数组索引 |
| `battery_status` | 0x4cd5 | 电池状态（正常、低电量、需要充电等） |
| `battery_value[BAT_ARRAY_LEN]` | 0x4cd6~0x4cdd | 电池电压值数组，用于平均计算 |
| `battery_level_low` | 0x4cde | 电池低电量阈值 |
| `battery_level_shutdown` | 0x4ce0 | 电池关机阈值 |
| `battery_low_led_flash_interval` | 0x4ce2 | 电池低电量时 LED 闪烁间隔 |
| `battery_low_led_flash_flag` | 0x4cf7 | 电池低电量 LED 闪烁标志 |
| `battery_level_percentage` | 0x4cf9 | 电池电量百分比 |
| `battery_level_full` | 0x4cfa | 电池满电阈值 |
| `battery_shutdown_flag` | - | 电池关机标志 |
| `last_battery_status` | - | 上次电池状态 |

### 5. 定时器管理

| 变量名 | 地址 | 含义 |
|-------|------|------|
| `sleepTimer` | 0x4ce3 | 睡眠定时器，用于进入低功耗模式 |
| `powerOn_timer` | 0x4ce5 | 开机定时器，用于处理开机相关逻辑 |
| `sys_numlockled_on_timer` | 0x4ce6 | Num Lock LED 开启定时器 |
| `sys_capslockled_on_timer` | 0x4ce7 | Caps Lock LED 开启定时器 |
| `sys_scrolllockled_on_timer` | 0x4ce8 | Scroll Lock LED 开启定时器 |
| `pairing_timeout` | 0x4ce9 | 配对超时定时器 |
| `delay_enter_lpm_timer` | 0x4cea | 延迟进入低功耗模式定时器 |
| `one_key_press_wait_release_timer` | 0x4cf8 | 单键按下等待释放定时器 |

### 6. 连接与重连管理

| 变量名 | 地址 | 含义 |
|-------|------|------|
| `pairing_g24_timeout` | 0x4cee | 2.4G 配对超时定时器 |
| `recon_delay` | 0x4cef | 重连延迟时间 |
| `recon_count` | 0x4cf0 | 重连尝试次数 |
| `recon_continue` | 0x4cf1 | 重连继续标志 |
| `ble_ramdon_lap1` | 0x4cf2 | BLE 随机 LAP1 值 |
| `ble_ramdon_lap2` | 0x4cf3 | BLE 随机 LAP2 值 |
| `recon_index` | - | 重连索引 |

### 7. 按键处理

| 变量名                        | 地址     | 含义                                                             |
| -------------------------- | ------ | -------------------------------------------------------------- |
| `release_data`             | 0x4ceb | 指示设备是否处于"释放/断开"状态 ：<br>1 ：设备已断开，需要重新连接或进入配对模式<br>0 ：设备处于正常连接状态 |
| `key_action`               | 0x4ced | 按键动作类型                                                         |
| `key_combination_step`     | 0x4cf4 | 按键组合步骤                                                         |
| `key_combination_ctrl`     | 0x4cf5 | 按键组合控制键                                                        |
| `key_combination_keyvalue` | 0x4cf6 | 按键组合键值                                                         |

### 8. 低功耗管理

| 变量名 | 地址 | 含义 |
|-------|------|------|
| `lockLpm` | 0x4cec | 锁定低功耗模式标志 |
| `g24_long_sleep_flag` | - | 2.4G 长睡眠标志 |

### 9. 传感器与鼠标数据

| 变量名 | 地址 | 含义 |
|-------|------|------|
| `mouse_data_send_flag` | - | 鼠标数据发送标志 |
| `sensor_key` | - | 传感器按键状态 |
| `sensor_x_l` | - | 传感器 X 坐标低字节 |
| `sensor_x_h` | - | 传感器 X 坐标高字节 |
| `sensor_y_l` | - | 传感器 Y 坐标低字节 |
| `sensor_y_h` | - | 传感器 Y 坐标高字节 |
| `sensor_wheel` | - | 传感器滚轮数据 |
| `sensor_titl` | - | 传感器倾斜数据 |

### 10. 其他系统变量

| 变量名                      | 地址  | 含义            |
| ------------------------ | --- | ------------- |
| `fast_connect_send_name` | -   | 快速连接时发送设备名称标志 |
| `system_mode_last`       | -   | 上次系统模式        |
| `connect_button_temp`    | -   | 连接按钮临时值       |
| `test_buff`              | -   | 测试缓冲区         |
|                          |     |               |

# button_flag 的各个位定义如下：
### 标志位详解

####  1. KEY_FLAG_BTKEY_PRESS (0x01)
- 含义 ：蓝牙按键按下标志
- 使用场景 ：当用户按下蓝牙相关按键时设置此标志
- 检测方式 ： if (g_variable.button_flag & KEY_FLAG_BTKEY_PRESS)
#### 2. KEY_FLAG_SAME_KEY_PRESS (0x02)
- 含义 ：经过双重检测验证的相同按键按下标志
- 使用场景 ：当两次键盘扫描结果一致且有按键按下时设置此标志
- 检测方式 ： if (g_variable.button_flag & KEY_FLAG_SAME_KEY_PRESS)
#### 3. KEY_FLAG_STOP_DISCOVERY (0x04)
- 含义 ：停止发现标志
- 使用场景 ：可能用于停止蓝牙设备的发现过程
- 检测方式 ： if (g_variable.button_flag & KEY_FLAG_STOP_DISCOVERY)
#### 4. KEY_FLAG_FN_DEVICE_BUTTON (0x08)
- 含义 ：FN设备按键标志
- 使用场景 ：当按下FN+设备切换按键时设置此标志
- 检测方式 ： if (g_variable.button_flag & KEY_FLAG_FN_DEVICE_BUTTON)
#### 5. KEY_FLAG_FN_24G_DEVICE_BUTTON (0x10)
- 含义 ：FN 24G设备按键标志
- 使用场景 ：当按下FN+24G设备切换按键时设置此标志
- 检测方式 ： if (g_variable.button_flag & KEY_FLAG_FN_24G_DEVICE_BUTTON)
#### 6. KEY_FLAG_FN_SYSTEM_MODE_BUTTON (0x20)
- 含义 ：FN系统模式按键标志
- 使用场景 ：当按下FN+系统模式切换按键时设置此标志
- 检测方式 ： if (g_variable.button_flag & KEY_FLAG_FN_SYSTEM_MODE_BUTTON)
## 技术要点
位掩码操作 ：
   - 使用位掩码可以在一个字节中存储多个状态标志
   - 通过位或操作 |= 设置标志位
   - 通过位与操作 &= 清除标志位
标志位组合 ：
   - 可以同时设置多个标志位，例如： g_variable.button_flag |= KEY_FLAG_BTKEY_PRESS | KEY_FLAG_SAME_KEY_PRESS;
   - 可以同时检查多个标志位，例如： if (g_variable.button_flag & (KEY_FLAG_BTKEY_PRESS | KEY_FLAG_SAME_KEY_PRESS))
标志位管理 ：
   - 及时设置和清除标志位，确保系统状态的准确性
   - 在适当的时机检查标志位，执行相应的操作

## 技术实现细节

### 内存布局

`c51_global_variable` 结构体的变量都有固定的内存地址，这是为了：
1. **内存管理**：确保变量在内存中有固定位置
2. **调试方便**：可以通过内存地址直接访问变量
3. **性能优化**：减少内存访问的开销

### 状态管理

系统采用了主状态和子状态的分层设计：
- **主状态**：表示系统的宏观状态，如连接、断开等
- **子状态**：表示主状态下的具体场景，如配对中的PIN码输入等

这种设计使得状态管理更加清晰，代码逻辑更加模块化。

### 低功耗优化

结构体中包含多个与低功耗相关的变量：
- `sleepTimer`：控制进入睡眠模式的时间
- `lockLpm`：锁定低功耗模式
- `g24_long_sleep_flag`：2.4G 长睡眠标志

这些变量共同作用，确保系统在空闲时能够进入低功耗状态，延长电池寿命。

### 电池管理

系统实现了完善的电池管理机制：
- 定期检查电池电压
- 计算电池电量百分比
- 低电量提醒
- 电池保护（低电量关机）

## 总结

`c51_global_variable` 结构体是一个综合性的数据结构，包含了键盘系统的各个方面：
- 设备连接状态管理
- 电池状态监控
- 按键处理
- 系统配置
- 低功耗管理
- 传感器数据处理

这些变量共同构成了键盘系统的状态机，确保设备能够正确响应用户操作，管理连接状态，并在各种条件下正常工作。每个变量都有其特定的用途，它们相互配合，使整个系统能够协调运行。

通过对这个结构体的分析，我们可以看到键盘固件的设计思路和实现细节，这对于理解整个系统的工作原理非常有帮助。