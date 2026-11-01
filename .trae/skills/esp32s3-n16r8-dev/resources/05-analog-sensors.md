---
name: esp32s3-analog-sensors
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的 ADC 单次/连续/校准/监控、ADC2 硬件限制、触摸传感器、温度传感器与模拟比较器。当任务涉及模拟量采集、电池电压检测、NTC/光敏、触摸按键、芯片温度、ADC 读数不准或 DMA 采样卡死时加载本技能。
---

# ESP32-S3N16R8 · ADC / 触摸 / 模拟传感

## 描述

规定 N16R8 上模拟采集的正确姿势：**优先 ADC1**，ADC2 受勘误 ADC-183 限制不得用 DMA；触摸需跳过前两帧无效数据（TOUCH-100）；所有读数需校准才有电压意义。

## 使用场景

- 单次/连续 ADC 采样、电压换算、校准曲线、阈值监控
- 电池电压分压检测、NTC/光敏/电流采样
- 电容触摸按键/滑条、触摸唤醒
- 芯片内部温度传感器、模拟比较器
- 排查「ADC2 卡死」「触摸第一次读数乱跳」「ADC 值不准」

## 指令

### 1. 通道映射（技能 01 已列，此处为速查）

| 单元 | 通道 → GPIO |
| --- | --- |
| **ADC1（推荐）** | CH0=GPIO1, CH1=GPIO2, CH2=GPIO3, CH3=GPIO4, CH4=GPIO5, CH5=GPIO6, CH6=GPIO7, CH7=GPIO8, CH8=GPIO9, CH9=GPIO10 |
| ADC2（受限） | CH0=GPIO11, CH1=GPIO12, CH2=GPIO13, CH3=GPIO14, CH4=GPIO15, CH5=GPIO16, CH6=GPIO17, CH7=GPIO18, CH8=GPIO19(USB D−), CH9=GPIO20(USB D+) |
| Touch | TOUCH1=GPIO1 … TOUCH14=GPIO14（仅 **GPIO14/TOUCH14** 可驱动屏蔽电极） |

- 2 个 12 位 SAR ADC，共 20 通道。
- **ADC2 与 Wi-Fi 共用硬件**：Wi-Fi 工作时 ADC2 读数不可靠；且勘误 ADC-183 导致 DIG ADC2 控制器（DMA/连续模式）可能收到伪采样使能进入死锁 → **ADC2 只能用 RTC 控制器单次读，禁止 `adc_continuous`**。
- GPIO19/20 是 USB D−/D+，尽量不要用作 ADC2_CH8/CH9。
- 硬件：每个 ADC 输入对地加 **0.1 µF** 滤波电容；走线远离数字信号与 RF。
- 衰减档与精度（硬件指南 1.3.11，校准后总误差）：`ATTEN=0` → 0~850 mV ±5 mV；`ATTEN=1` → 0~1100 mV ±6 mV；`ATTEN=2` → 0~1600 mV ±10 mV；`ATTEN=3` → 0~2900 mV ±50 mV。v5.4 用 `ADC_ATTEN_DB_0/2_5/6/12` 等枚举，不要直接写 0/1/2/3。

### 2. 单次采样（oneshot）+ 校准（标准写法）

```c
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

#define ADC_PIN   GPIO_NUM_4         /* ADC1_CH3 */
#define ADC_UNIT  ADC_UNIT_1
#define ADC_CHAN  ADC_CHANNEL_3

static bool adc_calibration_init(adc_unit_t unit, adc_channel_t chan, adc_atten_t atten,
                                 adc_cali_handle_t *out_handle)
{
    adc_cali_handle_t handle = NULL;
    esp_err_t ret = ESP_FAIL;
#if ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED
    adc_cali_curve_fitting_config_t cfg = {
        .unit_id = unit, .chan = chan, .atten = atten, .bitwidth = ADC_BITWIDTH_DEFAULT,
    };
    ret = adc_cali_create_scheme_curve_fitting(&cfg, &handle);
#elif ADC_CALI_SCHEME_LINE_FITTING_SUPPORTED
    adc_cali_line_fitting_config_t cfg = {
        .unit_id = unit, .atten = atten, .bitwidth = ADC_BITWIDTH_DEFAULT,
    };
    ret = adc_cali_create_scheme_line_fitting(&cfg, &handle);
#endif
    if (ret == ESP_OK) { *out_handle = handle; return true; }
    ESP_LOGW("adc", "calibration scheme unavailable: %s", esp_err_to_name(ret));
    return false;                       /* 无校准时只用 raw 值做相对判断 */
}

void app_main(void)
{
    adc_oneshot_unit_handle_t adc1;
    adc_oneshot_unit_init_cfg_t init_cfg = { .unit_id = ADC_UNIT_1 };
    ESP_ERROR_CHECK(adc_oneshot_new_unit(&init_cfg, &adc1));

    adc_oneshot_chan_cfg_t chan_cfg = {
        .atten    = ADC_ATTEN_DB_12,     /* 0~2900 mV 档 */
        .bitwidth = ADC_BITWIDTH_DEFAULT,
    };
    ESP_ERROR_CHECK(adc_oneshot_config_channel(adc1, ADC_CHAN, &chan_cfg));

    adc_cali_handle_t cali;
    bool do_cali = adc_calibration_init(ADC_UNIT_1, ADC_CHAN, ADC_ATTEN_DB_12, &cali);

    int raw, voltage_mv;
    ESP_ERROR_CHECK(adc_oneshot_read(adc1, ADC_CHAN, &raw));
    if (do_cali) {
        ESP_ERROR_CHECK(adc_cali_raw_to_voltage(cali, raw, &voltage_mv));
        ESP_LOGI("adc", "raw=%d, voltage=%d mV", raw, voltage_mv);
    }
    /* 释放：adc_cali_delete_scheme_*() → adc_oneshot_del_unit() */
}
```

