# YiChip BLE 编译系统 - Perl 处理详解

## 概述

YiChipBLE 芯片的编译系统使用 Perl 脚本进行代码预处理、内存分配、Patch 表生成和 ROM 文件处理。主要脚本位于 `util/` 目录下。

---

## 脚本文件

| 脚本 | 功能 |
|------|------|
| `util/mergepatch.pl` | 主处理脚本：条件编译、内存分配、Patch 表、OTP 生成 |
| `util/romcrc.pl` | CRC-16-CCITT 校验码计算 |

---

## do.bat 中 Perl 调用流程

```batch
:: 1. 主处理（条件编译、内存分配、Patch 表）
perl util/mergepatch.pl

:: 2. 生成认证 ROM（合并多个 .dat 文件）
perl util/mergepatch.pl mouse_ble_att_list usb_kbdata_vendor_define ...

:: 3. 添加 CRC 校验
perl util/romcrc.pl romcode.rom

:: 4. 生成 OTP 文件
perl util/mergepatch.pl otp
```

---

## mergepatch.pl 处理详解

### 输入文件

| 文件 | 来源 | 说明 |
|------|------|------|
| `output/bt_program23.meta` | `type program\*.prog` 合并 | 程序代码中间表示（汇编） |
| `output/bt_format.meta` | `type format\*.format` 合并 | 内存布局定义 |
| `output/sched.rom` | `copy sched/*.dat` 合并 | 调度配置 ROM |
| `sched/*.dat` | 配置文件 | 设备特定配置数据 |

---

### 处理步骤 1：条件编译处理 (`parseif`)

**功能**：处理 `ifdef/ifndef/else/endif/include/define` 指令

**源码位置**：`mergepatch.pl` 第 130-197 行

**处理逻辑**：
```perl
sub parseif {
    my($fname) = @_;
    @valid = (1);  # 条件栈初始化为真

    for each line in file {
        # 移除 /* ... */ 注释

        if (ifdef XXX) {
            # 检查 XXX 是否在 $defs 中定义
            push @valid, (defined ? 1 : 0) & $valid[top];
        }
        elsif (ifndef XXX) {
            # 检查 XXX 是否未定义
            push @valid, (defined ? 0 : 1) & $valid[top];
        }
        elsif (else) {
            # 反转当前条件
            $valid[top] = (1 - $valid[top]) & $valid[top-1];
        }
        elsif (endif) {
            pop @valid;  # 弹出条件栈
        }
        elsif (include file) {
            # 嵌入 program/file 的内容
        }
        elsif (define XXX) {
            # 记录宏定义
            $defs .= "XXX ";
        }
        elsif ($valid[top]) {
            # 只输出条件为真的行
            print line;
        }
    }
}
```

**输入示例**：
```perl
define SECURE_CONNECTION
define COMPILE_MODULE

ifdef SECURE_CONNECTION
    branch p_secure_connect
endif

ifndef SIM
    include patch.prog
endif
```

**输出示例**：
```perl
branch p_secure_connect
[patch.prog 的内容被嵌入]
```

**当前定义的宏**（根据 bt_program23.meta）：
```
SECURE_CONNECTION
COMPILE_SHUTTER
COMPILE_MOUSE
COMPILE_MODULE
COMPILE_USB
COMPILE_DONGLE
COMPILE_LE
COMPILE_24G
COMPILE_CAR
COMPILE_REMOTE_CAR
COMPLIE_ADPCM
NEC
DEBUG_RF_INIT
```

---

### 处理步骤 2：内存分配 (`malloc`)

**功能**：解析 `memalloc/xmemalloc/amemalloc/omemalloc` 块，分配内存地址

**源码位置**：`mergepatch.pl` 第 35-127 行

