---
name: esp32s3-power-security
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的功耗模式、动态调频、Light/Deep sleep 与唤醒源、ULP-RISC-V/FSM、RTC 存储器，以及 Flash 加密、安全启动、eFuse 与硬件加密加速器。当任务涉及电池供电、低功耗唤醒、睡眠电流、ULP 协处理器、固件防抄、密钥存储或安全启动配置时加载本技能。
---

# ESP32-S3N16R8 · 低功耗与安全

## 描述

把 ESP32-S3 的四种功耗模式、唤醒源、ULP 与 N16R8 的安全机制（Flash 加密 / 安全启动 / eFuse）落到可执行的配置与代码，并标注影响睡眠的芯片勘误。

## 使用场景

- 电池/低功耗设备、定时唤醒、GPIO/触摸/UART 唤醒
- Light-sleep / Deep-sleep / Modem-sleep 选择与电流评估
- 动态调频 (DFS) 与 `esp_pm` 电源管理锁
- ULP-RISC-V / ULP-FSM 在睡眠中采样
- Flash 加密、安全启动 v2、eFuse 烧写、硬件加密加速
- 睡眠相关勘误（RTC-126 / ANALOG-160 / ADC-183）

## 指令

### 1. 功耗模式与能力（A 数据手册）

| 模式 | 说明 | 关键事实 |
| --- | --- | --- |
| Active | CPU/无线全开 | 240 MHz 双核，Wi-Fi TX 时峰值电流最大，电源需 ≥500 mA |
| Modem-sleep | CPU 运行，Wi-Fi 按 DTIM 休眠 | `esp_wifi_set_ps(WIFI_PS_MIN_MODEM / MAX_MODEM)` |
| Light-sleep | CPU 暂停、数字外设时钟门控、保留内部 RAM | `esp_light_sleep_start()`；可被 GPIO/timer/UART/touch/ULP 唤醒，**唤醒后继续执行** |
| Deep-sleep | 关闭 CPU 与大部分数字域，仅 RTC 域供电 | **低至 7 µA**；`esp_deep_sleep_start()`；**唤醒后从复位重新运行**（不保留普通 RAM） |

保留内容：

- RTC FAST/SLOW 存储器共 **16 KB**（8+8 KB），Deep-sleep 期间保持 → 用 `RTC_DATA_ATTR` / `RTC_FAST_ATTR` / `RTC_SLOW_ATTR` 修饰变量。
- 只有 **VDD3P3_RTC 电源域的 GPIO（GPIO0~21）** 能在 Deep-sleep 中保持状态/作唤醒源；其余 GPIO 会掉电，用 `rtc_gpio_hold_en()` / `rtc_gpio_isolate()` 管理。
- Deep-sleep 前必须 `rtc_gpio_isolate()` 悬空引脚以降低漏电流；睡眠期间 GPIO 状态用 `gpio_hold_en()` / `gpio_deep_sleep_hold_en()` 保持。

### 2. 电源管理（DFS + Light-sleep 自动进入）

```c
#include "esp_pm.h"
esp_pm_config_t pm_cfg = {
    .max_freq_mhz = 240,
    .min_freq_mhz = 40,          /* 空闲时降频到 40 MHz */
    .light_sleep_enable = true,  /* 空闲时自动进 Light-sleep */
};
ESP_ERROR_CHECK(esp_pm_configure(&pm_cfg));

/* 需要高性能/精确时序时拿锁，用完释放 */
esp_pm_lock_handle_t lock;
ESP_ERROR_CHECK(esp_pm_lock_create(ESP_PM_CPU_FREQ_MAX, 0, "spi_burst", &lock));
ESP_ERROR_CHECK(esp_pm_lock_acquire(lock));
/* ... 高精度操作 ... */
ESP_ERROR_CHECK(esp_pm_lock_release(lock));
ESP_ERROR_CHECK(esp_pm_lock_delete(lock));
```

