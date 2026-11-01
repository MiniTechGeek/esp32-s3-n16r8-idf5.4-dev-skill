---
name: esp32s3-ble
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的 Bluetooth LE 开发：NimBLE 与 Bluedroid 主机栈选型、GATT 服务/特征、广播与扫描、配对绑定、BLE Mesh/HID 及 Wi-Fi 与 BLE 共存配置。当任务涉及 BLE 广播、手机连接、GATT 数据通道、iBeacon、BLE 配网或蓝牙/Wi-Fi 同时工作时加载本技能。
---

# ESP32-S3N16R8 · Bluetooth LE

## 描述

ESP32-S3 支持 **Bluetooth 5 (LE)** + Bluetooth Mesh，与 Wi-Fi 共用一根天线和 2.4 GHz 射频；不支持经典蓝牙（BR/EDR）音频。本技能给出 v5.4 的主机栈选型、初始化流程与共存约束。

## 使用场景

- NimBLE / Bluedroid 主机栈选型与 menuconfig 配置
- GATT Server（服务/特征/描述符）、GATT Client、通知/指示
- 广播（含扩展广播、多广播集）、扫描、连接参数、MTU
- 配对绑定、LE Secure Connections、隐私
- BLE 配网（与 Wi-Fi 联动）、BLE HID、BLE Mesh
- Wi-Fi + BLE 共存、天线与射频性能问题

## 指令

### 1. 硬件能力与栈选型

| 项目 | 事实（A 数据手册 / C 模组手册） |
| --- | --- |
| 协议 | Bluetooth 5 (LE)，Bluetooth Mesh；**无经典蓝牙 BR/EDR** |
| PHY | 125 kbps / 500 kbps / 1 Mbps / 2 Mbps（LE Coded / 2M） |
| 特性 | 广播扩展 (LE Advertising Extensions)、多广播集、信道选择算法 #2 |
| 发射功率 | 最高 20 dBm；接收灵敏度 125 kbps 时达 −104.5 dBm |
| 射频 | 与 Wi-Fi 共用天线，需软件共存（coex） |
| 控制器 | `components/bt/controller/esp32s3`（芯片专用） |

栈选型（v5.4，二者**只能选一个**）：

| | NimBLE | Bluedroid |
| --- | --- | --- |
| 配置 | `CONFIG_BT_ENABLED=y`, `CONFIG_BT_NIMBLE_ENABLED=y`, `CONFIG_BT_BLUEDROID_ENABLED=n` | `CONFIG_BT_ENABLED=y`, `CONFIG_BT_BLUEDROID_ENABLED=y`, `CONFIG_BT_NIMBLE_ENABLED=n` |
| 资源 | RAM/Flash 占用小，适合仅 BLE、资源敏感 | 功能全（经典 BT + BLE + Mesh/HID/SPP），体积大 |
| API | `host/ble_hs.h`, `host/ble_gatt.h`, `nimble/nimble_port.h`, `services/gap/ble_svc_gap.h`, `services/gatt/ble_svc_gatt.h` | `esp_gap_ble_api.h`, `esp_gatts_api.h`, `esp_gattc_api.h`, `esp_bt_main.h`, `esp_bt_device.h` |
| 示例 | `examples/bluetooth/nimble/bleprph`, `blecent`, `NimBLE_GATT_Server`, `NimBLE_Security` | `examples/bluetooth/bluedroid/ble/gatt_server`, `gatt_client`, `ble_ibeacon`, `ble_throughput`, `ble50_security_*` |
| 推荐场景 | 新项目、纯 BLE、Wi-Fi + BLE 共存、配网 | 需要 BLE Mesh / HID / SPP / 经典蓝牙兼容 |

> **WROOM-1-N16R8 有 16 MB flash + 8 MB PSRAM，两者都放得下**；但配网等场景 Bluedroid 二进制明显更大，`examples/provisioning/wifi_prov_mgr` 明确提示「Bluedroid as binary is larger than default size」→ 需换更大 app 分区（技能 02 的 4 MB OTA 分区已足够）。

