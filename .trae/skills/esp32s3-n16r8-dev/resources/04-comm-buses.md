---
name: esp32s3-comm-buses
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的 UART、I2C(master/slave)、SPI(master/slave)、I2S、TWAI(CAN)、SD/MMC/SDSPI、USB Serial/JTAG、USB OTG 与 GDMA 约束。当任务涉及串口通信、传感器总线、SPI 屏/Flash、音频 I2S、CAN 总线、SD 卡、USB 设备或 DMA 缓冲分配时加载本技能。
---

# ESP32-S3N16R8 · 通信总线与 USB

## 描述

覆盖 v5.4 的总线驱动 API、初始化顺序、DMA 缓冲约束与 N16R8 特有的引脚冲突（USB D−/D+ 在 GPIO19/20、UART0 在 GPIO43/44、Octal PSRAM 占用 GPIO33~37）。

## 使用场景

- UART/RS485、I2C 传感器、SPI 屏或 SD 卡、I2S 音频、TWAI/CAN
- USB 虚拟串口、TinyUSB 设备、USB 日志/下载通道
- DMA 传输报 `invalid descriptor` / 数据错乱
- 总线初始化顺序、超时、错误恢复

## 指令

### 1. 组件与 API 速查（v5.4）

| 总线 | 头文件 | API 主干 | 示例 |
| --- | --- | --- | --- |
| UART | `driver/uart.h` | `uart_driver_install`, `uart_param_config`, `uart_set_pin`, `uart_write_bytes`, `uart_read_bytes`, `uart_wait_tx_done`, `uart_flush_input`, `uart_enable_pattern_det_baud_intr`, `uart_set_mode`(RS485), `uart_vfs_dev_use_driver` | `peripherals/uart/uart_async_rxtxtasks`, `uart_echo` |
| I2C master | `driver/i2c_master.h` | `i2c_new_master_bus`, `i2c_master_bus_add_device`, `i2c_master_transmit`, `i2c_master_transmit_receive`, `i2c_master_receive`, `i2c_master_probe`, `i2c_master_bus_reset`, `i2c_del_master_bus` | `peripherals/i2c/i2c_eeprom` |
| I2C slave | `driver/i2c_slave.h` | `i2c_new_slave_device`, `i2c_slave_receive`, `i2c_slave_transmit`, `i2c_slave_read_ram`, `i2c_slave_write_ram`, `i2c_slave_register_event_callbacks` | `peripherals/i2c/i2c_slave_network_sensor` |
| SPI master | `driver/spi_master.h` | `spi_bus_initialize`, `spi_bus_add_device`, `spi_device_transmit`, `spi_device_queue_trans` + `spi_device_get_trans_result`, `spi_device_acquire_bus`, `spi_bus_dma_memory_alloc` | `peripherals/spi_master/lcd` |
| SPI slave / HD | `driver/spi_slave.h`, `driver/spi_slave_hd.h` | `spi_slave_initialize`, `spi_slave_queue_trans`, `spi_slave_transmit`, `spi_slave_hd_*` | `peripherals/spi_slave`, `spi_slave_hd` |
| I2S | `driver/i2s_std.h`, `i2s_pdm.h`, `i2s_tdm.h`, `i2s_common.h` | `i2s_new_channel`, `i2s_channel_init_std_mode`, `i2s_channel_enable`, `i2s_channel_write/read`, `i2s_channel_register_event_callback` | `peripherals/i2s/i2s_std`, `i2s_codec` |
| TWAI | `driver/twai.h` | `twai_driver_install`, `twai_start`, `twai_transmit`, `twai_receive`, `twai_read_alerts`, `twai_stop`, `twai_driver_uninstall` | `peripherals/twai/twai_network` |
| SD/MMC | `driver/sdmmc_host.h`, `sdmmc_cmd.h`, `esp_vfs_fat.h` | `sdmmc_host_init`, `sdmmc_host_init_slot`, `sdmmc_card_init`, `esp_vfs_fat_sdmmc_mount` | `storage/sd_card` |
| SDSPI | `driver/sdspi_host.h` | `sdspi_host_init`, `sdspi_host_init_device`, `esp_vfs_fat_sdspi_mount` | `peripherals/sd_card/sdspi` |
| USB Serial/JTAG | `driver/usb_serial_jtag.h` | `usb_serial_jtag_driver_install`, `usb_serial_jtag_read_bytes`, `usb_serial_jtag_write_bytes`, `usb_serial_jtag_vfs_register`, `usb_serial_jtag_is_connected` | `peripherals/usb_serial_jtag/usb_serial_jtag_echo` |
| USB OTG | `tinyusb.h`, `tusb_cdc_acm.h`, `tusb_console.h`, `tusb_msc_storage.h` | `tinyusb_driver_install`, `tusb_cdc_acm_init`, `tinyusb_cdcacm_register_callback` (TinyUSB 组件由组件管理器引入) | `peripherals/usb/device/tusb_console` |