要点：

- `adc_oneshot_new_unit()` → `adc_oneshot_config_channel()` → `adc_oneshot_read()` → `adc_oneshot_del_unit()`；**无须** `adc_oneshot_io_to_channel()` 时可自行映射，但推荐用它把 GPIO 转成 unit/channel，避免映射表写错。
- `adc_oneshot_get_calibrated_result(unit, cali, chan, &mV)` 可一步拿校准电压。
- 校准方案由 `adc_cali_scheme.h` 的宏决定：`ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED`（eFuse 有校准值时）或 `LINE_FITTING`；用 `adc_cali_scheme_line_fitting_check_efuse()` 预检。
- 无校准数据时不要伪造电压，明确记录「仅 raw 相对值」。
- 电池分压检测：分压电阻比 + 衰减档要匹配，输入阻抗与采样保持时间会影响精度，必要时加电压跟随器或延长采样；**不要**让 GPIO 电压超过 `VDDA`。

### 3. 连续采样（DMA，仅 ADC1）

```c
#include "esp_adc/adc_continuous.h"

adc_continuous_handle_t handle;
adc_continuous_handle_cfg_t adc_config = {
    .max_store_buf_size = 1024,
    .conv_frame_size    = 256,       /* 必须是 SOC_ADC_DIGI_DATA_BYTES_PER_CONV 的倍数 */
    .flags.flush_pool  = true,
};
ESP_ERROR_CHECK(adc_continuous_new_handle(&adc_config, &handle));

adc_continuous_config_t dig_cfg = {
    .sample_freq_hz = 20 * 1000,     /* 20 kSPS，勿超过硬件能力 */
    .conv_mode      = ADC_CONV_SINGLE_UNIT_1,
    .format         = ADC_DIGI_OUTPUT_FORMAT_TYPE1,
};
adc_digi_pattern_config_t pattern[] = {
    { .atten = ADC_ATTEN_DB_12, .channel = ADC_CHANNEL_3,
      .unit = ADC_UNIT_1, .bit_width = ADC_BITWIDTH_DEFAULT },
};
dig_cfg.pattern_num = 1;
dig_cfg.adc_pattern = pattern;
ESP_ERROR_CHECK(adc_continuous_config(handle, &dig_cfg));

adc_continuous_evt_cbs_t cbs = { .on_conv_done = conv_done_cb, .on_pool_ovf = pool_ovf_cb };
ESP_ERROR_CHECK(adc_continuous_register_event_callbacks(handle, &cbs, NULL));
ESP_ERROR_CHECK(adc_continuous_start(handle));

uint8_t buf[256]; uint32_t out_len;
ESP_ERROR_CHECK(adc_continuous_read(handle, buf, sizeof(buf), &out_len, pdMS_TO_TICKS(100)));
/* 解析 adc_digi_output_data_t（type1：unit/channel/raw_data） */
/* adc_continuous_stop() → adc_continuous_deinit() */
```

规则：

- **只能用 `ADC_CONV_SINGLE_UNIT_1`**（ADC1）；使用 ADC2 的连续/DMA 模式会触发勘误 ADC-183 的死锁。
- `adc_continuous_read()` 必须给有限超时；`on_pool_ovf` 回调说明内部池溢出 → 提高读取频率或减小 `sample_freq_hz`/增大 `max_store_buf_size`。
- 数据解析用 `esp_adc/adc_digi_pattern_config_t` 与 `adc_digi_output_data_t` 的定义，按 `ADC_DIGI_OUTPUT_FORMAT_TYPE1` 解析；不要硬编码位域。
- 连续模式缓冲由驱动管理（内部 RAM + GDMA），**不需要**用户分配 DMA buffer；但 `conv_frame_size` 要按 `SOC_ADC_DIGI_DATA_BYTES_PER_CONV` 对齐。
- 监控/滤波：`adc_continuous_monitor_*`（阈值监视）、`adc_new_iir_filter()` + `adc_continuous_iir_filter_enable()`（IIR 平滑）。

### 4. 触摸传感器（v5.4 新 driver）