### 2. 通用初始化顺序（Bluedroid）

```c
#include "esp_bt.h"
#include "esp_bt_main.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "nvs_flash.h"

/* 1) NVS（绑定密钥存这里） */
ESP_ERROR_CHECK(nvs_flash_init());   /* 含 NO_FREE_PAGES / NEW_VERSION_FOUND 处理 */

/* 2) 释放经典蓝牙内存（纯 BLE 项目省 RAM） */
ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

/* 3) 控制器 */
esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

/* 4) 主机栈 */
ESP_ERROR_CHECK(esp_bluedroid_init());
ESP_ERROR_CHECK(esp_bluedroid_enable());

/* 5) 注册回调（GAP / GATTS），之后才配置广播与 profile */
ESP_ERROR_CHECK(esp_ble_gap_register_callback(gap_event_handler));
ESP_ERROR_CHECK(esp_gatts_register_callback(gatts_event_handler));
ESP_ERROR_CHECK(esp_gatts_app_register(PROFILE_APP_ID));
```

NimBLE 初始化顺序：`nvs_flash_init()` → `nimble_port_init()` → `ble_svc_gap_init()` / `ble_svc_gatt_init()` → `ble_gatts_count_cfg()` / `ble_gatts_add_svcs()` → `ble_hs_cfg` 回调赋值 → `nimble_port_freertos_init(host_task)`（内部 `ble_hs` 事件循环）。见 `examples/bluetooth/nimble/bleprph`。

### 3. GATT Server（Bluedroid）事件链

```
ESP_GATTS_REG_EVT        → esp_ble_gap_set_device_name() + esp_ble_gap_config_adv_data()
                            + esp_gatts_create_service()
ESP_GATTS_CREATE_EVT      → esp_gatts_start_service() + esp_gatts_add_char()
ESP_GATTS_ADD_CHAR_EVT    → esp_gatts_add_char_descr()（可选 CCCD）
ESP_GATTS_CONNECT_EVT     → 更新连接参数 esp_ble_gap_update_conn_params()
ESP_GATTS_WRITE_EVT       → 处理写入；需回复 esp_gatts_send_response()
ESP_GATTS_READ_EVT        → 构造 esp_gatt_rsp_t 并 esp_gatts_send_response()
ESP_GATTS_MTU_EVT         → 记录协商后的 MTU（默认 23，可到 517）
ESP_GAP_BLE_ADV_START_COMPLETE_EVT / ESP_GAP_BLE_SEC_REQ_EVT / ... → 对应处理
```

```c
/* 广播参数（可连接 + 可扫描） */
static esp_ble_adv_data_t adv_data = {
    .set_scan_rsp = false,
    .include_name = true,
    .include_txpower = true,
    .min_interval = 0x0020,   /* 20 ms ~ 单位 0.625 ms */
    .max_interval = 0x0040,
    .appearance = 0x00,
    .manufacturer_len = 0,
    .p_manufacturer_data = NULL,
    .service_data_len = 0,
    .p_service_data = NULL,
    .service_uuid_len = sizeof(adv_service_uuid128),
    .p_service_uuid = adv_service_uuid128,
    .flag = (ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT),
};
esp_ble_adv_params_t adv_params = {
    .adv_int_min = 0x20, .adv_int_max = 0x40,
    .adv_type = ADV_TYPE_IND, .own_addr_type = BLE_ADDR_TYPE_PUBLIC,
    .channel_map = ADV_CHNL_ALL, .adv_filter_policy = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};
ESP_ERROR_CHECK(esp_ble_gap_config_adv_data(&adv_data));       /* 完成后在回调里 */
ESP_ERROR_CHECK(esp_ble_gap_start_advertising(&adv_params));
```

规则：