**内存区域**：
| 区域 | 起始地址 | 指令 |
|------|----------|------|
| 低地址区 | 0x0000 | `memalloc()` |
| 高地址区 | 0x4000 | `xmemalloc()` |
| 隔离区 | 0x4000+ | `omemalloc()` |
| 相对分配 | 基于基地址 | `amemalloc base_var()` |

**处理逻辑**：
```perl
sub malloc {
    $addr = 0;       # 低地址区指针
    $xaddr = 0x4000; # 高地址区指针

    for each line in bt_format.meta {
        if (memalloc()) {
            # 从 0 开始分配
            $addr += size;
            $bstr .= "0x%04x varname\n";
        }
        elsif (xmemalloc()) {
            # 从 0x4000 开始分配
            $xaddr += size;
            $xstr .= "0x%04x varname\n";
        }
        elsif (omemalloc()) {
            # 隔离分配（已废弃，兼容用）
            push @{$xmalloc{$ocnt}}, @data;
        }
        elsif (amemalloc base_var) {
            # 基于 base_var 地址继续分配
            push @{$malloc{$basev}}, @data;
        }
    }

    # 输出处理后的文件
    print file $bstr, $xstr, $sstr;
    print file1 $bstr, $xstr;  # memmap.format

    printf "Last allocated address is %04x\n", $bend;
    printf "Last allocated xmem address is %04x\n", $xend;
}
```

**输入示例**（bt_format.meta）：
```perl
// 从地址 0 开始分配
memalloc(
    10 mem_var_a      # 分配 10 字节
    20 mem_var_b      # 分配 20 字节
)

// 从地址 0x4000 开始分配 (高地址区)
xmemalloc(
    100 mem_le_buf    # 分配 100 字节
)

// 相互隔离的分配（已废弃，兼容用）
omemalloc(
    32 mem_le_tx_buffer0_omemalloc
    32 mem_le_tx_buffer1_omemalloc
)

// amemalloc - 基于已有变量地址继续分配
amemalloc mem_le_update_new_param(
    8 new_param_data
)
```

**输出示例**（memmap.format）：
```
0x0000 mem_le_adv_transmit
0x0001 mem_le_adv_waitcnt
0x0002 mem_le_adv_rcv
...
0x0842 last_var
0x4000 mem_le_buf
0x4064 mem_another_var
```

**实际输出统计**：
```
Last allocated address is 0842
Last allocated xmem address is 4e80
```

---

### 处理步骤 3：Patch 表生成 (`genpatch`)

**功能**：从代码中的 `beq patchXX_Y,` 指令提取 patch 位图，插入到 sched.rom

**源码位置**：`mergepatch.pl` 第 199-235 行

**Patch 指令格式**：
```
patch[00-3f]_[0-7]
- XX (00-3f): Patch 寄存器索引 (0-63)
- Y (0-7): 位索引 (0-7)
```

**处理逻辑**：
```perl
sub genpatch {
    # 1. 扫描所有 beq patchXX_Y 指令
    while (<bt_program23.meta>) {
        if (/beq\s+patch([0-9a-f]+)_([0-7]),/) {
            $a = hex($1);  # 寄存器索引
            $b = hex($2);  # 位索引
            $bits[$a] |= 1 << $b;  # 设置对应位
        }
    }

    # 2. 生成 64 字节的 patch 表
    for($j = 0; $j < 0x40; $j++) {
        $s .= sprintf("%02x   #mem_patch%02x\n", $bits[$j], $j);
    }

    # 3. 插入到 sched.rom 的 mem_patch00: 标签处
    if (found mem_patch00:) {
        splice(@sched, $skip, $count, $s);
    } else {
        # 添加 mem_patch00: 标签
        splice(@sched, 0, 0, "mem_patch00:\n" . $s);
    }

    print file @sched;
}
```

**输入示例**（bt_program23.meta）：
```perl
beq patch00_0, p_soft_reset
beq patch02_1, p_set_sync_on
beq patch02_4, p_set_lemode
beq patch02_5, p_rf_rx_enable
beq patch24_4, p_le_receive_rxon
beq patch24_5, p_le_rx_dec
beq patch24_6, p_le_rx_nopayload
```

