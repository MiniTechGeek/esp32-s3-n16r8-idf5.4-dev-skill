---
name: esp32s3-display-camera
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的 esp_lcd（SPI/I80/RGB 屏）、触摸面板、DVP 摄像头、JPEG 编解码、ISP、PPA 与帧缓冲放 8 MB PSRAM。当任务涉及驱动屏幕、显示图片、摄像头采集、AI 视觉预处理、RGB 并行屏时序或 LCD 花屏时加载本技能。
---

# ESP32-S3N16R8 · LCD / Camera / 图像

## 描述

ESP32-S3 的 LCD_CAM 控制器 + PIE 向量指令 + 8 MB Octal PSRAM 是为「屏 + 摄像头 + 轻量 AI」设计的。本技能规定 v5.4 的 `esp_lcd` / `esp_driver_cam` / JPEG / PPA 用法，以及帧缓冲必须放 PSRAM、DMA 约束与勘误 LCD-239。

## 使用场景

- SPI / I8080 / RGB 并行屏驱动、LVGL 移植、触摸面板
- OV2640/OV5640 等 DVP 摄像头采集、拍照/JPEG 输出
- JPEG 编解码、ISP、PPA 图像缩放/裁剪/格式转换
- 帧缓冲/图像缓冲分配、PSRAM 带宽优化
- 排查花屏、撕裂、帧率不足、camera 初始化失败

## 指令

### 1. 硬件能力

| 项目 | 事实（A 数据手册 / TRM 第 30 章 LCD_CAM） |
| --- | --- |
| LCD 接口 | LCD_CAM 控制器：RGB 并行、I8080 (Intel 8080)、SPI（经 SPI master）；支持 `esp_lcd` |
| Camera | DVP 8~16 位并行摄像头接口；v5.4 新增 `esp_driver_cam` (`esp_cam_ctlr_dvp.h`)，另有 CSI/ISP-DVP |
| 图像加速 | JPEG 编解码 (`esp_driver_jpeg`)、ISP (`esp_driver_isp`)、PPA (`esp_driver_ppa`) |
| CPU | PIE 128 位 SIMD/向量指令（FFT、乘加、饱和运算），适合语音/图像前处理 |
| 存储 | **8 MB Octal PSRAM** → 帧缓冲必须放这里；512 KB 内部 SRAM 放 DMA 描述符与热点数据 |
| GDMA | 5 RX + 5 TX 通道；外部 RAM 访问仅 `0x3C000000~0x3DFFFFFF` 且只支持突发 |

> **勘误 LCD-239**：RGB 模式若 `LCD_CAM_LCD_CLK_EQU_SYSCLK = 1`（分频=1），PCLK 无法设为下降沿触发，且连续帧（`NEXT_FRAME_EN=1`）时第二帧会插入上一帧最后一个数据；I8080 模式若 `ahead_cycle ≤ 2` 会导致首个数据与后续数据数量错误。**v5.4 已规避（v4.4.5+/v5.0.3+/v5.1+）**，但生成寄存器级配置时禁止再把 `LCD_CLK_EQU_SYSCLK` 置 1。

### 2. esp_lcd（v5.4 分层模型）

```
esp_lcd_panel_io_handle_t   ← 总线 IO 层（发命令/参数/像素）
esp_lcd_panel_handle_t      ← 面板层（init / draw_bitmap / mirror / invert / disp_on_off）
```

| 总线类型 | 创建 API | 头文件 |
| --- | --- | --- |
| SPI | `esp_lcd_new_panel_io_spi()` (`spi_master`) | `esp_lcd_panel_io.h`, `esp_lcd_panel_io_spi.h` |
| I80 (8080) | `esp_lcd_new_i80_bus()` → `esp_lcd_new_panel_io_i80()` | `esp_lcd_panel_io.h` |
| RGB 并行 | `esp_lcd_new_rgb_panel()` | `esp_lcd_panel_rgb.h` |
| I2C (小屏/触控) | `esp_lcd_new_panel_io_i2c_v2()` | `esp_lcd_panel_io_i2c.h` |