```c
#include "driver/touch_sensor.h"
#include "driver/touch_sensor_types.h"

touch_sensor_handle_t sens;
touch_sensor_config_t scfg = {
    /* 采样配置：见 touch_sensor_sample_config_t，用 TOUCH_SENSOR_DEFAULT_* 宏初始化 */
};
ESP_ERROR_CHECK(touch_sensor_new_controller(&scfg, &sens));

touch_channel_handle_t ch;
touch_channel_config_t ccfg = { .channel_num = TOUCH_CHAN_NUM_4, /* GPIO4 */
                                /* .abs_thresh / .charge_speed 等 */ };
ESP_ERROR_CHECK(touch_sensor_new_channel(sens, &ccfg, &ch));
ESP_ERROR_CHECK(touch_sensor_enable(sens));

uint32_t raw;
ESP_ERROR_CHECK(touch_sensor_channel_read_data(ch, TOUCH_DATA_TYPE_RAW, &raw));
/* touch_sensor_channel_read_data(..., TOUCH_DATA_TYPE_SMOOTH / BENCHMARK, ...) */
/* 回调：touch_sensor_register_callbacks(sens, &cbs, arg) */
```

规则：

- **勘误 TOUCH-100**：前两次 `TOUCH_SCAN_DONE_INT` 中断的原始数据未定义 → **丢弃前两次采样结果再启用判断**，之后关闭该中断或不再使用其 raw；驱动不一定替你过滤。
- 触摸通道 = GPIO1~14（TOUCH1~14）；仅 GPIO14 可驱动屏蔽电极；串 470 Ω~2 kΩ（推荐 510 Ω）抑制噪声。
- **触摸传感器未通过传导抗扰度 (CS) 测试**，工业强干扰场景慎用；防水/数字滤波特性可用。
- 触摸可作 Deep-sleep 唤醒（`esp_sleep_enable_touchpad_wakeup()`，技能 10），也可用 ULP-RISC-V 的 `ulp_riscv_touch_pad_*` 系列在睡眠中读数。
- 使用 `touch_sensor_v3` 示例（资料包 `examples/peripherals/touch_sensor/touch_sensor_v3`）作为模板，不要用旧 `driver/touch_pad.h`。

### 5. 片内温度传感器与模拟比较器

```c
#include "driver/temperature_sensor.h"
temperature_sensor_handle_t tsens;
temperature_sensor_config_t cfg = TEMPERATURE_SENSOR_CONFIG_DEFAULT(-10, 110);
ESP_ERROR_CHECK(temperature_sensor_install(&cfg, &tsens));
ESP_ERROR_CHECK(temperature_sensor_enable(tsens));
float celsius;
ESP_ERROR_CHECK(temperature_sensor_get_celsius(tsens, &celsius));  /* 芯片结温，非环境温度 */
ESP_ERROR_CHECK(temperature_sensor_disable(tsens));
ESP_ERROR_CHECK(temperature_sensor_uninstall(tsens));
/* temperature_sensor_set_absolute_threshold() / register_callbacks() 可做过温中断 */
```

- 温度传感器测的是**芯片内部结温**，用于过温保护/校准补偿，不要当环境温度传感器宣传。
- 模拟比较器：`driver/ana_cmpr.h`（`ana_cmpr_new_unit()`, `ana_cmpr_set_intl_ref()`, `ana_cmpr_register_event_callbacks()`, `ana_cmpr_enable()`），`driver/ana_cmpr_etm.h` 可接 ETM；参考 `examples/peripherals/analog_comparator`。
- ULP 侧可用 `ulp_riscv_touch_pad_*`、`ulp_riscv_i2c_master_*` 在 Deep-sleep 中低功耗采样（技能 10）。

## 示例

```c
/* 带滑窗平均与校准的电池电压读取（ADC1_CH3 / GPIO4，衰减 12 dB） */
#define DIV_RATIO  2.0f     /* 外部 100k/100k 分压 */
static int battery_mv(adc_oneshot_unit_handle_t unit, adc_cali_handle_t cali)
{
    int raw = 0, mv = 0;
    esp_err_t ret = adc_oneshot_read(unit, ADC_CHANNEL_3, &raw);
    if (ret != ESP_OK) { ESP_LOGE("bat", "adc read: %s", esp_err_to_name(ret)); return -1; }
    ret = adc_cali_raw_to_voltage(cali, raw, &mv);
    if (ret != ESP_OK) { ESP_LOGE("bat", "cali: %s", esp_err_to_name(ret)); return -1; }
    return (int)(mv * DIV_RATIO);
}
```

## 检查清单

- [ ] 模拟输入落在 ADC1（GPIO1~10）；未对 ADC2 使用 `adc_continuous`/DMA
- [ ] 有 0.1 µF 对地滤波；衰减档与输入电压范围匹配，未超 VDDA
- [ ] 使用 `adc_cali_*` 校准；无校准数据时明确输出 raw 而非伪造电压
- [ ] 连续采样只用 `ADC_CONV_SINGLE_UNIT_1`，`conv_frame_size` 对齐，读取带超时且处理 `on_pool_ovf`
- [ ] 触摸跳过前两帧无效数据（TOUCH-100），串 510 Ω 电阻，已知 CS 测试限制
- [ ] 温度值标注为芯片结温；所有 `esp_err_t` 均已检查
