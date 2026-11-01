---
name: esp32s3-debug-review
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的调试诊断、崩溃/Guru Meditation 与 core dump 分析、GDB/OpenOCD、看门狗复位定位、日志与运行时统计、常见问题排查、全部芯片勘误避坑清单与代码审查清单。当任务涉及程序崩溃、反复重启、看门狗触发、烧录失败、性能异常、代码评审或交付前检查时加载本技能。
---

# ESP32-S3N16R8 · 调试 / 勘误避坑 / 代码审查

## 描述

把资料包里的勘误、硬件陷阱与 ESP-IDF v5.4 的调试工具链变成可执行的排查流程和交付前审查清单。任何代码在交付前应过一遍本技能第 6 节。

## 使用场景

- Guru Meditation / panic / LoadProhibited / StoreProhibited / IllegalInstruction
- 反复重启、Brownout、WDT 复位、启动卡死
- 烧录失败、找不到端口、分区/Flash 配置错
- 内存不足、栈溢出、堆碎片、卡顿/性能分析
- 代码评审、交付前自检、故障复盘

## 指令

### 1. 工具链速查

| 目的 | 命令 / 工具 | 说明 |
| --- | --- | --- |
| 串口日志 | `idf.py -p COM3 monitor`（`Ctrl+]` 退出） | 内置 `idf_monitor.py`，自动解析 panic backtrace 地址到源码行 |
| 只烧录/只监视 | `idf.py -p COM3 flash` / `idf.py monitor` | |
| 崩溃转储 | `CONFIG_ESP_COREDUMP_ENABLE_TO_FLASH/UART` → `idf.py coredump-info` / `coredump-debug` | 组件 `espcoredump`；需分区表里留 `coredump` 分区 |
| GDB stub | `CONFIG_ESP_SYSTEM_PANIC_GDBSTUB=y` → panic 后 `idf.py gdb` | 组件 `esp_gdbstub`，示例 `examples/system/gdbstub` |
| JTAG 调试 | OpenOCD + `xtensa-esp32s3-elf-gdb`；JTAG 源由 GPIO3 + eFuse 决定（技能 01） | 或用内置 **USB-Serial-JTAG**（GPIO19/20）免外接调试器 |
| 运行时统计 | `CONFIG_FREERTOS_GENERATE_RUN_TIME_STATS`, `CONFIG_FREERTOS_USE_TRACE_FACILITY`, `vTaskGetRunTimeStats()`, `vTaskList()` | 组件 `freertos` |
| 跟踪 | `examples/system/sysview_tracing`（SystemView）、`app_trace_basic`（`esp_app_trace.h`） | 组件 `app_trace`, `perfmon` |
| 栈/堆 | `uxTaskGetStackHighWaterMark()`, `heap_caps_dump_all()`, `heap_caps_check_integrity_all()`, `esp_get_minimum_free_heap_size()` | 技能 09 |
| 体积 | `idf.py size`, `size-components`, `size-files`, `idf.py partition-table` | 检查 app 是否超分区 |
| 复位原因 | `esp_reset_reason()`, `esp_get_reset_reason()` (`esp_system.h`) | 区分上电/软复位/WDT/Brownout/Deep-sleep 唤醒 |
| 芯片信息 | `esp_chip_info()`, `esp_get_idf_version()`, `esp_flash_get_size()`, `esp_psram_get_size()` | 技能 02 示例 |

### 2. 崩溃分析流程

