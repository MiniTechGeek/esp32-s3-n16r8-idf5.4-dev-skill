---
name: esp32s3-gpio-timer-pwm
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的 GPIO/中断/毛刺过滤、RTC GPIO、gptimer、esp_timer、LEDC、RMT、PCNT、MCPWM 与任务看门狗的标准用法。当任务涉及按键/LED/中断、PWM 调光或舵机、WS2812 灯带、红外遥控、编码器计数、电机控制、精确定时或喂狗时加载本技能。
---

# ESP32-S3N16R8 · GPIO / 定时器 / PWM / 脉冲外设

## 描述

覆盖 v5.4 新一代 driver API（非 legacy），重点是与硬件约束对齐：ISR 最小化、引脚合法性、勘误 RMT-176、双核下的中断分配与栈。

## 使用场景

- 配置 GPIO 输入输出、上下拉、开漏、驱动能力、中断与 ISR 服务
- 按键消抖、旋转编码器、脉冲计数、WS2812/红外收发
- PWM（LEDC）、电机/舵机（MCPWM）、精确定时（gptimer/esp_timer）
- 任务看门狗（TWDT）与喂狗策略
- Deep-sleep 唤醒 GPIO（RTC GPIO）

## 指令

### 1. 组件与头文件（v5.4，禁止 legacy）

| 功能 | 头文件 | CMake `REQUIRES` |
| --- | --- | --- |
| GPIO / RTC IO / 毛刺过滤 / 专用 GPIO | `driver/gpio.h`, `driver/rtc_io.h`, `driver/gpio_filter.h`, `driver/dedic_gpio.h`, `driver/gpio_etm.h` | `esp_driver_gpio` |
| 通用定时器 | `driver/gptimer.h`, `driver/gptimer_types.h`, `driver/gptimer_etm.h` | `esp_driver_gptimer` |
| 高精度软件定时器 | `esp_timer.h` | `esp_timer` |
| LED PWM | `driver/ledc.h` | `esp_driver_ledc` |
| RMT | `driver/rmt_tx.h`, `driver/rmt_rx.h`, `driver/rmt_encoder.h`, `driver/rmt_types.h` | `esp_driver_rmt` |
| 脉冲计数器 | `driver/pulse_cnt.h` | `esp_driver_pcnt` |
| MCPWM | `driver/mcpwm_prelude.h`, `driver/mcpwm_timer.h`, `driver/mcpwm_oper.h`, `driver/mcpwm_cmpr.h`, `driver/mcpwm_gen.h`, `driver/mcpwm_fault.h`, `driver/mcpwm_cap.h`, `driver/mcpwm_sync.h` | `esp_driver_mcpwm` |
| 任务看门狗 | `esp_task_wdt.h` | `esp_system` |

**禁止**：`driver/timer.h` / `driver/timer_group.h`（旧 timer_group）、`driver/rmt.h`（旧 RMT）、`driver/mcpwm.h`（旧 MCPWM）、`driver/pcnt.h`（旧 PCNT）、`driver/ledc.h` 旧宏 `LEDC_TIMER_0` 配合 `timer_group` 的写法。

### 2. GPIO 标准流程

```c
#include "driver/gpio.h"
#include "driver/gpio_filter.h"

/* 输出 */
gpio_config_t out_cfg = {
    .pin_bit_mask = (1ULL << GPIO_NUM_38),
    .mode         = GPIO_MODE_OUTPUT,
    .pull_up_en   = GPIO_PULLUP_DISABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type    = GPIO_INTR_DISABLE,
};
ESP_ERROR_CHECK(gpio_config(&out_cfg));
ESP_ERROR_CHECK(gpio_set_level(GPIO_NUM_38, 1));

/* 输入 + 下降沿中断 */
gpio_config_t in_cfg = {
    .pin_bit_mask = (1ULL << GPIO_NUM_4),
    .mode         = GPIO_MODE_INPUT,
    .pull_up_en   = GPIO_PULLUP_ENABLE,
    .intr_type    = GPIO_INTR_NEGEDGE,
};
ESP_ERROR_CHECK(gpio_config(&in_cfg));

/* ISR 服务只需安装一次；ISR 内只投递队列/任务通知 */
static QueueHandle_t gpio_evt_queue;
static void IRAM_ATTR gpio_isr_handler(void *arg)
{
    uint32_t gpio_num = (uint32_t)arg;
    BaseType_t hp = pdFALSE;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, &hp);
    if (hp == pdTRUE) portYIELD_FROM_ISR();
}
ESP_ERROR_CHECK(gpio_install_isr_service(ESP_INTR_FLAG_IRAM | ESP_INTR_FLAG_LEVEL3));
ESP_ERROR_CHECK(gpio_isr_handler_add(GPIO_NUM_4, gpio_isr_handler, (void *)GPIO_NUM_4));
```

