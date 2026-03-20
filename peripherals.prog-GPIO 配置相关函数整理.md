# TEST/program/peripherals.prog 中 GPIO 配置相关函数整理

> 本文汇总 `TEST/program/peripherals.prog` 内所有与 GPIO 管脚配置、复用、低功耗相关的函数。根据代码语义分为 6 大类，并在每节列出函数职责、调用关键点与寄存器/内存侧写入位置，便于查阅与交叉定位。

## 1. 初始化/复用入口

- **`iicd_load_gpio_init` / `iicd_load_default_gpio` (`L389-L398`)**  
  - 处理 EEPROM SDA/SCL/WP 管脚号的装载。若 `mem_eeprom_sda_gpio` 与 `mem_eeprom_scl_gpio` 未设置则退回默认值 (WP=GPIO4, SCL=23, SDA=22)。  
  - 成功后统一跳转 `iicd_read_init_pin`，以写保护状态重映射 SCL/SDA 到 `gpcfg_iic_*`。
- **`iicd_init_pin` / `iicd_read_init_pin` / `iicd_init_pin_scl_sda` (`L1201-L1213`)**  
  - 写路径先 `iicd_eeprom_write_enable` 拉低 WP，再把 SCL/SDA 配置成 I²C 复用+上拉；读路径先 `iicd_eeprom_write_disable`，然后同样复用。  
  - 通过 `gpio_config_function_int` 写 `core_gpio_conf(_hi)`，保证 I²C DMA 前引脚已映射。
- **`spi_gpio_init` / `spi_gpio_default_init` (`L1137-L1156`)**  
  - 若未自定义 SPI 引脚则 `spi_gpio_default_init` 设定 CS=1、SI=3、SO=0、SCLK=2、WP=11、HOLD=10。  
  - 随后逐个调用 `gpio_config_function_int` 绑定到 `gpcfg_spid_*`，保障 SPI DMA/Flash 初始化使用的总线状态正确。
- **`pwm_gpio_select (+process)` (`L2802-L2809`)**  
  - 参数 `pdata` 高 8 位是 PWM 通道，低 8 位是 GPIO。  
  - `pwm_gpio_select_process` 通过 `gpcfg_pwm_out0 + channel` 计算复用值，再委托 `gpio_config_param` 写入，供 `pwm_enable/pwm_disable` 复用。
- **`ac_50hz_check` 注释模板 (`L3100-L3113`)**  
  - 展示 AC 检测需将 `mem_ac_detect_gpio` 绑定到 `gpcfg_ac_input`，同样复用路径依赖 `gpio_config_function_int`。

## 2. 唤醒/上下拉配置

- **`get_gpio_wakeup_index` (`L1989-L1998`)**  
  - 使用 `temp[4:0]` 计算 `mem_gpio_wakeup_cfg` 中的字节+高/低半字节掩码；偶数脚用 `0xF0`，奇数脚用 `0x0F`，供后续写入不互相破坏。
- **`gpio_set_wake_by_current_state` (`L2001-L2006`)**  
  - 暂时置位 `GPIO_ACTIVE_BIT`，调用 `gpio_get_bit` 采样实际电平，再回写极性位，避免设置唤醒前极性与硬件不一致。
- **`gpio_set_wake` / `gpio_set_wake_high` / `gpio_set_wake_low4bit` (`L2008-L2026`)**  
  - 校验 `UI_BUTTON_GPIO_DISABLE`，依据 `GPIO_ACTIVE_BIT` 选择高电平或低电平唤醒，向 nibble 写入 `debug` 值（低电平=2，高电平=4）。
- **`gpio_set_low_pullup` / `gpio_set_low_pullup_low4bit` (`L2029-L2043`)**  
  - 用 nibble 值 1 表示“低电平唤醒且需要上拉”，通常在进 LPM 前调用，保证空闲时脚被上拉但低脉冲仍能唤醒。
- **`gpio_clr_wake` (`L2048-L2055`)**  
  - 依据索引与掩码将 nibble 归零，实现快速禁用唤醒。`gpio_config_input_nowake` 先调用它再做输入配置。

## 3. 输入通路

- **`gpio_config_input` / `gpio_config_input_without_wake` (`L2063-L2073`)**  
  - 处理禁用脚判定，可选触发 `gpio_set_wake`，随后通过 `gpio_addr` 找到配置槽，清零后设置 bit6/bit7 以打开输入缓冲并按极性设置 `GPIO_ACTIVE_BIT`。  
  - 所有输入最终写回 `core_gpio_conf` 或 `_hi`，输出由 `gpio_write` 完成。
