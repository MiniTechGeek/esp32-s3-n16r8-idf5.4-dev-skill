---
name: esp32s3-idf-project
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的工程结构、CMake 配置、sdkconfig 基线、分区表、idf.py 构建烧录与组件依赖。当用户要新建工程、写 CMakeLists.txt 或 idf_component.yml、配置 Flash/PSRAM/CPU 频率、设计 partitions.csv、执行 idf.py 命令、排查编译/链接/烧录错误时加载本技能。
---

# ESP32-S3N16R8 · ESP-IDF v5.4 工程 / 构建 / 烧录

## 描述

规定 N16R8 工程的最小骨架与配置基线：target 必须是 `esp32s3`，flash 必须按 **16 MB**、PSRAM 必须按 **8 MB Octal**、CPU 240 MHz，分区表必须落在 16 MB 内。所有 API 与组件名以资料包 `D. ESP-IDF v5.4 文档\components\` 为准。

## 使用场景

- 创建/改造工程目录、写根 `CMakeLists.txt` 与 `main/CMakeLists.txt`
- 写 `sdkconfig.defaults`、`partitions.csv`、`idf_component.yml`
- 执行 `idf.py set-target / menuconfig / build / flash / monitor / size / erase-flash`
- 添加组件依赖（`REQUIRES`/`PRIV_REQUIRES`）、引入外部组件
- 排查「编译找不到头文件」「链接 undefined reference」「烧录后反复重启」「分区表放不下」

## 指令

### 1. 标准工程骨架

```
my_project/
├── CMakeLists.txt                 # 根：cmake_minimum_required + include($ENV{IDF_PATH}/tools/cmake/project.cmake) + project()
├── sdkconfig.defaults             # 版本可控的默认配置（不要用 sdkconfig）
├── partitions.csv                 # 自定义 16 MB 分区表
├── partitions_nvs.csv             # 可选
├── main/
│   ├── CMakeLists.txt             # idf_component_register(SRCS ... INCLUDE_DIRS . REQUIRES ...)
│   ├── idf_component.yml         # 组件管理器依赖（如 espressif/led_strip）
│   ├── app_main.c
│   └── Kconfig.projbuild          # 可选：项目级 menuconfig 项
├── components/                    # 可选：自定义组件
└── managed_components/            # 自动生成，勿手改（加入 .gitignore）
```

根 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.22)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(esp32s3_n16r8_app)
```

`main/CMakeLists.txt`（按需增删 REQUIRES）：

```cmake
idf_component_register(
    SRCS "app_main.c"
    INCLUDE_DIRS "."
    REQUIRES
        esp_driver_gpio esp_driver_gptimer esp_driver_ledc
        esp_driver_i2c esp_driver_spi esp_driver_uart
        esp_adc esp_timer esp_lcd
        nvs_flash spi_flash esp_partition app_update
        esp_wifi esp_netif esp_event esp_http_client mqtt
        freertos log esp_system esp_common esp_psram
        driver
)
```

> v5.4 起 legacy `driver` 组件已拆分为 `esp_driver_*` 系列；新代码**直接写具体组件名**，不要只写 `driver`。找不到头文件时先到 `D. ESP-IDF v5.4 文档\components\<组件>\include\` 确认真实路径。

### 2. sdkconfig.defaults 基线（N16R8 专用）

```ini
# ---- 目标与 Flash ----
CONFIG_IDF_TARGET="esp32s3"
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESPTOOLPY_FLASHSIZE="16MB"
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
CONFIG_BOOTLOADER_FLASH_DC_AWARE=y

# ---- PSRAM：8 MB Octal ----
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_TYPE_AUTO=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_SPIRAM_USE_MALLOC=y
CONFIG_SPIRAM_USE_CAPS_ALLOC=y
CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y
CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY=n
CONFIG_SPIRAM_FETCH_INSTRUCTIONS=n
CONFIG_SPIRAM_RODATA=n

# ---- CPU ----
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ=240
CONFIG_FREERTOS_HZ=1000
CONFIG_FREERTOS_UNICORE=n

# ---- 分区表 ----
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_FILENAME="partitions.csv"

# ---- 控制台/下载（二选一）----
# 方案 A：UART0（GPIO43/44），最稳，日志与下载分开
CONFIG_ESP_CONSOLE_UART_DEFAULT=y
# 方案 B：USB-Serial-JTAG（GPIO19/20），单 USB 口即可烧录+日志
# CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y

