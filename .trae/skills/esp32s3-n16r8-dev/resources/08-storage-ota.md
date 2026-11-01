---
name: esp32s3-storage-ota
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的 NVS、分区表 API、SPIFFS/FATFS/磨损均衡、OTA 升级与回滚。当任务涉及参数持久化、配置存储、文件系统、Flash 读写、固件升级、OTA 失败回滚或 NVS 报错时加载本技能。
---

# ESP32-S3N16R8 · 存储 / NVS / 文件系统 / OTA

## 描述

N16R8 有 **16 MB Quad SPI flash**（本技能的主战场）与 8 MB Octal PSRAM（易失，不做持久化）。所有分区必须落在 16 MB 内，写入必须遵守 erase-before-write 与 4 KB 对齐。

## 使用场景

- 保存 Wi-Fi 凭据、设备序列号、运行计数、校准参数
- 直接读写 flash 分区、制作自定义数据分区
- SPIFFS / FATFS（含 SD 卡）、磨损均衡、大文件
- OTA 固件升级、HTTPS OTA、回滚与防回滚
- NVS 报 `NO_FREE_PAGES` / `NEW_VERSION_FOUND` / 写入失败
- Flash 加密后读写异常

## 指令

### 1. 组件与头文件

| 功能 | 头文件 | `REQUIRES` | 示例 |
| --- | --- | --- | --- |
| NVS | `nvs_flash.h`, `nvs.h` | `nvs_flash` | `storage/nvs_rw_value`, `nvs_rw_blob`, `nvsgen` |
| 分区 API | `esp_partition.h` | `esp_partition`, `spi_flash` | `storage/partition_api/partition_ops` |
| Flash 底层 | `esp_flash.h`, `spi_flash_mmap.h` | `spi_flash` | `storage/custom_flash_driver`, `partition_api` |
| SPIFFS | `esp_spiffs.h` | `spiffs` | `storage/spiffs` |
| FATFS (flash) | `esp_vfs_fat.h`, `esp_vfs.h`, `wear_levelling.h` | `fatfs`, `wear_levelling`, `vfs` | `storage/fatfs/ext_flash` |
| FATFS (SD) | `esp_vfs_fat.h`, `driver/sdmmc_host.h` | `fatfs`, `esp_driver_sdmmc`, `sdmmc`, `vfs` | `storage/sd_card` |
| OTA | `esp_ota_ops.h`, `esp_app_desc.h`, `esp_https_ota.h` | `app_update`, `esp_https_ota`, `esp_partition` | `system/ota/simple_ota_example`, `native_ota_example` |
| 分区表工具 | `esp_partition.h`, `parttool` | — | `storage/parttool` |

### 2. NVS 标准写法

```c
#include "nvs_flash.h"
#include "nvs.h"

/* 初始化：必须处理分区被截断 / 版本升级 */
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());          /* 会清掉所有配置，产品需评估 */
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);

/* 打开命名空间 → 读 → 写 → 提交 → 关闭 */
nvs_handle_t handle;
ESP_ERROR_CHECK(nvs_open("storage", NVS_READWRITE, &handle));

int32_t counter = 0;
esp_err_t r = nvs_get_i32(handle, "restart_counter", &counter);
if (r == ESP_ERR_NVS_NOT_FOUND)      ESP_LOGI(TAG, "first boot");
else if (r != ESP_OK)                ESP_LOGE(TAG, "get: %s", esp_err_to_name(r));

counter++;
ESP_ERROR_CHECK(nvs_set_i32(handle, "restart_counter", counter));
ESP_ERROR_CHECK(nvs_commit(handle));             /* 关键：不 commit 不保证落盘 */
nvs_close(handle);
```

规则：

- **每个 `nvs_set_*` 后必须 `nvs_commit()`**（或依赖 `nvs_close()` 的隐式行为，但显式更安全）。
- 支持类型：`i8/u8/i16/u16/i32/u32/i64/u64`、`str`、`blob`；对应 `nvs_get_*` / `nvs_set_*`。读 `str`/`blob` 需两阶段（先传 NULL 拿 length）。
- `nvs_open_from_partition()` 可指定自定义分区；`nvs_get_stats()` / `nvs_get_used_entry_count()` 查容量。
- 单条 value 上限约 4000 B（blob 可更大但受页大小限制）；大数据用文件系统。
- NVS 适合**小、频繁、键值型**数据；写入有磨损均衡但不要 kHz 级高频写。
- Wi-Fi 凭据默认存 NVS（技能 06），所以 `nvs_flash_init()` 必须在所有网络/蓝牙栈之前。
- `nvs_flash_erase()` 会清掉 Wi-Fi 配网与 BLE 绑定，产品代码应只在首次/恢复出厂时擦，或改用 `nvs_flash_erase_partition()` 精确擦自定义分区。