1. **看 panic 首行**：`Guru Meditation Error: Core 0 panic'ed (LoadProhibited)` 等，记下 `Core`, `exception cause`, `epc`/`pc`, `backtrace`。
2. `idf.py monitor` 会自动把地址翻译成 `文件:行号`；若无符号，用 `xtensa-esp32s3-elf-addr2line -e build/app.elf <addr>`。
3. 看 `esp_reset_reason()` 与 RTC/ROM 日志，区分冷启动、WDT、Brownout、Deep-sleep 唤醒。
4. 检查 `CONFIG_ESP_SYSTEM_PANIC_*` 配置；生产建议 `PANIC_PRINT_REBOOT` + coredump 到 flash，便于现场取回。
5. 常见 cause 对应：
   - `LoadProhibited` / `StoreProhibited`：空指针、越界、访问已释放内存、访问非法地址（含 GDMA 描述符指到 PSRAM 触发的 `GDMA_ETXMEM_REJECT_INT`）。
   - `IllegalInstruction`：函数指针错、把 PSRAM 里的代码当可执行（未开 XIP）、IRAM_ATTR 缺失导致 flash cache 关闭期间取指。
   - `IntegerDivideByZero` / `Unhandled debug exception`：检查除零、断言。
   - `Interrupt wdt timeout on CPU0/1`：ISR 里做了耗时操作或 `portENTER_CRITICAL` 太久。
   - `Task watchdog got triggered`：任务长时间不喂狗/阻塞，见第 3 节。
6. 用 `idf.py coredump-info` 恢复崩溃时各任务栈与寄存器；`examples/system/gdbstub` 演示 panic 后直接进 GDB。

### 3. 反复重启 / WDT / Brownout 排查树

```
反复重启
├─ rst:0x? (Brownout detector was triggered) → 供电不足/上电斜率慢
│    → 3.3V ≥500 mA、10 µF 大电容、EN RC 10k+1µF、电源监控芯片（阈值 3.0 V）
├─ rst:0x? (TG1WDT/TWDT) → 任务阻塞或死循环
│    → 检查 portMAX_DELAY、socket 超时、TWDT 订阅/喂狗、esp_task_wdt_print_triggered_tasks()
├─ rst:0x? (RTCWDT / SW_CPU_RESET) → 软件复位/OTA 重启/看门狗
│    → esp_reset_reason()、OTA 回滚状态、esp_ota_get_state_partition()
├─ 启动卡在 "invalid header" / "image length doesn't fit"
│    → Flash 大小/模式配置错、分区表越界 → 技能 02 核对 16 MB/DIO/80 MHz
├─ "PSRAM ID read error" → CONFIG_SPIRAM_MODE_OCT 未开或焊错型号
└─ 上电后无日志 → 控制台通道选错（UART0 vs USB-Serial-JTAG）、GPIO19/20 被占用
```

- **Brownout** 最常见于 Wi-Fi TX 瞬态电流；N16R8 电源必须能供 ≥500 mA 且纹波小。
- `esp_task_wdt_print_triggered_tasks()` 可打印哪个任务没喂狗；`CONFIG_ESP_TASK_WDT_EN=y` + `trigger_panic=false` 先观察再决定是否 panic。
- 中断看门狗：`CONFIG_ESP_INT_WDT_TIMEOUT_MS`（默认 300 ms）；ISR 里禁止日志/延时/临界区过长。
- Deep-sleep 唤醒复位是**正常**行为，用 `esp_sleep_get_wakeup_cause()` 与 `RTC_DATA_ATTR` 区分（技能 10）。

### 4. 全部勘误避坑清单（Errata v1.3，逐条核对）