**禁止**：`driver/i2c.h` 旧 `i2c_param_config/i2c_driver_install/i2c_master_write_read_device`；`driver/i2s.h` 旧 `i2s_driver_install/i2s_set_pin`；`driver/spi_common.h` 里旧宏。v5.4 用上表新接口。

### 2. UART（3 个控制器：UART0/1/2）

```c
#include "driver/uart.h"
#define UART_NUM  UART_NUM_1
#define TXD_PIN   GPIO_NUM_17   /* U1TXD P1 固定功能，见技能 01 表 2-9 */
#define RXD_PIN   GPIO_NUM_18   /* U1RXD P1 */

uart_config_t cfg = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
    .parity    = UART_PARITY_DISABLE,
    .stop_bits = UART_STOP_BITS_1,
    .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
    .source_clk = UART_SCLK_DEFAULT,
};
ESP_ERROR_CHECK(uart_driver_install(UART_NUM, 2048, 2048, 16, &uart_queue, ESP_INTR_FLAG_LEVEL2));
ESP_ERROR_CHECK(uart_param_config(UART_NUM, &cfg));
ESP_ERROR_CHECK(uart_set_pin(UART_NUM, TXD_PIN, RXD_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE));

const char *msg = "ping\n";
int n = uart_write_bytes(UART_NUM, msg, strlen(msg));   /* 返回实际写入字节数 */
uint8_t buf[128];
int len = uart_read_bytes(UART_NUM, buf, sizeof(buf), pdMS_TO_TICKS(100)); /* 必须给超时 */
```

规则：

- `uart_read_bytes()` 必须给有限超时，禁止在任务里无界阻塞；配合 `uart_get_buffered_data_len()` 做流控。
- `uart_driver_install()` 的 RX/TX 缓冲区与队列大小按波特率计算，115200 下 2×2048 B + 16 事件够用；高波特率/大包要放大。
- UART0（GPIO43/44）默认被 ROM 日志与 console 占用；用 UART0 做业务前先在 menuconfig 切 console 到 USB Serial/JTAG。
- RS485：`uart_set_mode(UART_MODE_RS485_HALF_DUPLEX)` + `uart_set_rts()`；`uart_enable_pattern_det_baud_intr()` 可做 AT 指令/帧尾检测。
- `uart_vfs_dev_use_driver()` 可把 UART 挂到 VFS，用 `open/read/write` 访问。
- 所有 `uart_*` 返回值都要检查；`uart_wait_tx_done()` 在改变引脚复用/进入睡眠前调用。

### 3. I2C master（v5.4 新 API，重点）

```c
#include "driver/i2c_master.h"

i2c_master_bus_config_t bus_cfg = {
    .clk_source          = I2C_CLK_SRC_DEFAULT,
    .i2c_port           = -1,                 /* 自动分配端口 */
    .scl_io_num         = GPIO_NUM_9,
    .sda_io_num         = GPIO_NUM_8,
    .glitch_ignore_cnt  = 7,                   /* 硬件毛刺过滤 */
    .flags.enable_internal_pullup = true,
};
i2c_master_bus_handle_t bus;
ESP_ERROR_CHECK(i2c_new_master_bus(&bus_cfg, &bus));

i2c_device_config_t dev_cfg = {
    .dev_addr_length = I2C_ADDR_BIT_LEN_7,
    .device_address  = 0x50,
    .scl_speed_hz    = 400000,                 /* 400 kHz Fast mode */
};
i2c_master_dev_handle_t dev;
ESP_ERROR_CHECK(i2c_master_bus_add_device(bus, &dev_cfg, &dev));

uint8_t reg = 0x00, data[2] = {0};
/* 写寄存器地址再读：一次 combined transaction */
ESP_ERROR_CHECK(i2c_master_transmit_receive(dev, &reg, 1, data, sizeof(data), -1));
/* 探测设备：i2c_master_probe(bus, addr, -1); 总线挂死：i2c_master_bus_reset(bus); */
```

规则：

- **SCL/SDA 必须由外部上拉**（典型 4.7 kΩ @ 400 kHz），内部上拉较弱，仅作补充。
- `xfer_timeout_ms` 用有限值或 `-1`（由驱动默认）；任务里不要用无限等待。
- 总线挂死时 `i2c_master_bus_reset()`，并在硬件上确认 SDA 未被从机拉死；必要时用 `i2c_common_set_pins()` 重新映射。
- 同一 bus 上挂多个 device 用 `i2c_master_bus_add_device()` 添加多个 handle；释放顺序：先 `i2c_master_bus_rm_device()` 再 `i2c_del_master_bus()`。
- 回调：`i2c_master_register_event_callbacks()`（异步传输完成事件）。
- 引脚必须避开技能 01 黑名单；**GPIO19/20 是 USB，不要拿来做 I2C**。