**输出示例**（sched.rom）：
```
mem_patch00:
01   #mem_patch00    # 00000001 → bit0 有效 (patch00_0)
00   #mem_patch01    # 00000000 → 无有效位
72   #mem_patch02    # 01110010 → bit1,2,4,5 有效
...
70   #mem_patch24    # 01110000 → bit4,5,6 有效
```

---

### 处理步骤 4：跨区代码处理 (`zcode`)

**功能**：处理超过 100 字节偏移的跳转，添加中转桩代码

**源码位置**：`mergepatch.pl` 第 237-294 行

**处理逻辑**：
```perl
sub zcode {
    # 1. 扫描 org z 标签，记录 Z 区代码位置
    if (/org z/) {
        $line[$z/0x10000 + 1] = $i - 1;
        $z += 0x10000;
    }

    # 2. 收集所有跳转标签
    if (/^(\w+):\s*$/) {
        $lab{$1} = $z;
    }

    # 3. 处理跨区跳转
    for each line {
        if (branch/call/beq/bne label) {
            if ($lab{$label} > 100) {  # 偏移超过 100
                $nlabel = "jmpz_" . $label;
                # 替换标签
                $line =~ s/$label/$nlabel/;

                # 添加中转桩代码
                if (branch instruction) {
                    $line = "setarg 0x%x\nbranch p_zcode_entrance_2Bytes_common";
                } else {
                    # 在目标位置添加桩
                    $lab{$label} .= sprintf("%s:\n\tsetarg 0x%x\n\tbranch ...",
                                            $nlabel, $z);
                }
            }
        }
    }

    # 4. 添加 patch 入口点
    if (/bbit1 8,pf_patch_ext/) {
        print file "p_start:\n\tbranch p_patch_array\n\np_zcode:\n";
        for($j = 0; $j < 63; $j++) {
            print file "\tnop %d\n", $j + 1;
        }
        print file "p_patch_array:\n";
    }
}
```

---

### 处理步骤 5：认证 ROM 生成 (`authrom`)

**功能**：合并多个 .dat 文件到 auth.rom，并嵌入到 romcode.rom 末尾

**源码位置**：`mergepatch.pl` 第 296-324 行

**调用方式**：
```batch
perl util/mergepatch.pl mouse_ble_att_list usb_kbdata_vendor_define usb_kbdata usb_msdata usb_devicedata usb_confdata ble_shutter_gatt_list ble_shutter_key_value_list ble_car_att_list sha256
```

**处理逻辑**：
```perl
sub authrom {
    $addr = 0x9000;  # 认证数据起始地址

    # 1. 读取所有 .dat 文件参数
    foreach $s (@ARGV) {
        open f, "../sched/" . $s . ".dat";
        @ff = <f>;
        close f;

        # 移除空白字符
        foreach (@ff) {
            $_ =~ s/\s//g;
            push @auth, $_ . "\n";
        }
        $addr += $#ff + 1;
    }

    # 2. 写入 auth.rom
    open f, ">auth.rom";
    print f @auth;
    close f;

    # 3. 将 auth.rom 嵌入 romcode.rom 末尾 (0x200 字节)
    open f, "romcode.rom";
    @rom = <f>;
    close f;

    for($i = 0, $j = $#rom - 0x1ff; $i < 0x200; $i++, $j++) {
        # 4 字节打包成 32 位字 (小端序)
        for($k = 0, $l = ""; $k < 4; $k++) {
            $_ = $auth[$i*4 + $k];
            s/\s//g;
            $_ = "00" if(/^$/);
            $l = $_ . $l;  # 小端序
        }
        $rom[$j] = $l . "\n";
    }

    open f, ">romcode.rom";
    print f @rom;
    close f;
}
```