Kconfig：`CONFIG_PM_ENABLE=y`, `CONFIG_PM_DFS_INIT_AUTO`, `CONFIG_FREERTOS_USE_TICKLESS_IDLE=y`（tickless idle 才能自动 Light-sleep），`CONFIG_RTC_CLK_SRC_INT_RC` / `EXT_XTAL` 选 RTC 时钟源；外部 32.768 kHz 晶振可提高定时精度并降低平均功耗。

> **勘误 RTC-126**：不要在 Light-sleep 中掉电 RTC 外设，否则唤醒后 CPU 读 RTC 域寄存器可能出错；ESP-IDF v4.4+（含 v5.4）已规避，且不影响功耗 → **不要手动关闭 RTC 外设电源**。
> **勘误 ANALOG-160**：`BIAS_SLEEP=0` + `PD_CUR=1` 会**永久损坏芯片**（ULP/触摸在睡眠中使用时可能触发）；v5.0+ 已禁用该配置，**禁止手写寄存器绕过**。

### 3. 唤醒源与 Deep-sleep 标准写法

```c
#include "esp_sleep.h"
#include "driver/rtc_io.h"

/* 定时唤醒（µs） */
ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup(10ULL * 1000 * 1000));   /* 10 s */

/* ext0：单个 RTC GPIO 指定电平（需 RTC 域保持供电） */
ESP_ERROR_CHECK(esp_sleep_enable_ext0_wakeup(GPIO_NUM_4, 0));
/* ext1：多个 RTC GPIO 任一/全部满足（v5.4 推荐 *_io 版本） */
ESP_ERROR_CHECK(esp_sleep_enable_ext1_wakeup_io(1ULL << GPIO_NUM_2 | 1ULL << GPIO_NUM_4,
                                                 ESP_EXT1_WAKEUP_ANY_HIGH));
/* GPIO 唤醒（Light-sleep 可用）、触摸、UART、ULP、Wi-Fi/BT beacon */
ESP_ERROR_CHECK(esp_sleep_enable_gpio_wakeup());
/* esp_sleep_enable_touchpad_wakeup(); esp_sleep_enable_uart_wakeup(0);
   esp_sleep_enable_ulp_wakeup(); esp_sleep_enable_wifi_beacon_wakeup(); */

/* 记录跨睡眠状态：RTC 存储器 */
static RTC_DATA_ATTR uint32_t s_boot_count = 0;
static RTC_DATA_ATTR struct timeval s_sleep_enter;    /* 见 examples/system/deep_sleep */

esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
switch (cause) {
case ESP_SLEEP_WAKEUP_TIMER:  ESP_LOGI(TAG, "wake by timer"); break;
case ESP_SLEEP_WAKEUP_EXT1:  {
    uint64_t mask = esp_sleep_get_ext1_wakeup_status();
    ESP_LOGI(TAG, "wake by GPIO %d", __builtin_ffsll(mask) - 1); break;
}
case ESP_SLEEP_WAKEUP_UNDEFINED: ESP_LOGI(TAG, "cold start"); break;
default: break;
}

/* 睡眠前收尾 */
rtc_gpio_isolate(GPIO_NUM_12);              /* 悬空脚隔离降漏电 */
gettimeofday(&s_sleep_enter, NULL);
s_boot_count++;
esp_deep_sleep_start();                      /* noreturn */
```

其它 API：`esp_sleep_get_wakeup_cause()`, `esp_sleep_get_gpio_wakeup_status()`, `esp_sleep_get_ext1_wakeup_status()`, `esp_sleep_get_touchpad_wakeup_status()`, `esp_sleep_disable_wakeup_source()`, `esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_ON)`, `esp_deep_sleep_try_to_start()`, `esp_wake_stub`（deep_sleep_wake_stub 示例）。

规则：

- **Deep-sleep 唤醒 = 复位重启**，`app_main` 会重新执行，普通全局/静态变量不保留；只有 `RTC_DATA_ATTR` 与 `esp_sleep_get_wakeup_cause()` 可用于区分。
- ext0 需要 RTC 外设域保持供电（会略增电流）；ext1 更省电。只有 GPIO0~21 可用。
- 睡眠前必须 `fflush` / `uart_wait_tx_done()` 把日志发完；`CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP=y` 可缩短唤醒启动时间（见 `examples/system/deep_sleep` 的 sdkconfig.defaults）。
- 记录时间戳优先用 **RTC 存储器而非 NVS**（NVS 写入耗时会让睡眠计时不准，`examples/system/deep_sleep` 明确说明）。