### 4. SPI master（SPI2_HOST / SPI3_HOST）

```c
#include "driver/spi_master.h"

spi_bus_config_t buscfg = {
    .mosi_io_num = GPIO_NUM_11,
    .miso_io_num = GPIO_NUM_13,
    .sclk_io_num = GPIO_NUM_12,
    .quadwp_io_num = -1,
    .quadhd_io_num = -1,
    .max_transfer_sz = 4096,      /* DMA 时按最大帧设 */
};
ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));

spi_device_interface_config_t devcfg = {
    .clock_speed_hz = 20 * 1000 * 1000,   /* 20 MHz */
    .mode = 0,
    .spics_io_num = GPIO_NUM_10,
    .queue_size = 4,
    .flags = SPI_DEVICE_HALFDUPLEX,   /* 按需 */
};
spi_device_handle_t spi;
ESP_ERROR_CHECK(spi_bus_add_device(SPI2_HOST, &devcfg, &spi));

spi_transaction_t t = {
    .length = 8 * 2,
    .tx_buffer = tx,
    .rx_buffer = rx,       /* 全双工时 rx_buffer 有效；半双工用 rxlength */
};
ESP_ERROR_CHECK(spi_device_transmit(spi, &t));
```

规则：

- **SPI0/1 已被 flash/PSRAM 占用，只能用 `SPI2_HOST` 和 `SPI3_HOST`**。禁止指定 `SPI1_HOST`。
- 使用 DMA（`SPI_DMA_CH_AUTO`）时，`max_transfer_sz` 必须覆盖最大单次传输；传输缓冲必须内部 SRAM + DMA 能力，用 `spi_bus_dma_memory_alloc()` 或 `heap_caps_malloc(size, MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL)` 分配，**不能放 PSRAM**。
- 高频（>20 MHz）注意 GPIO 矩阵延迟，优先用 IO_MUX 固定引脚（SPI2 推荐 GPIO9~14 一组 FSPI 功能，见技能 01 表 2-9）。
- 多设备共享总线时 `spi_device_acquire_bus()` / `spi_device_release_bus()` 包裹连续事务，避免 CS 抖动。
- 异步用 `spi_device_queue_trans()` + `spi_device_get_trans_result()`，队列满了要处理背压。
- `spi_bus_remove_device()` → `spi_bus_free()` 顺序释放。

### 5. I2S / TWAI / SD 卡（要点）

- **I2S**：`i2s_new_channel(&i2s_chan_config_t, &tx, &rx)` → `i2s_channel_init_std_mode()`（或 `pdm_tx/rx`, `tdm`）→ `i2s_channel_enable()` → `i2s_channel_write/read()`。标准模式配置 `i2s_std_config_t` 含 `clk_cfg.sample_rate_hz`（如 44100）、`slot_cfg`（`I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(16, I2S_SLOT_MODE_STEREO)`）、`gpio_cfg`（MCLK/BCLK/WS/DOUT/DIN）。大音频缓冲放 PSRAM，但 DMA 描述符链由驱动管理；`i2s_channel_register_event_callback()` 用 `on_recv`/`on_sent` 事件驱动，避免轮询。
- **TWAI**：`twai_general_config_t`（`TWAI_GENERAL_CONFIG_DEFAULT(tx, rx, TWAI_MODE_NORMAL)`）+ `twai_timing_config_t`（`TWAI_TIMING_CONFIG_500KBITS()`）+ `twai_filter_config_t`（`TWAI_FILTER_CONFIG_ACCEPT_ALL()`）→ `twai_driver_install()` → `twai_start()` → `twai_transmit(&msg, pdMS_TO_TICKS(100))` / `twai_receive()` / `twai_read_alerts()`。TWAI 兼容 ISO 11898-1 CAN 2.0；**只有 1 个控制器**，且无 CAN-FD。
- **SD/MMC 主机**：仅 1 个 host，**不能作 SDIO slave**（硬件指南 1.3.12）。所有 SDIO GPIO 需外部上拉并预留串阻。`sdmmc_host_init()` → `sdmmc_host_init_slot(SLOT, &sdmmc_slot_config_t)` → `sdmmc_card_init()` → `esp_vfs_fat_sdmmc_mount("/sdcard", ...)`；卸载用 `esp_vfs_fat_sdcard_unmount()`（旧 `esp_vfs_fat_sdmmc_unmount` 已废弃）。SDSPI 用 `sdspi_host_init_device()` + `esp_vfs_fat_sdspi_mount()`，占用一个 SPI host。

### 6. USB