- **所有 GAP/GATTS 操作都是异步事件驱动**：配置广播、创建服务、添加特征都要在对应 `*_EVT` 完成后再做下一步，禁止连续同步调用后假设立即生效。
- `ESP_GATTS_WRITE_EVT` / `READ_EVT` 必须用 `esp_gatts_send_response()` 回复，否则对端会超时断连；`need_rsp=true` 时才回。
- 通知/指示：`esp_ble_gatts_send_indicate()`；指示需对端确认，通知不需要。
- MTU：`esp_ble_gatt_set_local_mtu(517)` 在 `esp_bluedroid_enable()` 之后、注册之前设置；实际以 `ESP_GATTS_MTU_EVT` 为准，分包发送。
- 特征值长度 ≤ MTU−3；大数据要分片或用 PSRAM 缓冲（技能 09）。
- 广播扩展/多广播集：`esp_ble_gap_ext_adv_set_params()`, `esp_ble_gap_ext_adv_set_data()`, `esp_ble_gap_ext_adv_start()`（Bluedroid `ble_50` 示例）；NimBLE 用 `ble_gap_ext_adv_*`。
- 参考示例：`bluedroid/ble/gatt_server`, `gatt_server_service_table`, `ble_ibeacon`, `gatt_security_server`, `ble50_security_server`, `esp_hid_device`。

### 4. 安全（配对/绑定）

```c
esp_ble_auth_req_t auth_req = ESP_LE_AUTH_REQ_SC_MITM_BOND;   /* LE Secure Connections + MITM + 绑定 */
esp_ble_io_cap_t iocap = ESP_IO_CAP_IO;                       /* 有显示+输入 */
uint8_t key_size = 16, init_key = ESP_BLE_ENC_KEY_MASK | ESP_BLE_ID_KEY_MASK,
        rsp_key  = ESP_BLE_ENC_KEY_MASK | ESP_BLE_ID_KEY_MASK;
esp_ble_gap_set_security_param(ESP_BLE_SM_AUTHEN_REQ_MODE, &auth_req, sizeof(uint8_t));
esp_ble_gap_set_security_param(ESP_BLE_SM_IOCAP_MODE, &iocap, sizeof(uint8_t));
esp_ble_gap_set_security_param(ESP_BLE_SM_MAX_KEY_SIZE, &key_size, sizeof(uint8_t));
esp_ble_gap_set_security_param(ESP_BLE_SM_SET_INIT_KEY, &init_key, sizeof(uint8_t));
esp_ble_gap_set_security_param(ESP_BLE_SM_SET_RSP_KEY,  &rsp_key,  sizeof(uint8_t));
/* ESP_GAP_BLE_SEC_REQ_EVT → esp_ble_gap_security_rsp()；
 * ESP_GAP_BLE_NC_REQ_EVT → esp_ble_confirm_reply()（Numeric Comparison）；
 * ESP_GAP_BLE_PASSKEY_REQ_EVT → esp_ble_passkey_reply() */
```

- 绑定信息存 **NVS**，故 `nvs_flash_init()` 必须先于蓝牙栈；擦除 NVS 会清掉所有绑定。
- `esp_ble_get_bond_device_num()` / `esp_ble_get_bond_device_list()` 管理已绑定设备；`esp_ble_remove_bond_device()` 解绑。
- 隐私/静态地址：`esp_ble_gap_config_local_privacy(true)`、`BLE_ADDR_TYPE_RANDOM`。
- 参考 `examples/bluetooth/bluedroid/ble/gatt_security_server` 与 `nimble/NimBLE_Security`。

### 5. Wi-Fi + BLE 共存

- `CONFIG_BT_ENABLED=y` + `CONFIG_ESP_COEX_ENABLED=y`（组件 `esp_coex`），`CONFIG_ESP_COEX_SW_COEXIST_ENABLE=y`；Wi-Fi 与 BLE 共天线时由软件共存仲裁。
- 二者同时工作会争抢 2.4 GHz 时隙，吞吐与延迟下降；高 Wi-Fi 吞吐场景限制 BLE 连接事件长度/广播间隔。
- `CONFIG_BT_CTRL_MODE_EFF` / `CONFIG_BT_BLE_50_FEATURES_SUPPORTED` 等按需求开；参考 `examples/bluetooth/nimble/bleprph_wifi_coex`、`esp_ble_mesh/wifi_coexist`（其 `sdkconfig.defaults.esp32s3` 明确 `CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y`）。
- 射频硬件：50 Ω 阻抗、CLC 匹配、晶振 ±10 ppm（技能 01）；Wi-Fi 连接失败在排除软件后需校准晶振负载电容。
- BLE 配网：`wifi_provisioning` 的 `wifi_prov_mgr` 用 BLE 传输配网凭据，需同时加载技能 06；注意 Bluedroid 体积、app 分区大小、以及 `CONFIG_BT_BLE_ENABLED`/`CONFIG_BT_NIMBLE_ENABLED` 与配网传输方式匹配。