# ---- 系统 ----
CONFIG_ESP_TASK_WDT_EN=y
CONFIG_ESP_SYSTEM_PANIC_PRINT_REBOOT=y
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
```

要点：

- `CONFIG_ESPTOOLPY_FLASHSIZE_16MB` 对应 Kconfig 枚举 `ESPTOOLPY_FLASHSIZE_16MB`（`components/esptool_py/Kconfig.projbuild` 第 126 行）。**不要用默认 2/4 MB**，否则 16 MB 后半段无法使用且分区表越界。
- N16R8 的 **PSRAM 是 Octal**，必须 `CONFIG_SPIRAM_MODE_OCT=y`；Quad PSRAM 配置会导致初始化失败或容量减半。
- Flash 默认 ≤80 MHz 且不支持自动暂停（模组手册 1.2 注 3）；**不要**擅自配 `CONFIG_ESPTOOLPY_FLASHFREQ_120M`。
- `CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY=n`：任务栈默认留在内部 SRAM，避免栈放 PSRAM 带来的延迟与 cache 问题；确需外置时显式开启并评估。
- `CONFIG_FREERTOS_HZ=1000` 提高定时精度；若追求低功耗可用默认 100 Hz 并配 `CONFIG_PM_ENABLE`。
- 控制台二选一：用 USB-Serial-JTAG 时日志走 USB，UART0 空闲；但注意勘误 USBOTG-4289 只影响 **USB-OTG 下载**，不影响 USB-Serial-JTAG。
- `CONFIG_ESP_MAIN_TASK_STACK_SIZE`（默认 3584）对带 Wi-Fi/BLE/JSON 的 app 偏小，建议 8192。

### 3. 16 MB 分区表模板（`partitions.csv`）

```csv
# Name,   Type, SubType,  Offset,   Size,     Flags
nvs,      data, nvs,      0x9000,   0x6000,
phy_init, data, phy,      0xf000,   0x1000,
factory,  app,  factory,  0x10000,  0x400000,
ota_0,    app,  ota_0,    0x410000, 0x400000,
ota_1,    app,  ota_1,    0x810000, 0x400000,
otadata,  data, ota,      0xc10000, 0x2000,
storage,  data, spiffs,   0xc12000, 0x3E0000,
```

- 最后一个分区结束 = `0xC12000 + 0x3E0000 = 0xFF2000` < `0x1000000`（16 MB），合法。
- `0x0~0x8000` 留给 bootloader + partition table，`0x9000` 起是 NVS；**不要**把 NVS 移到 `0x8000`。
- 两个 OTA 分区各 4 MB，配合 `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y` 可回滚。
- 若无 OTA 需求可用 `CONFIG_PARTITION_TABLE_SINGLE_APP_LARGE=y`（单一大 app 分区）。
- 使用 `idf.py partition-table` / `parttool.py` 校验；分区表 CSV 里 `Offset` 必须 4 KB 对齐（app 分区 64 KB 对齐）。

### 4. 命令速查

```bash
idf.py set-target esp32s3                 # 首次必做，生成 sdkconfig + bootloader
idf.py menuconfig                          # 交互配置
idf.py build                               # 编译
idf.py -p COM3 -b 921600 flash monitor     # 烧录 + 串口监视（Ctrl+] 退出）
idf.py -p COM3 flash                       # 只烧录
idf.py monitor                             # 只监视
idf.py erase-flash                         # 全片擦除（NVS/OTA 残留异常时用）
idf.py fullclean                           # 清构建产物（改 sdkconfig/target 后）
idf.py size / size-components / size-files # 体积分析
idf.py partition-table                     # 打印解析后的分区表
idf.py app / bootloader                    # 单独构建
idf.py coredump-info / coredump-debug       # 崩溃转储（见技能 12）
idf.py --list-targets                      # 查看支持的芯片
idf.py create-project                      # 从模板建工程
```

烧录通道（见技能 01）：

| 通道 | 引脚 | 进入下载模式 | 备注 |
| --- | --- | --- | --- |
| UART0 | GPIO43(TX)/GPIO44(RX) | GPIO0=0 且 GPIO46=0 | 最通用，需 USB-UART 桥 |
| USB-Serial-JTAG | GPIO19(D−)/GPIO20(D+) | 同上 | **推荐**，单 USB 口，绕开 USBOTG-4289 |
| USB-OTG | 外部 PHY 固定脚 | 同上 | 早期批次 eFuse 禁用，见勘误 USBOTG-4289 |

### 5. 组件与 API 溯源方法

资料包 `D. ESP-IDF v5.4 文档` 只有 `components/` 与 `examples/`，**没有 `docs/`**。因此：

1. 需要某 API 时，到 `D. ESP-IDF v5.4 文档\components\<组件>\include\` 找头文件确认签名。
2. 需要完整用法时，到 `D. ESP-IDF v5.4 文档\examples\<分类>\<示例>\main\*.c` 找可直接改编的模板。
3. 配置项到 `components/<组件>/Kconfig*` 查 `config XXX` 名字。
4. 组件共 106 个，示例 21 类（`get-started / peripherals / wifi / protocols / storage / system / bluetooth / security / provisioning / network / ethernet / mesh / phy / zigbee / ieee802154 / openthread / build_system / cxx / custom_bootloader / common_components`）。

常用「示例 → 头文件」对照：

| 需求 | 参考示例 | 主头文件 |
| --- | --- | --- |
| 最小工程 | `get-started/hello_world` | `esp_system.h`, `esp_chip_info.h` |
| GPIO/LED | `get-started/blink`, `peripherals/gpio/generic_gpio` | `driver/gpio.h` |
| I2C master | `peripherals/i2c/i2c_eeprom` | `driver/i2c_master.h` |
| SPI master | `peripherals/spi_master/lcd` | `driver/spi_master.h` |
| UART | `peripherals/uart/uart_async_rxtxtasks` | `driver/uart.h` |
| ADC | `peripherals/adc/oneshot_read`, `continuous_read` | `esp_adc/adc_oneshot.h`, `adc_cali.h` |
| 定时器 | `peripherals/timer_group/gptimer` | `driver/gptimer.h` |
| Wi-Fi STA | `wifi/getting_started/station` | `esp_wifi.h`, `esp_netif.h`, `esp_event.h` |
| NVS | `storage/nvs_rw_value` | `nvs_flash.h`, `nvs.h` |
| OTA | `system/ota/simple_ota_example` | `esp_ota_ops.h`, `esp_https_ota.h` |
| Deep sleep | `system/deep_sleep` | `esp_sleep.h`, `driver/rtc_io.h` |
| FreeRTOS SMP | `system/freertos/basic_freertos_smp_usage` | `freertos/FreeRTOS.h`, `freertos/task.h` |
| LCD | `peripherals/lcd/spi_lcd_touch` | `esp_lcd_panel_io.h`, `esp_lcd_panel_ops.h` |
| USB Serial/JTAG | `peripherals/usb_serial_jtag/usb_serial_jtag_echo` | `driver/usb_serial_jtag.h` |

### 6. 常见构建/烧录故障

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `fatal error: driver/i2c.h: No such file` | 用了 v4.x legacy 头 | 改 `driver/i2c_master.h`，`REQUIRES esp_driver_i2c` |
| `undefined reference to xxx` | `main/CMakeLists.txt` 缺 `REQUIRES` | 补对应组件名 |
| 烧录后反复重启 / `invalid header` | Flash 大小或模式配置错、 partitions 越界 | 核对 16 MB / DIO / 80 MHz、擦除后重烧、`idf.py partition-table` |
| `PSRAM ID read error` | 把 Octal 配成 Quad | `CONFIG_SPIRAM_MODE_OCT=y` |
| `A fatal error occurred: Failed to connect` | 未进下载模式 | 确认 GPIO0=0 & GPIO46=0，或按住 BOOT 再上电；换 USB-Serial-JTAG 通道 |
| `esp_image: Image length doesn't fit in partition` | app 分区太小 | 扩大 factory/ota 分区（各 4 MB 已足够大多数应用） |
| `Brownout detector was triggered` | 供电不足 | 3.3 V ≥500 mA，检查电源上升斜率与 EN RC |