### 4. ULP 协处理器（Deep-sleep 中工作）

ESP32-S3 支持 **ULP-RISC-V** 与 **ULP-FSM** 二选一（`components/ulp/Kconfig` 的 `choice ULP_COPROC_TYPE`：`CONFIG_ULP_COPROC_TYPE_RISCV` / `CONFIG_ULP_COPROC_TYPE_FSM`；ESP32-S3 默认 `ULP_COPROC_TYPE_RISCV`）。ULP 可访问 RTC GPIO、RTC I2C、SAR ADC、温度传感器、RTC 寄存器，达到阈值时唤醒主 CPU。

```c
/* 主 CPU 侧（ULP-RISC-V） */
#include "ulp_riscv.h"
#include "ulp_main.h"     /* 由 ulp_embed_binary 生成的头文件 */
ESP_ERROR_CHECK(ulp_riscv_load_binary(ulp_main_bin_start,
                                      (ulp_main_bin_end - ulp_main_bin_start)));
ESP_ERROR_CHECK(ulp_riscv_config_and_run(&(ulp_riscv_cfg_t){
    .main_cpu = ULP_RISCV_RUN_MODE_WAKEUP,
    .wakeup_source = ULP_RISCV_WAKEUP_SOURCE_TIMER,
    .timer_period_us = 1000000,
}));
```

ULP 侧可用 API（`ulp_riscv.h`, `ulp_riscv_gpio.h`, `ulp_riscv_i2c.h`, `ulp_riscv_touch.h`, `ulp_riscv_uart.h`, `ulp_riscv_lock.h`, `ulp_riscv_adc.h`）：`ulp_riscv_gpio_init/output/input`, `ulp_riscv_i2c_master_init/read_from_device/write_to_device`, `ulp_riscv_touch_pad_read_benchmark/sleep_channel_read_data`, `ulp_riscv_uart_putc`, `ulp_riscv_print_str/hex`, `ulp_riscv_halt()`, `ulp_riscv_timer_stop/resume()`, `ulp_riscv_gpio_wakeup_clear()`。

构建：`ulp_embed_binary(ulp_main "${CMAKE_CURRENT_SOURCE_DIR}/ulp/main.c" "ulp_riscv")`，Kconfig `CONFIG_ULP_COPROC_ENABLED=y` + `CONFIG_ULP_COPROC_RESERVE_MEM`（`components/ulp/Kconfig` 对 ESP32-S3 默认 **4096 B**，范围 32~8176 B，需按 ULP 程序实际大小调整）。参考 `examples/system/ulp/ulp_riscv/gpio`、`ulp_fsm`、`system/deep_sleep`。

> ULP 与触摸/模拟在睡眠中同时使用时最易踩 ANALOG-160，务必保持 IDF 默认的模拟电源配置。

### 5. 安全机制（A 数据手册 4.1.4）

硬件能力：安全启动、Flash 加密（XTS-AES）、内部/外部存储器权限控制（PMS）、SHA (FIPS 180-4)、AES (FIPS 197)、RSA、HMAC、RSA-DS 数字签名、RNG、时钟毛刺检测、4096 bit eFuse（1792 bit 用户可用，支持读写保护）。

```ini
# Flash 加密（开发模式，先看日志确认再切 Release）
CONFIG_SECURE_FLASH_ENC_ENABLED=y
CONFIG_SECURE_FLASH_ENCRYPTION_MODE_DEVELOPMENT=y
CONFIG_SECURE_BOOT=y
CONFIG_SECURE_BOOT_V2_ENABLED=y
CONFIG_SECURE_BOOT_BUILD_SIGNED_BINARIES=y
```

```c
/* 硬件加速示例：SHA / RNG */
#include "esp_random.h"
#include "mbedtls/sha256.h"       /* 底层走硬件 SHA 加速器 */
uint32_t nonce = esp_random();    /* 硬件 RNG，勿用 rand() 做密钥 */
```