**处理的 .dat 文件**：
| 文件 | 说明 |
|------|------|
| `mouse_ble_att_list.dat` | 鼠标 BLE 属性表 |
| `usb_kbdata_vendor_define.dat` | USB 键盘数据（厂商定义） |
| `usb_kbdata.dat` | USB 键盘数据 |
| `usb_msdata.dat` | USB 鼠标数据 |
| `usb_devicedata.dat` | USB 设备数据 |
| `usb_confdata.dat` | USB 配置数据 |
| `ble_shutter_gatt_list.dat` | BLE 快门 GATT 表 |
| `ble_shutter_key_value_list.dat` | BLE 快门键值表 |
| `ble_car_att_list.dat` | BLE 汽车属性表 |
| `sha256.dat` | SHA256 加密数据 |

---

### 处理步骤 6：OTP 生成 (`otp`)

**功能**：合并 eeprom.dat 和 otp_set.dat 生成 otp.dat

**源码位置**：`mergepatch.pl` 第 326-337 行

**调用方式**：
```batch
perl util/mergepatch.pl otp
```

**处理逻辑**：
```perl
sub otp {
    # 1. 读取 eeprom.dat
    open f, 'eeprom.dat';
    @a = <f>;
    close f;

    # 2. 读取 otp_set.dat
    open f, '../sched/otp_set.dat';
    @b = <f>;
    close f;

    # 3. 生成 otp.dat = otp_set.dat + (eeprom.dat - 前两行)
    open f, '>otp.dat';
    print f @b;           # 先写入 otp_set.dat
    splice(@a, 0, 2);     # 去掉 eeprom.dat 前两行
    print f @a;           # 再写入剩余部分
    close f;
}
```

---

## romcrc.pl 处理详解

**功能**：为 ROM 文件添加 CRC-16-CCITT 校验码

**源码位置**：`util/romcrc.pl`

**CRC 算法**：
```perl
sub crc16_ccitt2 {
    my($crc, $c) = @_;

    $crc  = ($crc >> 8) | ($crc << 8);
    $crc ^= $c;
    $crc ^= ($crc & 0xff) >> 4;
    $crc ^= $crc << 12;
    $crc ^= ($crc & 0xff) << 5;
    $crc &= 0xffff;
    return $crc;
}

sub gencrc {
    my($crc, $c) = @_;
    $c =~ s/\s//g;
    for($i = 0; $i < length($c); $i += 2) {
        $crc = crc16_ccitt2($crc, hex(substr($c, $i, 2)));
    }
    return $crc;
}
```

**处理逻辑**：
```perl
# 1. 读取 ROM 文件
open f, "$ARGV[0]";
@txt = <f>;
close f;

$len = $#txt;
$len = hex($ARGV[1]) if(@ARGV > 1);  # 可选：指定长度

# 2. 计算 CRC
for($i = 0, $crc = 0xffff; $i < $len; $i++) {
    $_ = $txt[$i];
    s/\s//g;
    $crc = gencrc($crc, $_);
}

# 3. 在文件末尾追加 CRC（2 字节）
if($wid > 4) {
    $txt[$len] = sprintf("%04x", $crc);
} else {
    $txt[$len] .= sprintf("%02x\n%02x\n", $crc >> 8, $crc & 0xff);
}

# 4. 写回文件
open f, ">$ARGV[0]";
print f @txt;
close f;

printf "%02x\n%02x\n", $crc >> 8, $crc & 0xff;
```

**调用方式**：
```batch
perl util/romcrc.pl romcode.rom
```

**输出示例**：
```
e7  # CRC 高字节
a3  # CRC 低字节
```

---