硬性规则：

- `gpio_install_isr_service()` 全局只调一次；多个脚用 `gpio_isr_handler_add()` 分别挂。
- ISR 函数加 `IRAM_ATTR`（放内部 RAM，避免 flash cache 关闭期间取指失败）；回调内**禁止** `ESP_LOGx`、`vTaskDelay`、`malloc`、阻塞 API。
- 共享 ISR 标志建议 `ESP_INTR_FLAG_SHARED | ESP_INTR_FLAG_LEVEL2/3`；高实时中断用 `ESP_INTR_FLAG_IRAM` 且回调里的所有数据与函数都必须在 IRAM。
- 引脚必须先过技能 01 的黑名单校验；`gpio_reset_pin()` 用于释放复用。
- 未用引脚显式配置上下拉或 `gpio_sleep_sel_en()` 以避免悬空功耗；上电毛刺见技能 01（GPIO1~14、17~20 约 60 µs 低/高电平毛刺，GPIO19/20 上电两次高电平毛刺总计 3.2 ms / 2 ms）。

硬件毛刺过滤（比软件延时可靠）：

```c
gpio_pin_glitch_filter_config_t cfg = {
    .clk_src = GLITCH_FILTER_CLK_SRC_DEFAULT,
    .gpio_num = GPIO_NUM_4,
};
gpio_glitch_filter_handle_t flt;
ESP_ERROR_CHECK(gpio_new_pin_glitch_filter(&cfg, &flt));
ESP_ERROR_CHECK(gpio_glitch_filter_enable(flt));
```

柔性过滤器 `gpio_new_flex_glitch_filter()` 可设窗口/阈值；`gpio_del_glitch_filter()` 释放。

Deep-sleep 唤醒相关：`rtc_gpio_init()`, `rtc_gpio_set_direction()`, `rtc_gpio_pullup_en/dis()`, `rtc_gpio_hold_en()`, `rtc_gpio_isolate()`, `rtc_gpio_wakeup_enable()`, `gpio_deep_sleep_wakeup_enable()`；**只有 VDD3P3_RTC 电源域的 GPIO（GPIO0~21）能在 Deep-sleep 中保持/唤醒**。

### 3. gptimer（4 个 54 位通用定时器）

```c
#include "driver/gptimer.h"

gptimer_handle_t gptimer = NULL;
gptimer_config_t timer_config = {
    .clk_src       = GPTIMER_CLK_SRC_DEFAULT,   /* APB 默认 80 MHz */
    .direction     = GPTIMER_COUNT_UP,
    .resolution_hz = 1 * 1000 * 1000,           /* 1 µs / tick */
    .intr_priority = 0,
};
ESP_ERROR_CHECK(gptimer_new_timer(&timer_config, &gptimer));

gptimer_alarm_config_t alarm_config = {
    .alarm_count  = 1000000,   /* 1 s */
    .reload_count = 0,
    .flags.auto_reload_on_alarm = true,
};
ESP_ERROR_CHECK(gptimer_set_alarm_action(gptimer, &alarm_config));

gptimer_event_callbacks_t cbs = { .on_alarm = on_alarm_cb };  /* cb 在 ISR 上下文 */
ESP_ERROR_CHECK(gptimer_register_event_callbacks(gptimer, &cbs, user_ctx));
ESP_ERROR_CHECK(gptimer_enable(gptimer));
ESP_ERROR_CHECK(gptimer_start(gptimer));
/* 停止/释放：gptimer_stop() → gptimer_disable() → gptimer_del_timer() */
```

其它 API：`gptimer_set_raw_count()`, `gptimer_get_raw_count()`, `gptimer_get_captured_count()`, `gptimer_get_resolution()`, `gptimer_set_alarm_action()`, `gptimer_acquire_group_handle()`, `gptimer_select_periph_clock()`, `gptimer_new_etm_event/task()`（与 ETM 事件矩阵联动，可完全硬件触发 GPIO）。

### 4. esp_timer（微秒级一次性/周期软件定时器）

```c
#include "esp_timer.h"
static void timer_cb(void *arg) { /* 回调在 esp_timer 任务上下文，不是 ISR */ }

const esp_timer_create_args_t args = {
    .callback = &timer_cb,
    .arg = NULL,
    .dispatch_method = ESP_TIMER_TASK,   /* 或 ESP_TIMER_ISR（回调必须极短） */
    .name = "periodic",
    .skip_unhandled_events = true,      /* 防止事件堆积 */
};
esp_timer_handle_t h;
ESP_ERROR_CHECK(esp_timer_create(&args, &h));
ESP_ERROR_CHECK(esp_timer_start_periodic(h, 1000));   /* 1000 µs */
/* esp_timer_stop() / esp_timer_delete() / esp_timer_dump() */
```