规则：

- **eFuse 一次性可编程**：烧成 1 不可逆。安全启动/Flash 加密相关 eFuse 烧写前必须确认固件可正常启动、有回滚/恢复方案；开发用 `..._MODE_DEVELOPMENT`，量产才用 Release。
- Flash 加密开启后，`esp_partition_write()` 自动加密、`esp_partition_read()` 自动解密；**禁止**用 `esp_partition_write_raw()` 写明文数据（技能 08）。
- 安全启动 v2 启用后，OTA 镜像必须带签名，且 `CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK` + eFuse `secure_version` 防降级；签名私钥绝不入库/入代码。
- `esp_random()` 来自硬件 RNG；`rand()` / `srand(time(NULL))` 不得用于密钥、nonce、IV。
- 密钥存 **eFuse 或加密 NVS**（`nvs_sec_provider` / HMAC-based NVS encryption），不要明文放 NVS/flash 分区。
- 调试接口：量产可烧 `EFUSE_DIS_PAD_JTAG` / `EFUSE_DIS_USB_JTAG` / `EFUSE_STRAP_JTAG_SEL` 关闭 JTAG（技能 01 表 3-5）；USB-OTG 下载用 `EFUSE_DIS_USB_OTG_DOWNLOAD_MODE`。
- 参考 `examples/security/flash_encryption`, `security_features_app`, `nvs_encryption_hmac`, `hmac_soft_jtag`。

### 6. 低功耗设计检查顺序

1. 先确认供电与 EN 上电时序（技能 01），再谈睡眠电流。
2. 选模式：需要快速恢复并保留 RAM → Light-sleep；数分钟~小时级唤醒 → Deep-sleep。
3. 关掉不用的外设时钟与电源域；悬空 GPIO 用 `rtc_gpio_isolate()`；关闭板载 LED/传感器电源（用 GPIO 控制负载开关）。
4. Wi-Fi 用 Modem-sleep（`esp_wifi_set_ps`）而非一直断连重连；BLE 广播间隔加大。
5. `CONFIG_FREERTOS_USE_TICKLESS_IDLE=y` + `esp_pm_configure()` 让空闲自动降频/浅睡。
6. 用万用表/电流计实测各模式电流，不要只按数据手册典型值估算。

## 示例

```c
/* 周期性采集 + Deep-sleep（10 s 唤醒一次） */
void app_main(void)
{
    static RTC_DATA_ATTR uint32_t boot_cnt = 0;
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
    ESP_LOGI(TAG, "boot #%" PRIu32 ", cause=%d", ++boot_cnt, cause);

    float value = read_sensor_once();          /* ADC1，见技能 05 */
    store_sample(value);                       /* RTC 内存或批量写 NVS */

    ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup(10ULL * 1000 * 1000));
    ESP_ERROR_CHECK(uart_wait_tx_done(UART_NUM_0, pdMS_TO_TICKS(100)));
    esp_deep_sleep_start();
}
```

## 检查清单

- [ ] 明确区分 Light-sleep（继续执行）与 Deep-sleep（复位重启），跨睡眠数据用 `RTC_DATA_ATTR`
- [ ] 唤醒 GPIO 在 GPIO0~21 RTC 域内，睡眠前隔离悬空引脚
- [ ] `CONFIG_PM_ENABLE` + tickless idle + `esp_pm_configure` 配置正确，长时序操作拿了 PM lock
- [ ] 未手动关闭 RTC 外设电源（RTC-126），未改动模拟电源配置（ANALOG-160）
- [ ] ULP 场景确认 `CONFIG_ULP_COPROC_*`、reserve mem 与二进制嵌入正确
- [ ] 安全启动/Flash 加密使用 Development 模式验证后再考虑量产，eFuse 烧写有恢复方案
- [ ] 密钥/随机数来自 eFuse 与 `esp_random()`，未硬编码；JTAG/USB 下载口按量产需求关闭
- [ ] 各模式电流已实测