## 示例

最小 `app_main.c`（v5.4 写法，含芯片信息打印）：

```c
#include <stdio.h>
#include <inttypes.h>
#include "sdkconfig.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_chip_info.h"
#include "esp_flash.h"
#include "esp_system.h"
#include "esp_log.h"
#include "esp_psram.h"

static const char *TAG = "app";

void app_main(void)
{
    esp_chip_info_t info;
    esp_chip_info(&info);
    uint32_t flash_size = 0;
    ESP_ERROR_CHECK(esp_flash_get_size(NULL, &flash_size));
    ESP_LOGI(TAG, "target=%s cores=%d rev=v%d.%d flash=%" PRIu32 " MB",
             CONFIG_IDF_TARGET, info.cores,
             info.revision / 100, info.revision % 100,
             flash_size / (1024U * 1024U));
    ESP_LOGI(TAG, "PSRAM: %s, size=%zu B, free heap=%" PRIu32 " B",
             esp_psram_is_initialized() ? "OK" : "NOT FOUND",
             esp_psram_get_size(),
             esp_get_free_heap_size());
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

> 期望输出：`target=esp32s3 cores=2 flash=16 MB; PSRAM: OK, size=8388608 B`。若 flash 显示 2/4 MB 或 PSRAM NOT FOUND，说明 sdkconfig 未按第 2 节配置。

## 检查清单

- [ ] `idf.py set-target esp32s3` 已执行，`CONFIG_IDF_TARGET="esp32s3"`
- [ ] flash = 16 MB、DIO、80 MHz；PSRAM = **Octal** 8 MB、80 MHz
- [ ] CPU 240 MHz，双核未误配 `FREERTOS_UNICORE`
- [ ] 分区表末端 < `0x1000000`，app 分区足够大，OTA 场景有 otadata
- [ ] `main/CMakeLists.txt` 的 `REQUIRES` 覆盖所有用到的组件
- [ ] 未使用任何 legacy 头文件；所有 API 都能在 D 目录头文件中找到
- [ ] 控制台/下载通道与硬件连线一致，烧录失败时能解释 GPIO0/GPIO46 时序
