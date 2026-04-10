GET_RAW_KEY_SEL(y, g, n)：使用宏定义的一个扫键函数
	y:扫键结果存储变量
	g:GPIO引脚号，要扫键的c口
	n:GPIO引脚掩码（1026GPIO分组进行查询，每8个引脚为一组）

使用实例：
```
        tgroup = col[i]  >> 3;

        tgpionum = 1 << (col[i] & 7);

        GET_RAW_KEY_SEL(ksSelMapCurr[i], tgroup, tgpionum);
```


Keyscan文件分则处理扫键(`KS_ScanMatrix`)任务，在检测到按键按下后通过`KeyIn`函数来将按键放入`ksPool`,在检测到按键松开后通过`KeyOut`来将按键释放位置位，并将已释放的按键从事件池移除

# static void keyOut(byte key)主要功能
### 1. 判断按键类型
按键类型 条件 处理方式 多键/系统键 (key & 0xF0) == 0xf0 或 (key & 0xD0) == 0xD0 设置 mult_key_status = KEY_RELEASE 普通标准键 其他 设置 standard_key_release_flag = 1

### 2. 修饰键释放处理
- 修饰键范围 ： HID_KEY_LEFT_CTL ~ HID_KEY_RIGHT_GUI （Ctrl, Shift, Alt, GUI）
- 处理方式 ：清除 standardSel 中对应状态位
- 特点 ：不存储在 ksPool 数组中，直接返回
### 3. 普通键移除
- 存储位置 ： ksPool[6] 数组
- 移除方式 ：元素前移，保持数组紧凑
- 示例 ：释放 'B' → ['A','B','C','D','E',0] → ['A','C','D','E','0','0']
## 数据结构
```
ksEvtPool 结构体：
├── standardSel  (修饰键状态，1字节，8位)
└── ksPool[6]    (普通按键缓冲，6字节)
```
## 核心逻辑流程
```
keyOut(key)
    │
    ├─► 判断按键类型 → 设置释放标志
    │
    ├─► 是修饰键? → 清除standardSel位 → 返回
    │
    └─► 否 → 遍历ksPool数组
            │
            ├─► 找到目标按键 → 标记found=1
            │
            └─► 元素前移 → 保持数组紧凑
```

# KS_Initialize 函数流程
### 函数原型
```
void KS_Initialize()
```
### 初始化步骤
```
KS_Initialize()
    │
    ├─► 1. GPIO选择寄存器初始化
    │    REG_GPIO_SELECT(0) = 0
    │    REG_GPIO_SELECT(1) = 0
    │    REG_GPIO_SELECT(2) = 0
    │
    ├─► 2. 原始扫描初始化
    │    ksRawInitialize()
    │    └─► 设置GPIO为输入模式
    │
    └─► 3. 列初始化
         ksColInitialize()
         └─► 配置列引脚
```
### 功能
- 配置GPIO为键盘扫描模式
- 初始化扫描相关参数
- 准备接收按键输入


## KS_Unistall 函数流程
### 函数原型
```
void KS_Unistall()
```
### 卸载步骤
```
KS_Unistall()
    │
    ├─► 1. 清理列引脚GPIO配置
    │    for (遍历col数组)
    │       ├─► 禁用上拉电阻
    │       ├─► 设置为输出模式
    │       └─► 输出低电平
    │
    ├─► 2. 配置下拉电阻
    │    GPIO_fillpd()
    │
    ├─► 3. 配置唤醒功能
    │    │
    │    ├─► 读取GPIO状态
    │    │
    │    └─► 根据long_press_flag设置唤醒
    │         ├─► 有长按 → 按当前状态设置唤醒
    │         └─► 无长按 → 保存当前状态作为唤醒条件
    │
    └─► 4. 返回 (进入低功耗模式)
```


# bit_count 函数
bit_count 函数是一个用于 计算一个字节中置位（为1）的位数 的函数，也称为"汉明重量"（Hamming Weight）或"population count"。

## 代码实现
```
static byte bit_count(byte v)
{
    unsigned char c;
    for (c = 0; v; c++) {
        v &= v - 1;
    }
    return c;
}
```
## 算法原理
这个函数使用了经典的 Brian Kernighan算法 来计算置位位数：

1. 核心操作 ： v &= v - 1
   
   - 每次执行都会清除 v 中最低位的 1
   - 例如： 0b10101000 &= 0b10101000 - 1 = 0b10101000 & 0b10100111 = 0b10100000
2. 执行过程示例 ：
   
   ```
   假设 v = 0b10101010 (十进制 170，有4个位为1)
   
   第1次循环：
     v = 0b10101010，非零，c = 1
     v &= v - 1 = 0b10101010 & 0b10101001 = 0b10101000
   
   第2次循环：
     v = 0b10101000，非零，c = 2
     v &= v - 1 = 0b10101000 & 0b10100111 = 0b10100000
   
   第3次循环：
     v = 0b10100000，非零，c = 3
     v &= v - 1 = 0b10100000 & 0b10011111 = 0b10000000
   
   第4次循环：
     v = 0b10000000，非零，c = 4
     v &= v - 1 = 0b10000000 & 0b01111111 = 0b00000000
   
   第5次循环：
     v = 0，退出循环
   
   返回 c = 4
   ```