---
name: esp32s3-freertos-core
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的 FreeRTOS 双核任务架构、栈高水位与核亲和性、队列/信号量/事件组、堆与 8 MB PSRAM 分配、esp_event、日志与错误处理。当任务涉及创建任务、任务间通信、内存不足、缓冲区放 PSRAM、双核同步、esp_event 或代码规范审查时加载本技能。
---

# ESP32-S3N16R8 · FreeRTOS / 内存 / 系统机制

## 描述

把「不会崩」的固件约束固化下来：静态优先、栈实测、ISR 最小化、所有返回值检查、大缓冲放 PSRAM 而 DMA 缓冲放内部 SRAM。

## 使用场景

- 创建/设计任务、设置优先级与核亲和、栈大小评估
- 队列、信号量、互斥量、事件组、任务通知、流/消息缓冲
- `malloc` 失败、堆碎片、把大数组放 8 MB PSRAM
- 双核共享数据同步、优先级反转、死锁
- `esp_event` 事件循环、日志级别、`esp_err_t` 错误处理
- 启动流程与 `app_main` 的职责

## 指令

### 1. 硬件与 RTOS 基线

| 项目 | 事实 |
| --- | --- |
| CPU | 双核 Xtensa LX7，最高 240 MHz（`CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y`） |
| 内部 RAM | 512 KB SRAM（其中 480 KB GDMA 可访问区 `0x3FC88000~0x3FCFFFFF`） |
| 外部 RAM | **8 MB Octal PSRAM**（`CONFIG_SPIRAM_MODE_OCT=y`），映射 `0x3C000000~0x3DFFFFFF` |
| RTC RAM | 16 KB（8 KB FAST + 8 KB SLOW），Deep-sleep 保持 |
| tick | `CONFIG_FREERTOS_HZ`（默认 100，实时场景建议 1000）；`configUSE_PREEMPTION=y` |
| SMP | IDF 的 FreeRTOS 支持双核 SMP，`xTaskCreatePinnedToCore()` 可绑核，`tskNO_AFFINITY` 不绑 |
| 主任务 | `app_main` 跑在 `main` 任务，`CONFIG_ESP_MAIN_TASK_STACK_SIZE`（建议 8192） |
| 看门狗 | 任务看门狗 TWDT（技能 03）、中断看门狗、RTC/系统 WDT |

### 2. 任务创建与栈

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define TASK_STACK 4096          /* 单位：字节（IDF 的 FreeRTOS 栈深度按字节算） */
#define TASK_PRIO  5

static TaskHandle_t s_task_hdl;