规则：回调默认在专用 timer 任务里，可调用多数非阻塞 API；`ESP_TIMER_ISR` 分发时同样受 ISR 约束。周期必须 > 时钟粒度，长时间任务用 FreeRTOS 任务而非 esp_timer 回调。Light-sleep 下 esp_timer 会唤醒芯片，需与功耗策略配合（技能 10）。

### 5. LEDC（8 通道 PWM）

```c
#include "driver/ledc.h"

ledc_timer_config_t tcfg = {
    .speed_mode      = LEDC_LOW_SPEED_MODE,
    .timer_num       = LEDC_TIMER_0,
    .duty_resolution = LEDC_TIMER_13_BIT,
    .freq_hz         = 5000,               /* 5 kHz；分辨率与频率相互制约 */
    .clk_cfg         = LEDC_AUTO_CLK,
    .deconfigure     = false,
};
ESP_ERROR_CHECK(ledc_timer_config(&tcfg));

ledc_channel_config_t ccfg = {
    .gpio_num   = GPIO_NUM_38,
    .speed_mode = LEDC_LOW_SPEED_MODE,
    .channel    = LEDC_CHANNEL_0,
    .timer_sel  = LEDC_TIMER_0,
    .duty       = 0,
    .hpoint     = 0,
    .intr_type  = LEDC_INTR_DISABLE,
};
ESP_ERROR_CHECK(ledc_channel_config(&ccfg));

/* 渐变需先装服务 */
ESP_ERROR_CHECK(ledc_fade_func_install(ESP_INTR_FLAG_IRAM));
ESP_ERROR_CHECK(ledc_set_fade_time_and_start(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0,
                                             4096, 500, LEDC_FADE_NO_WAIT));
/* 或立即：ledc_set_duty() + ledc_update_duty() */
```

`ledc_find_suitable_duty_resolution()` 可按源时钟与目标频率反算最大分辨率；`ledc_set_freq()`, `ledc_stop()`, `ledc_cb_register()`, `ledc_timer_pause/resume()` 亦可用。注意高分辨率与高频率不可兼得（APB 80 MHz 下 13 bit @ 5 kHz 可行，20 kHz 时分辨率下降）。

### 6. RMT（TX/RX，含 WS2812 / 红外）

```c
#include "driver/rmt_tx.h"
#include "driver/rmt_encoder.h"

rmt_tx_channel_config_t tx_cfg = {
    .gpio_num       = GPIO_NUM_38,
    .clk_src        = RMT_CLK_SRC_DEFAULT,
    .resolution_hz  = 10 * 1000 * 1000,   /* 10 MHz → 100 ns/tick */
    .mem_block_symbols = 64,
    .trans_queue_depth = 4,
    .intr_priority  = 0,
};
rmt_channel_handle_t tx_chan;
ESP_ERROR_CHECK(rmt_new_tx_channel(&tx_cfg, &tx_chan));

rmt_copy_encoder_config_t copy_cfg = {};
rmt_encoder_handle_t copy_enc;
ESP_ERROR_CHECK(rmt_new_copy_encoder(&copy_cfg, &copy_enc));
ESP_ERROR_CHECK(rmt_enable(tx_chan));
/* rmt_transmit(tx_chan, copy_enc, symbols, len, &tx_config); rmt_tx_wait_all_done(tx_chan, pdMS_TO_TICKS(100)); */
```

其它：`rmt_new_bytes_encoder()`, `rmt_new_simple_encoder()`, `rmt_new_rx_channel()`, `rmt_receive()`, `rmt_rx_register_event_callbacks()`, `rmt_new_sync_manager()`（多路同步）, `rmt_apply_carrier()`（38 kHz 红外载波）, `rmt_select_periph_clock()`。

> **勘误 RMT-176**：连续 TX 停止后 idle 电平由回卷数据决定而非 end-marker。ESP-IDF v5.0 起（含 v5.4）驱动已强制 `RMT_IDLE_OUT_EN_CHn=1` 用寄存器控制 idle 电平，**不要手动把该位改回 0**。

WS2812 推荐用组件管理器引入 `espressif/led_strip`（见 `examples/get-started/blink` 的 `idf_component.yml`），其 RMT 后端默认 10 MHz、无 DMA。