```c
#include "esp_lcd_panel_io.h"
#include "esp_lcd_panel_ops.h"
#include "esp_lcd_panel_vendor.h"    /* 具体 IC 的初始化命令序列 */

/* SPI 总线（技能 04 的 SPI2_HOST） */
spi_bus_config_t buscfg = {
    .mosi_io_num = GPIO_NUM_11, .miso_io_num = GPIO_NUM_13,
    .sclk_io_num = GPIO_NUM_12, .quadwp_io_num = -1, .quadhd_io_num = -1,
    .max_transfer_sz = 64 * 1024,
};
ESP_ERROR_CHECK(spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO));

esp_lcd_panel_io_spi_config_t io_cfg = {
    .dc_gpio_num = GPIO_NUM_9, .cs_gpio_num = GPIO_NUM_10,
    .pclk_hz = 20 * 1000 * 1000, .lcd_cmd_bits = 8, .lcd_param_bits = 8,
    .spi_mode = 0, .trans_queue_depth = 8,
    .on_color_trans_done = on_color_done, .user_ctx = NULL,
};
esp_lcd_panel_io_handle_t io;
ESP_ERROR_CHECK(esp_lcd_new_panel_io_spi((esp_lcd_spi_bus_handle_t)SPI2_HOST, &io_cfg, &io));

esp_lcd_panel_dev_config_t panel_cfg = {
    .reset_gpio_num = GPIO_NUM_14, .color_space = ESP_LCD_COLOR_SPACE_RGB,
    .bits_per_pixel = 16, .reset_active_high = false,
};
esp_lcd_panel_handle_t panel;
ESP_ERROR_CHECK(esp_lcd_new_st7789(io, &panel_cfg, &panel));   /* 或 new_ili9341 等 */

ESP_ERROR_CHECK(esp_lcd_panel_reset(panel));
ESP_ERROR_CHECK(esp_lcd_panel_init(panel));
ESP_ERROR_CHECK(esp_lcd_panel_invert_color(panel, true));
ESP_ERROR_CHECK(esp_lcd_panel_disp_on_off(panel, true));
ESP_ERROR_CHECK(esp_lcd_panel_draw_bitmap(panel, 0, 0, 320, 240, framebuffer));
```

其它 API：`esp_lcd_panel_io_tx_param()`, `esp_lcd_panel_io_tx_color()`, `esp_lcd_panel_io_register_event_callbacks()`, `esp_lcd_panel_io_del()`, `esp_lcd_panel_mirror()`, `esp_lcd_panel_swap_xy()`, `esp_lcd_panel_set_gap()`, `esp_lcd_panel_disp_sleep()`。

RGB 并行屏：`esp_lcd_rgb_panel_config_t`（`clk_src`, `pclk_hz`, `hsync/vsync/de_gpio_num`, `data_gpio_num[16]`, `disp_gpio_num`, `timings`, `flags.fb_in_psram=true`, `num_fbs`, `bounce_buffer_size_px`）→ `esp_lcd_new_rgb_panel()`；`esp_lcd_rgb_panel_register_event_callbacks()`, `esp_lcd_rgb_panel_set_pclk()`, `esp_lcd_rgb_panel_restart()`, `esp_lcd_rgb_panel_get_frame_buffer()`, `esp_lcd_rgb_panel_refresh()`。**必须设 `flags.fb_in_psram = true`**，否则 320×240×2×2 = 300 KB 帧缓冲放不进内部 SRAM。

参考：`examples/peripherals/lcd/spi_lcd_touch`, `lcd/rgb_lcd`, `lcd/i80_controller`, `lcd/lvgl`。

### 3. 帧缓冲与内存（N16R8 的 8 MB PSRAM 是核心优势）

```c
#include "esp_heap_caps.h"
/* RGB565 320x240 = 153600 B；双缓冲 300 KB → 必须 PSRAM */
size_t fb_size = 320 * 240 * 2;
uint8_t *fb0 = heap_caps_malloc(fb_size, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);
uint8_t *fb1 = heap_caps_malloc(fb_size, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);
if (!fb0 || !fb1) { ESP_LOGE(TAG, "PSRAM framebuffer alloc failed"); return ESP_ERR_NO_MEM; }
```