- **`gpio_get_bit` / `gpio_get_bit_reverse` (`L2076-L2090`)**  
  - 计算 `core_gpio_in` 字节地址，读取后根据 `GPIO_ACTIVE_BIT` 决定是否翻转。调用者拿到的永远是“逻辑 active”状态。

## 4. 输出与模拟控制

- **`gpio_out_inactive` / `gpio_out_active` / `gpio_out_flag` / `gpio_out` (`L2092-L2117`)**  
  - `gpio_out_active`=输出配置入口，`gpio_out_inactive` 用于驱动到“非激活”态；二者都校验禁用脚，设置或清除 `GPIO_ACTIVE_BIT`。  
  - `gpio_out` 将 `temp` 的 bit7 视为目标电平，再组合 `gpcfg_output_high` 写入实际寄存器，从而自动处理低电平有效的场景。
- **`gpio_check_active` (`L2119-L2130`)**  
  - 读取当前配置，看输出是否处于“激活电平”。常用于睡眠前判定是否需要翻转。
- **`gpio_set_analog` / `gpio_set_high_impedance` (`L2132-L2144`)**  
  - 通过 `gpcfg_no_ie` / `gpcfg_high_impedance` 关闭数字输入或输出，避免模拟测量/节能时的漏电。
- **`gpio_write` (`L2135-L2137`)**  
  - 所有配置最终汇聚到该函数，负责把 `pdata` 写入 `contw` 所指寄存器。

## 5. 寄存器寻址与复用基础

- **`gpio_addr` (+`_next`/`_ext`) (`L2146-L2157`)**  
  - `temp` 减去 `GPIO_NUMBER-1` 后区分低 16 脚 (`core_gpio_conf`) 与高脚 (`core_gpio_conf_hi`)，再乘以 2 得到偏移。
- **`gpio_config_param` (`L2160-L2163`)**  
  - 解析打包参数：低 8 位是 GPIO，高 8 位是功能号；拆解后直接调用 `gpio_config_function_int`。PWM/AC 等模块都复用此接口。
- **`gpio_config_function`/`gpio_config_function_int` (+ext/next) (`L2165-L2180`)**  
  - `gpio_config_function` 做极性检查，实际写操作在 `_int`：按 GPIO 号走低表或高表，`istoret 1,contw` 把功能号写到 mux 寄存器。  
  - `gpio_config_function_int_ext` 专门处理高编号脚，内部 `increase -16,pdata` 再复用同一循环。
- **`gpio_get_config` (+ext/next) (`L2183-L2196`)**  
  - 与 `gpio_addr` 逻辑对称，用于读取当前配置值（排查或调试）。

## 6. 低功耗准备

- **`gpio_set_before_lpm` / `_ext` (`L2199-L2223`)**  
  - 遍历 `core_gpio_conf` 以及 `_hi`，识别 SPI/I²C/纯输入等配置，将它们统统覆写为 `gpcfg_pullup`，确保进入休眠时管脚有已知电位。  
  - 需要所有配置都能正确寻址 (`contw` 指针)；若全表扫描完毕则比较 `contr` 与 `pdata` 以判断是否需要切换到高表。
- **`setgpio_pullup` / `setgpio_pulldown` (`L2225-L2232`)**  
  - 为 LPM 扫描提供的原子操作，分别将当前 `contw` 位置写为上拉/下拉后立即返回循环。

## 函数之间的依赖关系概览

1. **寻址层**：`gpio_addr` ↔ `core_gpio_conf(_hi)` 是所有配置函数的硬件入口；`gpio_write` 完成最终寄存器写入。  
2. **功能层**：`gpio_config_function_int`、`gpio_config_param`、`pwm_gpio_select_process` 等负责“把某个 GPIO 绑定到某个外设功能”。  
3. **行为层**：`gpio_set_wake*`、`gpio_config_input`、`gpio_out_*` 等在 mux 设置后进一步写入方向、上下拉和唤醒策略。  
4. **系统层**：`iicd_*`、`spi_gpio_init`、`pwm_enable*` 等高层模块在内部调用上述接口，确保驱动逻辑与物理引脚配置一致。

> 若在阅读过程中遇到汇编语义不清，可回到培训 PDF（`培训文档PDF版` 目录）检索指令含义。