### 7. PCNT 与 MCPWM（要点）

- PCNT：`pcnt_new_unit()` → `pcnt_new_channel()` → `pcnt_channel_set_edge_action()` / `set_level_action()` → `pcnt_unit_add_watch_point()` → `pcnt_unit_enable/start()` → `pcnt_unit_get_count()`；配 `pcnt_unit_set_glitch_filter()` 做硬件消抖。参考 `examples/peripherals/pcnt/rotary_encoder`。
- MCPWM：分层对象 `mcpwm_new_timer()` → `mcpwm_new_operator()` → `mcpwm_operator_connect_timer()` → `mcpwm_new_comparator()` → `mcpwm_new_generator()` → `mcpwm_generator_set_action_on_*_event()` → `mcpwm_timer_enable/start_stop()`；故障刹车用 `mcpwm_new_gpio_fault()` + `mcpwm_generator_set_action_on_fault_event()`；死区用 `mcpwm_generator_set_dead_time()`。参考 `examples/peripherals/mcpwm/mcpwm_bdc_speed_control`。
- 两者都可接 **ETM**（`*_new_etm_event/task`）实现纯硬件联动，避开 ISR 抖动。

### 8. 任务看门狗 TWDT

```c
#include "esp_task_wdt.h"

esp_task_wdt_config_t twdt_cfg = {
    .timeout_ms     = 3000,
    .idle_core_mask = (1 << 0) | (1 << 1),  /* 双核都监控 idle 任务 */
    .trigger_panic  = true,
};
ESP_ERROR_CHECK(esp_task_wdt_init(&twdt_cfg));
ESP_ERROR_CHECK(esp_task_wdt_add(NULL));      /* 订阅当前任务 */
/* 循环里定期：ESP_ERROR_CHECK(esp_task_wdt_reset()); */
/* esp_task_wdt_add_user() / reset_user() 用于非任务上下文 */
```

规则：不要在 ISR 里 `esp_task_wdt_reset()`；长任务必须按最坏执行时间设 timeout；`esp_task_wdt_status()`, `esp_task_wdt_print_triggered_tasks()` 用于诊断（技能 12）。Kconfig：`CONFIG_ESP_TASK_WDT_EN=y`, `CONFIG_ESP_TASK_WDT_USE_ESP_TIMER`。

## 示例

```c
/* 按键中断 → 队列 → 任务处理（ISR 只做投递） */
static QueueHandle_t evt_q;
static void IRAM_ATTR isr(void *arg) {
    uint32_t pin = (uint32_t)arg;
    BaseType_t hp = pdFALSE;
    xQueueSendFromISR(evt_q, &pin, &hp);
    if (hp) portYIELD_FROM_ISR();
}
static void button_task(void *arg) {
    uint32_t pin;
    while (1) {
        if (xQueueReceive(evt_q, &pin, portMAX_DELAY) == pdTRUE) {
            ESP_LOGI("btn", "press on GPIO %" PRIu32, pin);  /* 日志放任务里 */
        }
    }
}
void app_main(void) {
    evt_q = xQueueCreate(8, sizeof(uint32_t));
    assert(evt_q);
    gpio_config_t cfg = { .pin_bit_mask = 1ULL << GPIO_NUM_4,
                          .mode = GPIO_MODE_INPUT, .pull_up_en = GPIO_PULLUP_ENABLE,
                          .intr_type = GPIO_INTR_NEGEDGE };
    ESP_ERROR_CHECK(gpio_config(&cfg));
    ESP_ERROR_CHECK(gpio_install_isr_service(ESP_INTR_FLAG_LEVEL2));
    ESP_ERROR_CHECK(gpio_isr_handler_add(GPIO_NUM_4, isr, (void *)GPIO_NUM_4));
    xTaskCreate(button_task, "button", 3072, NULL, 5, NULL);   /* 栈需实测高水位 */
}
```

## 检查清单

- [ ] 全部使用 v5.4 `esp_driver_*` 新 API，无 legacy 头
- [ ] 引脚通过技能 01 黑名单校验
- [ ] ISR 加 `IRAM_ATTR`，只做 `FromISR` 投递，无日志/延时/分配
- [ ] `gpio_install_isr_service()` 只调用一次
- [ ] PWM 频率与占空比分辨率相互兼容；RMT 分辨率按 100 ns 量级换算 symbol
- [ ] 未触碰 RMT idle 电平寄存器（RMT-176）
- [ ] TWDT 订阅与喂狗覆盖所有长任务，`timeout_ms` 大于最坏执行时间
- [ ] 任务栈用 `uxTaskGetStackHighWaterMark()` 实测过（技能 09）