| 接口 | 引脚 | Kconfig / 用途 |
| --- | --- | --- |
| USB Serial/JTAG | GPIO19 = D−, GPIO20 = D+ | `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` 走日志；`usb_serial_jtag_driver_install()` 做 CDC-like 字节流；`usb_serial_jtag_vfs_register()` 挂 VFS |
| USB OTG 1.1 Full-Speed | 内部 PHY 同上；外部 PHY 用 VP/VM/RCV/OEN/VPO/VMO 固定脚 | TinyUSB 组件（`esp_tinyusb`），例 `tusb_console` |

```c
#include "driver/usb_serial_jtag.h"
usb_serial_jtag_driver_config_t cfg = {
    .rx_buffer_size = 1024,
    .tx_buffer_size = 1024,
};
ESP_ERROR_CHECK(usb_serial_jtag_driver_install(&cfg));
/* 阻塞读需给超时，注意 usb_serial_jtag_is_connected() 判断主机是否打开串口 */
int n = usb_serial_jtag_read_bytes(buf, sizeof(buf), pdMS_TO_TICKS(100));
usb_serial_jtag_write_bytes(msg, strlen(msg));
```

规则：

- GPIO19/20 上电有毛刺且默认启用 USB 功能；要当普通 GPIO 必须先禁用 USB（IO_MUX 的 `USB_SERIAL_JTAG_DP/DM_PULLUP` 等），并接受上电毛刺对外部电路的影响。
- **勘误 USBOTG-4289**：Date Code < 2219 / PW < PW-2022-06-XXXX 的批次 `EFUSE_DIS_USB_OTG_DOWNLOAD_MODE` 默认烧 1，USB-OTG 下载不可用 → 下载一律优先 **USB-Serial-JTAG 或 UART0**。
- 启用 USB-OTG Download Boot 时 ROM 会初始化外部 PHY 的 IO pad（VP/VM/RCV 输入，OEN 输出高，VPO/VMO 输出低）；不需要该功能就烧 `EFUSE_DIS_USB_OTG_DOWNLOAD_MODE` 避免 IO 状态被改。
- `USB_SERIAL_JTAG_EXCHG_PINS` 位可交换 D+/D− 引脚（硬件布线需要时）。

### 7. GDMA 硬约束（所有 DMA 外设通用）

- 5 RX + 5 TX 通道，与 CPU 共用地址空间。
- 内部 RAM 可访问区：`0x3FC88000 ~ 0x3FCFFFFF`（480 KB）。
- 外部 RAM 可访问区：`0x3C000000 ~ 0x3DFFFFFF`（32 MB），**只支持突发模式**，块大小 16/32/64 B。
- **描述符必须在内部 RAM**；描述符里的 buffer 指针若在非法区会触发 `GDMA_ETXMEM_REJECT_INT`，可通过 `GDMA_ETXMEM_REJECT_ADDR` 等寄存器定位。
- 因此：DMA 源/目的缓冲用 `heap_caps_malloc(size, MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL)`；大帧数据的**最终存放区**可放 PSRAM，但需先 DMA 到内部缓冲再拷贝，或使用驱动提供的 `*_dma_memory_alloc()`。
- IDF 驱动（spi_master、i2s、adc_continuous、sdmmc、parlio、camera）通常已处理描述符；自己写 ETM/GDMA 直驱时才需手工管理 inlink/outlink。

## 示例

```c
/* 组合事务：I2C 读 16 位寄存器，带有限超时与错误处理 */
static esp_err_t reg16_read(i2c_master_dev_handle_t dev, uint8_t reg, uint16_t *out)
{
    uint8_t buf[2] = {0};
    esp_err_t ret = i2c_master_transmit_receive(dev, &reg, 1, buf, 2, pdMS_TO_TICKS(100));
    if (ret != ESP_OK) {
        ESP_LOGE("i2c", "reg 0x%02x read failed: %s", reg, esp_err_to_name(ret));
        return ret;                    /* 不静默吞错 */
    }
    *out = ((uint16_t)buf[0] << 8) | buf[1];
    return ESP_OK;
}
```

## 检查清单

- [ ] 使用 v5.4 新 API，未出现 `i2c_driver_install` / `i2s_driver_install` 等 legacy
- [ ] UART0/USB/JTAG/strapping 引脚未与业务总线冲突
- [ ] SPI 只用 `SPI2_HOST` / `SPI3_HOST`；DMA 缓冲在内部 SRAM 且带 `MALLOC_CAP_DMA`
- [ ] I2C 有外部上拉，`glitch_ignore_cnt` 已设，传输带有限超时
- [ ] 所有阻塞读/写都有超时与返回值检查，总线异常有 reset/恢复路径
- [ ] USB 下载优先走 USB-Serial-JTAG（USBOTG-4289）
- [ ] SD 卡场景确认只用 1 个 SD/MMC host 且加了上拉电阻