| 编号 | 影响 | ESP-IDF v5.4 状态 | 应用侧必须做的事 |
| --- | --- | --- | --- |
| CACHE-126 | cache 回写时同 cache line 访问返回错数据 | **已自动规避**（v4.4.6+/v5.0.4+/v5.1.1+/v5.2+） | 不要自行改写 cache 回写/关中断流程；保持 IDF 版本 |
| RTC-126 | Light-sleep 后读 RTC 域寄存器可能错 | **已规避**（v4.4+） | 不要在 Light-sleep 中掉电 RTC 外设（不影响功耗） |
| ANALOG-160 | `BIAS_SLEEP=0 + PD_CUR=1` **永久损坏芯片** | **已禁用**（v4.4.2+/v5.0+） | 绝不手写模拟电源寄存器；ULP/触摸睡眠场景保持默认配置 |
| LCD-239 | RGB 分频=1 时 PCLK/连续帧异常；I8080 `ahead_cycle≤2` 首数据错 | **已规避**（v4.4.5+/v5.0.3+/v5.1+） | 用 `esp_lcd` 高层 API；禁止寄存器级 `LCD_CLK_EQU_SYSCLK=1` |
| USBOTG-4289 | 早期批次 USB-OTG 下载被 eFuse 禁用 | 部分 v0.2 批次已修 | **下载优先 USB-Serial-JTAG(GPIO19/20) 或 UART0(GPIO43/44)**；检查 Date Code ≥2219 / PW ≥PW-2022-06-XXXX |
| RMT-176 | 连续 TX 停止后 idle 电平不确定 | **已配置**（v5.0+ 用寄存器控制） | 不要手动把 `RMT_IDLE_OUT_EN_CHn` 改回 0 |
| TOUCH-100 | 前两次 `TOUCH_SCAN_DONE_INT` 原始数据未定义 | 驱动未完全封装 | **应用层丢弃前两次采样结果**，然后关闭/停用该中断 |
| ADC-183 | SAR ADC2 数字控制器(DMA) 可能死锁 | 未修复 | **ADC2 禁止 `adc_continuous`/DMA**，改用 RTC 控制器单次读；模拟输入优先 ADC1(GPIO1~10) |

芯片版本识别：eFuse `EFUSE_RD_MAC_SPI_SYS_5_REG[25:23]` + `EFUSE_RD_MAC_SPI_SYS_3_REG[20:18]`；芯片丝印 Tracking Information 第二个字符 `A/B/C` → v0.0/v0.1/v0.2；模组丝印 Specification Identifier `—/M0/MC`。`esp_chip_info_t.revision` 可打印（技能 02 示例）。

### 5. 常见问题速查

| 现象 | 高概率原因 | 处理 |
| --- | --- | --- |
| 编译找不到头文件 | 用了 legacy 头 / `REQUIRES` 缺失 | 技能 02/03/04 换 v5.4 新头并补组件 |
| `undefined reference` | 组件未加入 `REQUIRES` | 补 CMake；`idf.py fullclean` 后重编 |
| 烧录连不上 | 未进 Joint Download Boot | GPIO0=0 且 GPIO46=0；按住 BOOT 上电；换 USB-Serial-JTAG |
| 串口乱码 | 波特率/晶振频偏 | 115200/921600 一致；40 MHz 晶振 ±10 ppm（技能 01） |
| Wi-Fi 连不上 | 加密方式/信号/晶振频偏 | `threshold.authmode`、RSSI、负载电容校准频偏 |
| ADC 读数飘 | 无 0.1 µF 滤波 / 用了 ADC2+Wi-Fi / 未校准 | 技能 05 |
| 触摸首次乱跳 | TOUCH-100 前两帧 | 丢弃前两次 |
| 内存不足 | 大缓冲放内部 SRAM | `MALLOC_CAP_SPIRAM`（技能 09） |
| DMA 传输出错 | 缓冲/描述符放 PSRAM | `MALLOC_CAP_DMA|INTERNAL`，内部 RAM 区 `0x3FC88000~0x3FCFFFFF` |
| 屏幕花屏/撕裂 | 引脚/pclk/缓冲/同步 | 技能 11 |
| OTA 后回滚 | 新固件未标记 valid | 技能 08 |

### 6. 交付前代码审查清单（每次生成/修改代码都要过）