## 完整数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                        原始输入                                  │
├─────────────────────────────────────────────────────────────────┤
│  program/*.prog ──┐                                             │
│  format/*.format ─┼─→ do.bat (type 合并) ──→ .meta 文件         │
│  sched/*.dat ─────┘                                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Perl 处理 1 (mergepatch.pl)                   │
├─────────────────────────────────────────────────────────────────┤
│  1. parseif("bt_program23.meta")  → 条件编译处理                │
│  2. parseif("bt_format.meta")     → 条件编译处理                │
│  3. genpatch()                    → Patch 表生成 (sched.rom)     │
│  4. malloc()                      → 内存分配 (memmap.format)     │
│  5. zcode()                       → 跨区代码处理                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Perl 处理 2 (mergepatch.pl authrom)                │
├─────────────────────────────────────────────────────────────────┤
│  输入：sched/*.dat (10 个认证相关文件)                            │
│  输出：output/auth.rom                                          │
│  嵌入：romcode.rom 末尾 0x200 字节                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                Perl 处理 3 (romcrc.pl)                          │
├─────────────────────────────────────────────────────────────────┤
│  输入：romcode.rom                                              │
│  处理：CRC-16-CCITT 计算                                         │
│  输出：romcode.rom (末尾追加 2 字节 CRC)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                Perl 处理 4 (mergepatch.pl otp)                  │
├─────────────────────────────────────────────────────────────────┤
│  输入：eeprom.dat + sched/otp_set.dat                           │
│  输出：output/otp.dat                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        最终输出                                  │
├─────────────────────────────────────────────────────────────────┤
│  romcode.rom      - 可烧录的 ROM 镜像（含 CRC）                    │
│  sched.rom        - 调度配置 ROM（含 Patch 表）                   │
│  otp.dat          - OTP 烧录文件                                 │
│  eeprom.dat       - EEPROM 配置                                  │
│  memmap.format    - 内存映射表                                   │
│  auth.rom         - 认证数据 ROM                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键文件说明

### output/bt_program23.meta
- **来源**：`program/bt.prog` + 所有 `program/*.prog` 合并
- **内容**：汇编代码，包含 patch 指令、条件编译指令
- **处理后**：条件编译被移除，只保留有效代码

### output/bt_format.meta
- **来源**：`format/bt.format` + 所有 `format/*.format` 合并
- **内容**：内存布局定义，包含 memalloc/xmemalloc 块
- **处理后**：添加内存地址，生成 memmap.format

### output/memmap.format
- **来源**：malloc() 函数生成
- **内容**：所有变量的内存地址映射
- **格式**：`0x0000 varname`

### output/sched.rom
- **来源**：`sched/*.dat` 合并 + genpatch() 插入 Patch 表
- **内容**：设备配置参数 + 64 字节 Patch 表

### output/romcode.rom
- **来源**：汇编器输出 + authrom() 嵌入认证数据 + romcrc.pl 添加 CRC
- **内容**：最终可烧录的 ROM 镜像

---

## 常见问题

### Q: 为什么需要条件编译？
A: 同一份代码需要支持多种设备类型（鼠标、键盘、快门等），通过 define 宏控制编译哪些功能模块。

### Q: Patch 表的作用是什么？
A: Patch 表用于在运行时动态启用/禁用某些补丁功能。每个 bit 对应一个 patch 函数入口。

### Q: memalloc 和 xmemalloc 有什么区别？
A: memalloc 从地址 0 开始分配（低地址区），xmemalloc 从地址 0x4000 开始分配（高地址区）。这是为了将不同类型的变量分开存放。

### Q: OTP 是什么？
A: OTP (One-Time Programmable) 是一次性可编程存储器，用于存储芯片的固定配置参数。otp.dat 文件用于烧录到 OTP 区域。

---

## 参考资料

- `util/mergepatch.pl` - 主处理脚本
- `util/romcrc.pl` - CRC 计算脚本
- `do.bat` - 编译主脚本
- `format/*.format` - 内存布局定义文件
- `program/*.prog` - 程序源代码文件
- `sched/*.dat` - 设备配置文件