### 6. 常见故障

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `esp_bt_controller_enable` 失败 | 未释放经典 BT 内存 / 与 NimBLE 同时启用 | 检查 `CONFIG_BT_BLUEDROID_ENABLED` 与 `NIMBLE` 互斥 |
| GATT 服务创建后手机看不到 | 在 `REG_EVT` 前就 `create_service` | 严格按事件链顺序 |
| 手机写入后断开 | `ESP_GATTS_WRITE_EVT` 未 `esp_gatts_send_response()` | 每个 need_rsp 请求都回复 |
| 广播不出现 | `esp_ble_gap_start_advertising` 在 `ADV_DATA_SET_COMPLETE_EVT` 之前 | 等回调；检查广播数据与 `flag` |
| 绑定后重连失败 | NVS 被擦除或密钥配置不一致 | 检查 `nvs_flash_init()` 与 `ESP_BLE_SM_*` 参数 |
| 共存时 Wi-Fi 掉速 | 未开 coex / BLE 广播太密 | `CONFIG_ESP_COEX_SW_COEXIST_ENABLE=y`，加大广播间隔 |

## 示例

```c
/* 事件链骨架（Bluedroid GATTS）——异步、每步等事件 */
static void gatts_cb(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if,
                     esp_ble_gatts_cb_param_t *param)
{
    switch (event) {
    case ESP_GATTS_REG_EVT:
        ESP_ERROR_CHECK(esp_ble_gap_set_device_name("ESP32S3-N16R8"));
        ESP_ERROR_CHECK(esp_ble_gap_config_adv_data(&adv_data));
        ESP_ERROR_CHECK(esp_gatts_create_service(gatts_if, &svc_id, 4));
        break;
    case ESP_GATTS_CREATE_EVT:
        ESP_ERROR_CHECK(esp_gatts_start_service(param->create.service_handle));
        ESP_ERROR_CHECK(esp_gatts_add_char(param->create.service_handle, &char_uuid,
                                           ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                                           ESP_GATT_CHAR_PROP_BIT_READ |
                                           ESP_GATT_CHAR_PROP_BIT_WRITE |
                                           ESP_GATT_CHAR_PROP_BIT_NOTIFY,
                                           NULL, NULL));
        break;
    case ESP_GATTS_WRITE_EVT:
        if (param->write.need_rsp) {
            esp_gatt_rsp_t rsp = {0};
            ESP_ERROR_CHECK(esp_gatts_send_response(gatts_if, param->write.conn_id,
                                                    param->write.trans_id, ESP_GATT_OK, &rsp));
        }
        break;
    default: break;
    }
}
```

## 检查清单

- [ ] 明确选型 NimBLE 或 Bluedroid，且 Kconfig 中二者互斥
- [ ] `nvs_flash_init()` 在蓝牙栈之前；纯 BLE 项目释放了经典 BT 内存
- [ ] GATT 操作严格按事件链异步推进，所有 READ/WRITE 有 response
- [ ] MTU 协商后按 `MTU−3` 分片；广播在 `ADV_DATA_SET_COMPLETE_EVT` 后启动
- [ ] 绑定参数与 NVS 持久化一致，密钥/IO 能力符合产品安全需求
- [ ] Wi-Fi + BLE 场景开启 `esp_coex`，并评估吞吐/延迟影响
- [ ] app 分区足够容纳所选主机栈（Bluedroid 偏大）