### 3. 分区 API（直接读写 flash）

```c
#include "esp_partition.h"

const esp_partition_t *part = esp_partition_find_first(ESP_PARTITION_TYPE_DATA,
                                                      ESP_PARTITION_SUBTYPE_DATA_SPIFFS,
                                                      "storage");
if (!part) { ESP_LOGE(TAG, "partition not found"); return ESP_ERR_NOT_FOUND; }

uint8_t buf[256] = {0};
/* 写前必须先擦除，且 offset/size 按 4 KB (SPI_FLASH_SEC_SIZE) 对齐 */
ESP_ERROR_CHECK(esp_partition_erase_range(part, 0, 4096));
ESP_ERROR_CHECK(esp_partition_write(part, 0, buf, sizeof(buf)));
ESP_ERROR_CHECK(esp_partition_read(part, 0, buf, sizeof(buf)));
```

- `esp_partition_find_first()` / `esp_partition_find()` + 迭代器 → `esp_partition_iterator_release()`；`esp_partition_get_sha256()`, `esp_partition_verify()`, `esp_partition_mmap()` 用于校验与内存映射。
- **写前擦除、4 KB 对齐**是硬规则；`esp_partition_write_raw()` 用于绕过加密层的特殊场景（谨慎）。
- 分区 CSV 定义见技能 02；`esp_partition_t.address/size` 是绝对地址/字节数。
- `spi_flash_mmap_pages()` 可把 flash 只读数据映射到 CPU 地址空间（如字库、图片资源），配合 `spi_flash_munmap()` 释放；映射是只读 32 MB 数据空间，别当 RAM 写。
- Flash 加密开启后，`esp_partition_write()` 会自动加密、`esp_partition_read()` 自动解密；用 `esp_partition_write_raw()` 会写入密文导致后续读出乱码（技能 10）。

### 4. 文件系统

```c
/* SPIFFS */
#include "esp_spiffs.h"
esp_vfs_spiffs_conf_t conf = {
    .base_path = "/spiffs", .partition_label = "storage",
    .max_files = 5, .format_if_mount_failed = true,
};
ESP_ERROR_CHECK(esp_vfs_spiffs_register(&conf));
size_t total = 0, used = 0;
ESP_ERROR_CHECK(esp_spiffs_info("storage", &total, &used));
/* 之后用标准 fopen/fread/fwrite；esp_vfs_spiffs_unregister() 卸载 */

/* FATFS on flash（带磨损均衡） */
#include "esp_vfs_fat.h"
esp_vfs_fat_mount_config_t mcfg = {
    .format_if_mount_failed = true, .max_files = 4,
    .allocation_unit_size = CONFIG_WL_SECTOR_SIZE,
};
wl_handle_t wl = WL_INVALID_HANDLE;
ESP_ERROR_CHECK(esp_vfs_fat_spiflash_mount_rw_wl("/fat", "storage", &mcfg, &wl));
/* ... fopen("/fat/x.txt", "w") ... */
ESP_ERROR_CHECK(esp_vfs_fat_spiflash_unmount_rw_wl("/fat", wl));
```

- `esp_vfs_fat_sdmmc_mount()` / `esp_vfs_fat_sdspi_mount()` 挂 SD 卡；卸载用 `esp_vfs_fat_sdcard_unmount()`（旧 `esp_vfs_fat_sdmmc_unmount` 已废弃）。
- **flash 上的 FATFS 必须通过 `wear_levelling`**（`*_rw_wl` 系列），否则反复写同一扇区会快速磨损。
- SPIFFS 适合小文件、掉电安全要求不高的场景；FATFS 适合大文件/与 PC 交换；两者不要挂同一分区。
- `format_if_mount_failed = true` 会在首次启动格式化，产品发布前要评估数据丢失风险。
- 参考 `examples/storage/spiffs`, `fatfs/ext_flash`, `sd_card`, `wear_levelling`, `perf_benchmark`。

### 5. OTA（含回滚，生产必做）