**版本与 API**
- [ ] `idf.py set-target esp32s3`，IDF v5.4；`CONFIG_ESPTOOLPY_FLASHSIZE_16MB`、`CONFIG_SPIRAM_MODE_OCT`、CPU 240 MHz 已配
- [ ] 未使用 legacy 头（`i2c.h`/`rmt.h`/`timer.h`/`adc.h`/`esp_adc_cal.h`/`touch_pad.h`/`i2s.h`/`mcpwm.h` 旧接口）
- [ ] 所有 API 都在 `D. ESP-IDF v5.4 文档\components\<组件>\include\` 中真实存在

**硬件合法性**
- [ ] 无 GPIO26~37；已核对 strapping(0/3/45/46)、USB(19/20)、UART0(43/44)、JTAG(39~42)、DevKitC-1 RGB LED(38)
- [ ] 模拟输入优先 ADC1；未对 ADC2 用 DMA；触摸跳过前两帧
- [ ] 供电 3.3 V/≥500 mA、EN RC、40 MHz ±10 ppm、RF 50 Ω 已确认

**代码正确性**
- [ ] 每个 `esp_err_t` 都检查；致命路径 `ESP_ERROR_CHECK()`，运行期返回错误码
- [ ] ISR 加 `IRAM_ATTR`，只用 `FromISR` API 投递，无日志/延时/malloc，处理 `portYIELD_FROM_ISR()`
- [ ] 所有阻塞调用有有限超时；长任务订阅 TWDT 并喂狗
- [ ] 任务栈用 `uxTaskGetStackHighWaterMark()` 实测，余量 ≥25%
- [ ] 大缓冲 `MALLOC_CAP_SPIRAM`，DMA 缓冲 `MALLOC_CAP_DMA|MALLOC_CAP_INTERNAL`
- [ ] 共享数据有同步原语，锁顺序固定；无 `portMAX_DELAY` 无界阻塞
- [ ] 队列/信号量/任务创建返回值已检查（NULL / pdPASS）
- [ ] 日志统一 `static const char *TAG`，ISR/高频循环无无节流打印

**功能专项**
- [ ] 网络：初始化顺序正确、重连有上限退避、socket/HTTP/MQTT 有超时、HTTPS 有 CA
- [ ] 存储：NVS commit、flash 写前擦除 + 4 KB 对齐、分区末端 <16 MB
- [ ] OTA：有 otadata + 回滚自检标记 + 版本/防降级校验
- [ ] 低功耗：区分 Light/Deep-sleep、跨睡眠数据 `RTC_DATA_ATTR`、未触碰 RTC/模拟电源勘误配置
- [ ] 安全：密钥来自 eFuse/加密 NVS，随机数用 `esp_random()`，eFuse 烧写有恢复方案

**验证**
- [ ] `idf.py build` 通过；`idf.py size` 显示 app 未超分区
- [ ] 冷启动日志正常；`esp_reset_reason()` 无非预期复位
- [ ] 至少一次故障注入（断网、传感器异常、队列满）验证错误路径
- [ ] 72 h 压力测试无栈溢出、无 WDT 复位（量产前）

## 示例

```c
/* 复位原因自助诊断：开机第一行日志 */
#include "esp_system.h"
static const char *rst_names[] = {"UNKNOWN","POWERON","SW","PANIC","INT_WDT",
    "TASK_WDT","WDT","DEEPSLEEP","BROWNOUT","SDIO"};
void app_main(void)
{
    esp_reset_reason_t r = esp_reset_reason();
    ESP_LOGI("boot", "reset reason: %s (%d), idf: %s, heap: %" PRIu32,
             (r < 10 ? rst_names[r] : "?"), r, esp_get_idf_version(),
             esp_get_free_heap_size());
}
```

> 若这里打印 `TASK_WDT` 或 `BROWNOUT`，直接按第 3 节排查树定位，不要先改业务代码。

## 检查清单（本技能自身）

- [ ] 崩溃分析先取 panic 首行 + backtrace + reset reason，再改代码
- [ ] 8 条勘误逐条确认应用侧动作（尤其 ADC-183 / TOUCH-100 / USBOTG-4289）
- [ ] 交付前完整走完第 6 节审查清单
- [ ] 所有结论能回溯到资料包 PDF 章节或 D 目录头文件/示例
