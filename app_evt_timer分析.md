## app_evt_timer 100 ms 定时器笔记

### 触发链路
- `ui_timer_check` 以 `clkn_bt` 为基准，每累计 320 个 BT 时钟（源码注释：`320btclk = 100ms`）就调用 `ui_timer_check_send_evt`。citeprogram/ui.prog:262
- `ui_timer_check_send_evt` 直接 `branch app_evt_timer`，确保 100 ms Tick 恒定触发 `app_evt_timer`。

### 函数流程（program/app.prog:101 起）
1. `app_evt_timer`：入口先 `store 1,mem_app_evt_timer_count`，为循环准备执行计数。
2. `app_evt_100ms_loop`：
   - `fetch 1,mem_app_evt_timer_count`；为 0 时 `rtn blank`，否则 `increase -1` 并写回，实现“只执行一次/延迟执行”机制。citeprogram/app.prog:105
   - 顺序调用：
     - `ui_button_polling`
     - `app_lpm_wake_auto_lock_timer`
     - `flash_write_spi_sm_timer`
     - `fetch 2,mem_cb_event_timer` → `call callback_func`
   - 最后 `branch app_evt_100ms_loop`，保持 100 ms 周期。

### RAM 变量
- `mem_app_evt_timer_count`：`format/app.format` 的 `memalloc` 内 1 单元变量，可由补丁读写控制循环次数。
- `mem_cb_event_timer`：`xmemalloc` 中 2 字节函数指针，运行时指向不同应用的 100 ms 回调（如 `shutter_le_bb_event_timer`）。

### 使用方式
1. 在补丁或应用初始化中实现自定义回调（例如 `my_timer_handler`）。
2. 需要保留原逻辑时，先保存旧指针，再写包装回调：
   - 保存：`fetch 2,mem_cb_event_timer` → 存到自定义 RAM。
   - 注册：`setarg my_timer_wrapper` → `store 2,mem_cb_event_timer`。
3. 在自定义回调中执行额外逻辑，必要时调用旧指针；注意保持执行时间短，避免阻塞 100 ms 循环。

### 局限与扩展
- 100 ms 周期由 ROM 固定，无法直接变成真实的 10 ms 定时器；软件拆分只能模拟更细的逻辑步长。
- 若需更高分辨率，应查找芯片的其他硬件定时资源或更高频事件入口；否则只能在 100 ms Tick 内监测 `clkn_bt` 变化做补偿。
