# YiChip 开发脚本分析

本文档分析了项目目录下所有 `.bat` 批处理脚本的功能和用途。

---

## 脚本概览

| 文件名             | 主要用途         |
| --------------- | ------------ |
| `fp.bat`        | Flash烧录脚本    |
| `a.bat`         | RAM 烧录脚本     |
| `do.bat`        | 代码编译脚本       |
| `eotp.bat`      | OTP ROM 烧录脚本 |
| `ep.bat`        | EEROM烧录脚本    |
| `dump_gpio.bat` | 打印GPIO 状态脚本  |

---

## 详细分析

### 1. do.bat - 主构建脚本

**核心功能**：编译和生成完整的蓝牙设备固件

**支持的_device_option_配置**：
- `shutter` - 快门遥控器
- `shutter_dy` - 动态快门
- `mouse` - 鼠标
- `keyboard` - 键盘
- `hci` - HCI 模式
- `dongle` - 接收器
- `car` - 车控
- `remote_car` - 远程车控
- `module` - 模块模式
- `antilost` - 防丢器
- `mesh` - Mesh 网络
- `otp` - OTP 模式
- `flippen` - 翻转模式

**构建流程**：

1. **动态补丁构建**
   ```batch
   set patch_sources=program\patch_base.prog
   for /f "delims=" %%f in ('dir /b /o /s program\my_feature\*.prog') do (
       set patch_sources=!patch_sources! "%%f"
   )
   type %patch_sources% > program\patch.prog
   ```

2. **协议栈合并**
   - 蓝牙协议栈 (`ble_protocol_stack/*.prog`)
   - G24 协议栈 (`g24_protocol_stack/*.prog`)
   - Mesh 协议栈 (`mesh_protocol_stack/*.prog`)
   - 主程序 (`program/*.prog`)
   - 生成 `output/bt_program23.meta`

3. **格式文件合并**
   - 合并所有 `.format` 文件
   - 生成 `output/bt_format.meta`

4. **调度 ROM 生成**
   - 根据设备类型选择对应的调度文件
   - 生成 `output/sched.rom`

5. **最终编译**
   - `osiuasm` - 汇编编译
   - `geneep` - 生成 EEPROM 数据
   - `romcrc.pl` - 计算 CRC 校验
   - 生成最终的 `romcode.rom`

---

### 2. eotp.bat - OTP 烧录脚本

**功能**：将配置烧录到芯片的 OTP（一次性可编程）存储器

**执行步骤**：
```batch
call do.bat eep          # 初始化 EEPROM
set baud=a0              # 设置波特率
e pu                     # 上电
e 8043 00                # 配置寄存器
e otr 0 40               # OTP 相关配置
e otp output/otp.dat     # 烧录 OTP 数据
e otr 0 40               # 验证配置
e ku                     # 断电
```

---

### 3. ep.bat - EEPROM 烧录脚本

**功能**：烧录数据到 EEPROM 存储器

**执行步骤**：
```batch
call do.bat eep          # 初始化
e pu                     # 上电
e 8043 00                # 配置寄存器

# 解除写保护（IO1 为 E2P 写保护引脚）
e 8071 3e                # 推挽下拉，关闭写保护
e er 0 10                # 擦除 EEPROM
e ew 0 0000              # 写入数据
e er 0 10                # 验证擦除
e ep ./output/eeprom.dat # 导出 EEPROM 数据
e ku                     # 断电
e au                     # 自动检测
```

---

### 4. a.bat - ROM 烧录脚本

**功能**：烧录 RAM 代码

**执行步骤**：
```batch
e ku              # 断电
e pu              # 上电
e 8043 00         # 配置寄存器
e pu              # 上电
e hu output/ramcode.rom 0  # 加载 RAM 代码
e pu              # 上电
e su output/sched.rom      # 加载调度代码
e cu              # 配置完成
```

---

### 5. fp.bat - Flash 编程脚本

**功能**：擦除和编程芯片内部 Flash 存储器

**执行步骤**：
```batch
e pu                    # 上电
e 8043 00               # 配置寄存器
e 8070 0000000000000000 # 配置 GPIO
e 8078 00000000         # 配置寄存器
e 8077 20               # Flash 控制寄存器
e 8078 1f               # Flash 配置
e 807f 22               # Flash 模式
e 807e 21               # Flash 时序
e fc                    # Flash 擦除
e fr 0 20               # 读取 Flash
e fp                    # Flash 编程
e fr 0 20               # 验证读取
e ku                    # 断电
e au                    # 自动检测
```

---

### 6. dump_gpio.bat - GPIO 配置导出脚本

**功能**：调用 Python 脚本导出当前 GPIO 配置信息

**脚本内容**：
```batch
@echo off
set SCRIPT_PATH=%~dp0..\..\..\Agent\script\dump_gpio.py
python "%SCRIPT_PATH%"
```

---

## 工具链依赖

构建过程需要以下外部工具：

| 工具 | 用途 |
|------|------|
| `osiuasm` | 汇编编译器 |
| `geneep` | EEPROM 数据生成 |
| `perl` | 脚本解释器（运行 mergepatch.pl, romcrc.pl） |
| `python` | GPIO 配置导出 |
| `eeprom2fulleeprom.exe` | EEPROM 格式转换 |
| `crc16.exe` | CRC 校验计算 |

---

## 典型工作流程

1. **开发阶段**：修改 `program/` 下的 `.prog` 源文件
2. **编译代码**：运行 `do.bat` 生成 `romcode.rom`
3. **烧录代码**：连接烧录工具，运行`a.bat`将编译的代码写入芯片
4. **调试**：运行 `dump_gpio.bat` 查看 GPIO 配置

---

*文档生成时间：2026-03-16*