- `esp_lcd_i80_alloc_draw_buffer(io, size, MALLOC_CAP_SPIRAM | MALLOC_CAP_DMA?)` 注意：I80/RGB 的 bounce buffer 若参与 DMA，**必须在内部 SRAM**（`MALLOC_CAP_INTERNAL | MALLOC_CAP_DMA`），最终帧缓冲在 PSRAM；`esp_lcd_new_rgb_panel` 的 `bounce_buffer_size_px` 就是为解决 PSRAM 带宽不足/撕裂而设的内部中转。
- PSRAM 带宽低于内部 SRAM，高分辨率/高帧率 RGB 屏要评估：`CONFIG_SPIRAM_SPEED_80M=y`、减小 `pclk_hz`、启用 bounce buffer、双缓冲 + VSYNC 同步。
- 图像/音频/JSON 大缓冲一律 `MALLOC_CAP_SPIRAM`；DMA 描述符与 bounce buffer 一律内部 SRAM（技能 04/09）。
- `CONFIG_SPIRAM_FETCH_INSTRUCTIONS` / `CONFIG_SPIRAM_RODATA` 一般保持 `n`（XIP 场景另论，见 `examples/system/xip_from_psram`）。

### 4. 摄像头（DVP）

```c
#include "esp_cam_ctlr.h"
#include "esp_cam_ctlr_dvp.h"

esp_cam_ctlr_dvp_pin_config_t pins = {
    .data_width = 8,
    .data_io = { GPIO_NUM_11, GPIO_NUM_9, GPIO_NUM_8, GPIO_NUM_10,
                 GPIO_NUM_12, GPIO_NUM_13, GPIO_NUM_14, GPIO_NUM_15 },
    .vsync_io = GPIO_NUM_6, .href_io = GPIO_NUM_7,
    .pclk_io  = GPIO_NUM_5, .xclk_io = GPIO_NUM_4,
};
esp_cam_ctlr_handle_t cam;
esp_cam_ctlr_config_t cfg = {
    .ctlr_id = 0, .clk_src = CAM_CLK_SRC_DEFAULT,
    .h_res = 320, .v_res = 240, .data_width = 8,
    .pic_format_jpeg = true,
    .dma_burst_size = 64, .byte_swap_en = false,
};
ESP_ERROR_CHECK(esp_cam_ctlr_dvp_init(&pins, &cfg, &cam));     /* 见 esp_cam_ctlr_dvp.h */
/* esp_cam_ctlr_register_event_callbacks(cam, &cbs, arg);
   esp_cam_ctlr_start() / receive() / stop() / del() */
```

- v5.4 新组件：`esp_driver_cam`（`esp_cam_ctlr.h`, `esp_cam_ctlr_dvp.h`, `esp_cam_ctlr_csi.h`, `esp_cam_ctlr_isp_dvp.h`）、`esp_driver_isp`、`esp_driver_jpeg`、`esp_driver_ppa`。
- 摄像头配置（SCCB/I2C 写寄存器）用技能 04 的 `i2c_master`；XCLK 可由 LEDC 或 CAM 控制器提供。
- 帧缓冲放 PSRAM；DVP 的 DMA 描述符由驱动管理，但 bounce/对齐缓冲仍需内部 SRAM。
- 参考 `examples/peripherals/camera/camera_dsi`（v5.4 示例目录为 `camera_dsi`）、`esp_driver_cam/test_apps`。

### 5. JPEG / ISP / PPA

| 组件 | 头文件 | 主要 API |
| --- | --- | --- |
| JPEG 编解码 | `driver/jpeg_encode.h`, `jpeg_decode.h`, `jpeg_types.h` | `jpeg_new_encoder_engine()`, `jpeg_encoder_process()`, `jpeg_new_decoder_engine()`, `jpeg_decoder_process()`, `jpeg_del_*()` |
| ISP | `driver/isp_core.h`, `isp_ae.h`, `isp_af.h`, `isp_awb.h`, `isp_bf.h`, `isp_ccm.h`, `isp_color.h`, `isp_demosaic.h`, `isp_gamma.h`, `isp_hist.h`, `isp_lsc.h`, `isp_sharpen.h` | `esp_isp_new_processor()`, `esp_isp_enable()/disable()`, `esp_isp_*` 各模块 |
| PPA | `driver/ppa.h`, `ppa_types.h` | `ppa_new_srm/ppa_new_blend/ppa_new_fill()`, `ppa_do_scale_rotate_mirror()`, `ppa_do_blend()`, `ppa_do_fill()` |
| LCD 面板 | `esp_lcd_panel_ops.h`, `esp_lcd_panel_rgb.h` | 见第 2 节 |