static void sensor_task(void *arg)
{
    sensor_data_t data;
    while (1) {
        if (read_sensor(&data) == ESP_OK) {
            /* 队列满时有限等待，不能无限阻塞 */
            if (xQueueSend(s_queue, &data, pdMS_TO_TICKS(10)) != pdTRUE) {
                ESP_LOGW(TAG, "queue full, drop sample");
            }
        }
        /* 运行期监控栈：必须实测，不能靠猜 */
        ESP_LOGD(TAG, "stack HWM: %" PRIu32 " B",
                 (uint32_t)uxTaskGetStackHighWaterMark(NULL) * sizeof(StackType_t));
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void app_main(void)
{
    s_queue = xQueueCreate(8, sizeof(sensor_data_t));
    if (s_queue == NULL) { ESP_LOGE(TAG, "queue create failed"); return; }
    /* 绑核到 APP_CPU (core 1)，把 PRO_CPU (core 0) 留给 Wi-Fi/BLE 协议栈 */
    BaseType_t ok = xTaskCreatePinnedToCore(sensor_task, "sensor", TASK_STACK,
                                             NULL, TASK_PRIO, &s_task_hdl, 1);
    if (ok != pdPASS) { ESP_LOGE(TAG, "task create failed"); return; }
}
```

规则：

- **栈大小必须实测**：`uxTaskGetStackHighWaterMark(NULL)`（当前任务）或 `uxTaskGetStackHighWaterMark2()`，保留 ≥25% 余量；`CONFIG_FREERTOS_USE_TRACE_FACILITY` / `CONFIG_FREERTOS_GENERATE_RUN_TIME_STATS` 可看全局。
- `xTaskCreatePinnedToCore()` 的栈深度单位是**字节**（与原生 FreeRTOS 的 word 不同，IDF 已统一）；`vTaskDelay(pdMS_TO_TICKS(ms))` 别直接传毫秒。
- 优先用**静态分配**：`xTaskCreateStaticPinnedToCore()` + `StaticTask_t` + `StackType_t[]`，`xQueueCreateStatic()` 等；`CONFIG_FREERTOS_SUPPORT_STATIC_ALLOCATION=y`。
- **不要在任务运行期反复 `malloc/free`**：用内存池、静态缓冲或启动时一次性分配；碎片在 512 KB 内部 RAM 上很致命。
- 核心分工建议：PRO_CPU(0) 跑 Wi-Fi/BLE 协议栈与系统回调，APP_CPU(1) 跑业务逻辑；高实时任务可绑核并用 `ESP_INTR_FLAG_IRAM`。
- 禁止在任务里 `portMAX_DELAY` 无界阻塞而无超时策略（网络/传感器异常时会触发 TWDT）；必须给有限 ticks 或配合 TWDT 订阅与喂狗。
- `app_main` 应只做初始化 + 建任务，**不要**在 `app_main` 里跑无限业务循环；`app_main` 返回后 main 任务会被删除，但已创建的任务继续运行。

### 3. 任务间通信与同步

| 机制 | 适用 | 关键 API |
| --- | --- | --- |
| 队列 | 传数据、生产者→消费者 | `xQueueCreate`, `xQueueSend`, `xQueueReceive`, `xQueueSendFromISR`, `xQueueOverwrite`, `xQueuePeek` |
| 二进制/计数信号量 | 事件通知、资源计数 | `xSemaphoreCreateBinary`, `xSemaphoreGive/Take`, `xSemaphoreGiveFromISR` |
| 互斥量 | 共享资源（含优先级继承） | `xSemaphoreCreateMutex`, `xSemaphoreTake/Give`；**不能**在 ISR 用 |
| 递归互斥量 | 可能重入的临界区 | `xSemaphoreCreateRecursiveMutex` |
| 事件组 | 多条件组合（如 Wi-Fi 连接 + SNTP 完成） | `xEventGroupCreate`, `xEventGroupSetBits`, `xEventGroupWaitBits`, `...FromISR`（IDF 支持） |
| 任务通知 | 轻量单接收者事件，比信号量快 | `xTaskNotify`, `xTaskNotifyWait`, `xTaskNotifyGive`, `ulTaskNotifyTake`, `...FromISR` |
| 流/消息缓冲 | 字节流 / 变长消息 | `xStreamBufferCreate`, `xMessageBufferCreate` |
| 环形缓冲 | 大数据块、多生产者 | `esp_ringbuf` 组件 (`xRingbufferCreate`, `xRingbufferSend`, `xRingbufferReceive`) |

- **ISR 只能用 `...FromISR()` 变体**，且要处理 `BaseType_t xHigherPriorityTaskWoken` + `portYIELD_FROM_ISR()`。
- 保护共享数据：短临界区用 `taskENTER_CRITICAL(&spinlock)`（IDF SMP 用 `portMUX_TYPE` + `portENTER_CRITICAL`）；长临界区用互斥量，注意**优先级反转**与死锁（按固定顺序获取多把锁）。
- 队列/信号量创建失败要检查返回值（`NULL`），不要直接解引用。

### 4. 堆与 PSRAM（N16R8 有 8 MB，用对地方）

```c
#include "esp_heap_caps.h"
#include "esp_psram.h"

/* 大缓冲（图像/音频/JSON）→ PSRAM */
uint8_t *frame = heap_caps_malloc(320 * 240 * 2, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);
if (!frame) { ESP_LOGE(TAG, "PSRAM alloc failed"); return ESP_ERR_NO_MEM; }

/* DMA 缓冲 → 内部 SRAM + DMA 能力，绝不能放 PSRAM */
uint8_t *dma_buf = heap_caps_malloc(4096, MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL);
if (!dma_buf) { ESP_LOGE(TAG, "DMA alloc failed"); return ESP_ERR_NO_MEM; }

/* 优先内部、不够再退回 PSRAM */
uint8_t *p = heap_caps_malloc_prefer(8192, 2,
                MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT,
                MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);

ESP_LOGI(TAG, "PSRAM=%s size=%zu, free internal=%" PRIu32 ", free PSRAM=%zu",
         esp_psram_is_initialized() ? "OK" : "NO", esp_psram_get_size(),
         esp_get_free_heap_size(), heap_caps_get_free_size(MALLOC_CAP_SPIRAM));
```

| caps | 用途 |
| --- | --- |
| `MALLOC_CAP_INTERNAL` | 内部 512 KB SRAM，CPU 访问最快 |
| `MALLOC_CAP_DMA` | DMA 可用（须与 `INTERNAL` 组合） |
| `MALLOC_CAP_SPIRAM` | 8 MB Octal PSRAM（**不可用于 DMA 描述符/缓冲**） |
| `MALLOC_CAP_8BIT` / `MALLOC_CAP_32BIT` | 按对齐要求 |
| `MALLOC_CAP_EXEC` | 可执行（IROM/IRAM） |
| `MALLOC_CAP_IRAM_8BIT` | ISR 里要访问的数据/函数 |

其它：`heap_caps_malloc_extmem_enable(limit)` 设自动使用 PSRAM 的阈值；`heap_caps_check_integrity_all()`, `heap_caps_dump_all()`, `esp_get_minimum_free_heap_size()` 用于诊断；`heap_caps_get_largest_free_block()` 判断碎片。

Kconfig：`CONFIG_SPIRAM_USE_MALLOC=y`（`malloc` 自动可用 PSRAM）、`CONFIG_SPIRAM_USE_CAPS_ALLOC=y`、`CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y`（把 Wi-Fi/LwIP 缓冲挪到 PSRAM 省内部 RAM）、`CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY=n`（默认栈留内部）。

> **GDMA 约束（技能 01/04）**：外部 RAM 仅在 `0x3C000000~0x3DFFFFFF` 且只支持突发模式；描述符必须在内部 RAM。DMA 缓冲放 PSRAM 会触发 `GDMA_ETXMEM_REJECT_INT`。

### 5. esp_event（事件循环）

```c
#include "esp_event.h"
ESP_EVENT_DEFINE_BASE(MY_EVENTS);
enum { MY_EVENT_CONNECTED, MY_EVENT_DATA };

ESP_ERROR_CHECK(esp_event_loop_create_default());
esp_event_handler_instance_t inst;
ESP_ERROR_CHECK(esp_event_handler_instance_register(MY_EVENTS, MY_EVENT_DATA,
                                                     on_data, NULL, &inst));
/* 发送（可带数据，默认拷贝） */
esp_event_post(MY_EVENTS, MY_EVENT_DATA, &payload, sizeof(payload), pdMS_TO_TICKS(100));
/* 释放 */
ESP_ERROR_CHECK(esp_event_handler_instance_unregister(MY_EVENTS, MY_EVENT_DATA, inst));
ESP_ERROR_CHECK(esp_event_loop_delete_default());
```

- 系统默认事件循环（`esp_event_loop_create_default()`）被 Wi-Fi/IP/MQTT 等共用，**全局只创建一次**（见技能 06）。
- `esp_event_post_to()` 投递到自定义循环；`esp_event_handler_register_with()` 注册到自定义循环。
- 事件回调运行在事件循环任务上下文，**不是 ISR**；回调里避免长耗时（会阻塞其它事件，如 Wi-Fi 断连处理）。
- Kconfig：`CONFIG_ESP_SYSTEM_EVENT_QUEUE_SIZE`, `CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE`（默认 2304，回调复杂要加大）。

### 6. 日志与错误处理

```c
static const char *TAG = "my_module";
ESP_LOGE(TAG, "failed: %s", esp_err_to_name(err));   /* 宏自动加换行 */
ESP_LOGW(TAG, "retry %d", n);
ESP_LOGI(TAG, "heap=%" PRIu32, esp_get_free_heap_size());
esp_log_level_set("wifi", ESP_LOG_WARN);              /* 按 tag 调级别 */
```

```c
/* 致命初始化路径 */
ESP_ERROR_CHECK(esp_wifi_init(&cfg));                 /* 失败 abort 并打印 0x... 错误码 */
/* 运行期路径：不 abort，返回错误码让上层决定 */
esp_err_t ret = sensor_read(&v);
if (ret != ESP_OK) { ESP_LOGE(TAG, "read: %s", esp_err_to_name(ret)); return ret; }
```

规则：

- 所有 `esp_err_t` 必须检查；`ESP_ERROR_CHECK()` 只用于**不可恢复的初始化**，运行期用 `ESP_ERROR_CHECK_WITHOUT_ABORT()` 或显式判断。
- 用 `esp_err_to_name()` 把错误码转字符串，不要直接打印数字。
- `ESP_LOGx` 不要在 ISR 或高频循环里无节流调用；`CONFIG_LOG_DEFAULT_LEVEL`、`esp_log_level_set()`、`CONFIG_LOG_COLORS`、`CONFIG_LOG_TIMESTAMP_SOURCE` 可调。
- `ESP_LOG_BUFFER_HEXDUMP_LEVEL()` 用于调试二进制数据。
- panic 处理：`CONFIG_ESP_SYSTEM_PANIC_PRINT_REBOOT`（默认）/ `PANIC_GDBSTUB` / `PANIC_PRINT_HALT`；`CONFIG_ESP_SYSTEM_PANIC_REBOOT_DELAY_SECONDS`。
- 系统信息：`esp_chip_info()`, `esp_get_idf_version()`, `esp_restart()`, `esp_reset_reason()`, `esp_get_free_heap_size()`, `esp_get_minimum_free_heap_size()`, `esp_mac.h` 的 `esp_read_mac()`。

## 示例

```c
/* 静态任务 + 静态队列：无运行期动态分配，栈可实测 */
static StackType_t  s_stack[4096 / sizeof(StackType_t)];
static StaticTask_t s_tcb;
static uint8_t      s_q_storage[8 * sizeof(uint32_t)];
static StaticQueue_t s_q_tcb;
static QueueHandle_t s_q;

void app_main(void)
{
    s_q = xQueueCreateStatic(8, sizeof(uint32_t), s_q_storage, &s_q_tcb);
    assert(s_q);
    TaskHandle_t h = xTaskCreateStaticPinnedToCore(worker, "worker",
                           sizeof(s_stack), NULL, 5, s_stack, &s_tcb, 1);
    assert(h);
}
```

## 检查清单

- [ ] 每个任务栈用 `uxTaskGetStackHighWaterMark()` 实测且余量 ≥25%
- [ ] 优先静态分配；运行期无反复 malloc/free；队列创建检查 NULL
- [ ] ISR 只用 `FromISR` API 并 `portYIELD_FROM_ISR()`，无日志/延时/分配
- [ ] 无 `portMAX_DELAY` 无界阻塞；长任务订阅 TWDT 并定期喂狗
- [ ] 大缓冲用 `MALLOC_CAP_SPIRAM`，DMA 缓冲用 `MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL`
- [ ] 双核共享数据用互斥量/临界区，锁顺序固定，无优先级反转
- [ ] 默认事件循环只创建一次；事件回调不做长耗时
- [ ] 所有 `esp_err_t` 已检查，日志统一 `TAG` 且不在 ISR/高频循环滥用