```c
#include "esp_ota_ops.h"
#include "esp_app_desc.h"

const esp_partition_t *update_part = esp_ota_get_next_update_partition(NULL);
ESP_ASSERT_ON_FALSE(update_part, ESP_ERR_NOT_FOUND, TAG, "no OTA partition");

esp_ota_handle_t ota;
ESP_ERROR_CHECK(esp_ota_begin(update_part, OTA_SIZE_UNKNOWN, &ota));
/* 循环接收固件分片，每片： */
ESP_ERROR_CHECK(esp_ota_write(ota, chunk, chunk_len));
/* 收完后： */
ESP_ERROR_CHECK(esp_ota_end(ota));
ESP_ERROR_CHECK(esp_ota_set_boot_partition(update_part));
esp_restart();   /* 重启进新固件 */

/* 新固件自检通过后必须标记有效，否则下次启动回滚 */
void app_main(void) {
    /* ... 业务自检 ... */
    const esp_partition_t *running = esp_ota_get_running_partition();
    esp_ota_img_states_t state;
    if (esp_ota_get_state_partition(running, &state) == ESP_OK &&
        state == ESP_OTA_IMG_PENDING_VERIFY) {
        if (self_test_ok()) ESP_ERROR_CHECK(esp_ota_mark_app_valid_cancel_rollback());
        else { ESP_LOGE(TAG, "self-test failed"); esp_ota_mark_app_invalid_rollback_and_reboot(); }
    }
}
```

HTTPS OTA 更常用：

```c
#include "esp_https_ota.h"
esp_https_ota_config_t cfg = {
    .http_config = &(esp_http_client_config_t){
        .url = CONFIG_FIRMWARE_URL,
        .crt_bundle_attach = esp_crt_bundle_attach,
        .timeout_ms = 10000,
    },
    .http_client_init_cb = NULL,
    .bulk_flash_erase = false,
    .partial_http_download = false,
};
ESP_ERROR_CHECK(esp_https_ota(&cfg));   /* 或 begin/perform/is_complete/finish 分步控制 */
```

规则：

- 分区表必须有 `ota_0` / `ota_1` + `otadata`（技能 02 模板已含），且 `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y` 才能回滚。
- 必须实现「启动自检 → `esp_ota_mark_app_valid_cancel_rollback()`」，否则 `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` 会在下次重启回滚到旧固件。
- 用 `esp_app_desc_t` / `esp_ota_get_partition_description()` 校验版本号、编译时间、`project_name`，**拒绝**降级或错误固件的写入（配合 `CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK` + eFuse secure_version）。
- 下载过程要处理断网/超时/校验和；`esp_https_ota` 内部有超时但外层仍需看门狗订阅与重试策略。
- Flash 加密 / 安全启动启用后 OTA 镜像必须是已签名/加密格式，不能直接用明文 bin（技能 10）。
- 参考 `examples/system/ota/simple_ota_example`、`native_ota_example`、`ota/advanced_https_ota`、`storage/nvs_bootloader`。

### 6. 常见故障

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `ESP_ERR_NVS_NO_FREE_PAGES` | 分区被截断或多版本共存 | `nvs_flash_erase()` + 重 init（注意会清配置） |
| 写 flash 后读出全 0xFF/错乱 | 未先 erase / 未 4 KB 对齐 | `esp_partition_erase_range()` 后再写 |
| `esp_ota_begin` 失败 | 无 OTA 分区 / 镜像太大 | 检查 `partitions.csv` 与 `idf.py size` |
| 升级后自动回滚 | 新固件未标记 valid / 自检失败 | 调 `esp_ota_mark_app_valid_cancel_rollback()` |
| FATFS 挂载失败 | 分区未格式化或越界 | `format_if_mount_failed`，核对分区 offset/size |
| 开 Flash 加密后读出乱码 | 用了 `esp_partition_write_raw()` | 改用 `esp_partition_write()`（自动加密） |

## 示例

```c
/* 保存 Wi-Fi 配置到 NVS（凭据不硬编码） */
static esp_err_t save_wifi_cred(const char *ssid, const char *pass)
{
    nvs_handle_t h;
    esp_err_t ret = nvs_open("wifi_cfg", NVS_READWRITE, &h);
    if (ret != ESP_OK) return ret;
    ret = nvs_set_str(h, "ssid", ssid);
    if (ret == ESP_OK) ret = nvs_set_str(h, "pass", pass);
    if (ret == ESP_OK) ret = nvs_commit(h);
    nvs_close(h);
    if (ret != ESP_OK) ESP_LOGE(TAG, "save cred failed: %s", esp_err_to_name(ret));
    return ret;
}
```

## 检查清单

- [ ] `nvs_flash_init()` 在最前面，处理了 NO_FREE_PAGES / NEW_VERSION_FOUND
- [ ] 每个 `nvs_set_*` 后有 `nvs_commit()`，handle 已 close
- [ ] 分区读写遵循 4 KB 对齐 + 写前擦除；分区末端 < 16 MB
- [ ] flash 上的 FATFS 使用 wear_levelling；SPIFFS 与 FATFS 未共用分区
- [ ] OTA 分区 + otadata 齐全，启用了回滚保护并实现启动自检标记
- [ ] HTTPS OTA 配了 CA 证书、超时与版本/防降级校验
- [ ] 16 MB flash 分区表与 `CONFIG_ESPTOOLPY_FLASHSIZE_16MB` 一致