- PPA 做硬件缩放/旋转/混合/填充，避免 CPU 软件循环；输入输出都在 PSRAM 时效率最高，但 DMA 描述符仍受内部 RAM 约束。
- JPEG 编码输出 JPEG 流，可直接存 SD/FATFS（技能 08）或经 HTTP 上传（技能 06）。
- PIE 向量指令（TRM 第 1 章）提供 128 位 SIMD、FFT、饱和乘加，写自定义图像/音频算子时优先用 intrinsics，而不是标量 C 循环。

### 6. 常见故障

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 花屏/错位 | 引脚映射错、pclk 过高、`bits_per_pixel`/color_space 与面板不符 | 核对数据线顺序、`pclk_hz`、`ESP_LCD_COLOR_SPACE_*` |
| 撕裂 | 单缓冲 + 无 VSYNC 同步 | 双缓冲 + `esp_lcd_rgb_panel_register_event_callbacks()` 的 vsync 回调，或 bounce buffer |
| 帧率低 | 帧缓冲在 PSRAM 带宽不足 / SPI 时钟低 | 提高 `SPIRAM_SPEED_80M`、用 I80/RGB 并行、加大 bounce buffer |
| `MALLOC_CAP_SPIRAM` 分配失败 | `CONFIG_SPIRAM` 未开或配成 Quad | 技能 02：`CONFIG_SPIRAM=y` + `CONFIG_SPIRAM_MODE_OCT=y` |
| camera 初始化失败 | XCLK 没输出 / SCCB(I2C) 不通 / 引脚与 flash/PSRAM 冲突 | 检查 XCLK、I2C ACK、引脚黑名单（技能 01） |
| RGB 屏第二帧串数据 | 勘误 LCD-239（寄存器级分频=1） | 用 `esp_lcd` 高层 API（v5.4 已规避），禁止手写 `LCD_CLK_EQU_SYSCLK=1` |

## 示例

```c
/* 双缓冲刷新（示意）：在 VSYNC 回调里切换，避免撕裂 */
static void IRAM_ATTR on_vsync(esp_lcd_panel_handle_t panel,
                                const esp_lcd_rgb_panel_event_data_t *edata, void *user_ctx)
{
    /* 只置标志，实际绘制放任务里 —— ISR 最小化 */
    BaseType_t hp = pdFALSE;
    xTaskNotifyFromISR(s_render_task, 1, eSetBits, &hp);
    if (hp) portYIELD_FROM_ISR();
}

void app_main(void)
{
    esp_lcd_rgb_panel_config_t cfg = {
        .clk_src = LCD_CLK_SRC_DEFAULT, .pclk_hz = 8 * 1000 * 1000,
        .h_res = 320, .v_res = 240, .data_width = 16,
        .num_fbs = 2, .flags.fb_in_psram = true,     /* 关键 */
        .bounce_buffer_size_px = 320 * 8,            /* 内部 SRAM 中转 */
        /* ... hsync/vsync/de/pclk/data_gpio_num ... */
    };
    esp_lcd_panel_handle_t panel;
    ESP_ERROR_CHECK(esp_lcd_new_rgb_panel(&cfg, &panel));
    ESP_ERROR_CHECK(esp_lcd_panel_init(panel));
    /* esp_lcd_rgb_panel_get_frame_buffer(panel, 2, &fb0, &fb1); */
}
```

## 检查清单

- [ ] 帧缓冲用 `MALLOC_CAP_SPIRAM` 分配并检查 NULL，RGB 配置 `flags.fb_in_psram=true`
- [ ] DMA/bounce 缓冲在内部 SRAM（`MALLOC_CAP_INTERNAL | MALLOC_CAP_DMA`）
- [ ] SPI/I80/RGB 引脚通过技能 01 黑名单校验，未占用 flash/PSRAM/USB/UART0
- [ ] 未手写 `LCD_CAM_LCD_CLK_EQU_SYSCLK=1`（LCD-239）
- [ ] 高分辨率场景评估了 PSRAM 带宽、bounce buffer 与双缓冲/VSYNC 同步
- [ ] 摄像头 XCLK、SCCB(I2C) 与 DVP 数据线逐一核对
- [ ] 大图像/音频/JSON 缓冲放 PSRAM，热点数据放内部 SRAM
