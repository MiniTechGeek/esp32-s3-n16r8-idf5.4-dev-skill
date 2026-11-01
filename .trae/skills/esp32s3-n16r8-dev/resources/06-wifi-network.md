---
name: esp32s3-wifi-network
description: ESP32-S3N16R8 在 ESP-IDF v5.4 下的 Wi-Fi STA/AP/扫描/省电/ESP-NOW、esp_netif 与 esp_event 初始化顺序，以及 lwIP socket、SNTP、HTTP 客户端/服务端、HTTPS、MQTT、mDNS 与配网。当任务涉及联网、连路由器、开热点、HTTP/MQTT 上报、时间同步或 Wi-Fi 断线重连时加载本技能。
---

# ESP32-S3N16R8 · Wi-Fi 与网络协议

## 描述

规定 v5.4 网络栈的正确初始化顺序（**nvs → netif → event loop → wifi**）与事件驱动模型，避免阻塞式轮询与「没网就死等」。

## 使用场景

- Wi-Fi STA 连接、断线重连、SoftAP、扫描、信道/国家码、省电
- ESP-NOW、SmartConfig、Wi-Fi Provisioning
- socket / SNTP / HTTP client / HTTP server / HTTPS / MQTT / mDNS
- 网络初始化顺序、事件回调、IP 获取、超时与错误恢复

## 指令

### 1. 组件与初始化顺序（不可颠倒）

```c
/* 1) NVS（Wi-Fi 驱动用它存校准/配置） */
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);

/* 2) netif + 默认事件循环 */
ESP_ERROR_CHECK(esp_netif_init());
ESP_ERROR_CHECK(esp_event_loop_create_default());
esp_netif_t *sta = esp_netif_create_default_wifi_sta();   /* 或 esp_netif_create_default_wifi_ap() */
assert(sta);

/* 3) Wi-Fi 驱动 */
wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
ESP_ERROR_CHECK(esp_wifi_init(&cfg));

/* 4) 注册事件处理（实例注册，便于后续反注册） */
esp_event_handler_instance_t any_id, got_ip;
ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT, ESP_EVENT_ANY_ID,
                                                     &wifi_event_handler, NULL, &any_id));
ESP_ERROR_CHECK(esp_event_handler_instance_register(IP_EVENT, IP_EVENT_STA_GOT_IP,
                                                     &wifi_event_handler, NULL, &got_ip));

/* 5) 配置并启动 */
wifi_config_t wifi_cfg = {
    .sta = { .ssid = CONFIG_ESP_WIFI_SSID, .password = CONFIG_ESP_WIFI_PASSWORD,
             .threshold.authmode = WIFI_AUTH_WPA2_PSK,
             .sae_pwe_h2e = WPA3_SAE_PWE_BOTH },
};
ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_cfg));
ESP_ERROR_CHECK(esp_wifi_start());
/* 最后在 WIFI_EVENT_STA_START 回调里调 esp_wifi_connect() */
```

```c
static void wifi_event_handler(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (base == WIFI_EVENT && id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (base == WIFI_EVENT && id == WIFI_EVENT_STA_DISCONNECTED) {
        /* 有限次重连，避免无限打日志刷屏 */
        if (s_retry < MAX_RETRY) { esp_wifi_connect(); s_retry++; }
        else { xEventGroupSetBits(s_evt, WIFI_FAIL_BIT); }
    } else if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *e = (ip_event_got_ip_t *)data;
        ESP_LOGI(TAG, "got ip: " IPSTR, IP2STR(&e->ip_info.ip));
        s_retry = 0;
        xEventGroupSetBits(s_evt, WIFI_CONNECTED_BIT);
    }
}
```

规则：

- 顺序固定：`nvs_flash_init` → `esp_netif_init` → `esp_event_loop_create_default` → `esp_netif_create_default_wifi_sta` → `esp_wifi_init` → 注册事件 → `esp_wifi_set_mode` → `esp_wifi_set_config` → `esp_wifi_start`。
- `esp_wifi_connect()` 在 `WIFI_EVENT_STA_START` 之后调用；断线重连要有次数上限与退避（如 `vTaskDelay(pdMS_TO_TICKS(1000 << retry))`）。
- 用 `EventGroupHandle_t` 或任务通知等待「拿到 IP」或「连接失败」，**不要**用 `while(!connected) vTaskDelay()` 无限轮询；等待要设超时。
- 反注册用 `esp_event_handler_instance_unregister()`；`esp_wifi_stop()` → `esp_wifi_deinit()` → `esp_netif_destroy_default_wifi()` → `esp_event_loop_delete_default()` 顺序释放。
- 头文件：`esp_wifi.h`, `esp_netif.h`, `esp_event.h`, `nvs_flash.h`, `lwip/err.h`, `lwip/sys.h`；`REQUIRES esp_wifi esp_netif esp_event nvs_flash lwip`。
- 参考示例：`examples/wifi/getting_started/station`、`softAP`、`scan`、`espnow`、`power_save`、`smart_config`。
- **芯片特性注意**：STA 扫描时 SoftAP 信道会随之改变；4 个虚拟接口；支持 802.11mc FTM；不支持 5 GHz。

### 2. Wi-Fi 配置要点

| 需求 | API / Kconfig |
| --- | --- |
| 省电 | `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)` / `WIFI_PS_MAX_MODEM`；配合 `CONFIG_PM_ENABLE=y`（技能 10） |
| 扫描 | `esp_wifi_scan_start(&wifi_scan_config_t, true)` + `esp_wifi_scan_get_ap_num/records`；`WIFI_EVENT_SCAN_DONE` |
| 固定信道/带宽 | `esp_wifi_set_channel`, `esp_wifi_set_bandwidth(WIFI_IF_STA, WIFI_BW_HT20/HT40)`；40 MHz 仅 2.4G HT40 |
| 国家码 | `esp_wifi_set_country_code("CN", true)` / `esp_wifi_set_country()` |
| 发射功率 | `esp_wifi_set_max_tx_power(int8_t)`（单位 0.25 dBm） |
| 静态 IP | 用 `esp_netif_dhcpc_stop()` + `esp_netif_set_ip_info()`；或 `CONFIG_LWIP_DHCP_*` |
| 存配置位置 | `esp_wifi_set_storage(WIFI_STORAGE_FLASH)` 默认存 NVS；RAM 用 `WIFI_STORAGE_RAM` |
| SoftAP | `esp_netif_create_default_wifi_ap()` + `WIFI_MODE_AP` + `.ap` 配置（`channel`, `max_connection`（默认 4，最多支持 10）, `authmode`） |
| 混杂/CSI | `esp_wifi_set_promiscuous`, `esp_wifi_set_csi_config`, `esp_wifi_set_csi_rx_cb`（需要 PSRAM 缓冲） |

> `esp_wifi_set_ant_gpio` / `esp_wifi_set_ant` 已废弃，改用 `esp_phy_set_ant_gpio()` / `esp_phy_set_ant()`（见 `esp_phy.h`）。

### 3. lwIP socket 与 SNTP

```c
/* TCP client：所有阻塞调用都设超时 */
int sock = socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
if (sock < 0) { ESP_LOGE(TAG, "socket: errno %d", errno); return ESP_FAIL; }
struct timeval to = { .tv_sec = 5, .tv_usec = 0 };
setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &to, sizeof(to));
struct sockaddr_in dst = { .sin_family = AF_INET, .sin_port = htons(80),
                           .sin_addr.s_addr = inet_addr("192.168.1.10") };
if (connect(sock, (struct sockaddr *)&dst, sizeof(dst)) != 0) { /* 处理 errno */ }
int n = send(sock, req, strlen(req), 0);
n = recv(sock, buf, sizeof(buf) - 1, 0);
close(sock);
```

- lwIP socket 是标准 BSD 风格；`<lwip/sockets.h>`、`lwip/netdb.h`、`lwip/err.h`。
- **所有 `connect/recv/send` 必须设超时**（`SO_RCVTIMEO`/`SO_SNDTIMEO` 或非阻塞 + `select()`），否则网络异常时任务永久阻塞、触发 TWDT。
- SNTP：`esp_sntp.h`（`esp_sntp_setoperatingmode(ESP_SNTP_OPMODE_POLL)`, `esp_sntp_setservername(0, "pool.ntp.org")`, `esp_sntp_init()`），或 LwIP SNTP；用 `time()`/`gettimeofday()` 前等 `SNTP_SYNC_STATUS_COMPLETED`。参考 `examples/protocols/sntp`。

### 4. HTTP 客户端 / 服务端 / MQTT

```c
/* HTTP client（esp_http_client.h） */
esp_http_client_config_t cfg = {
    .url = "https://example.com/api",
    .method = HTTP_METHOD_GET,
    .timeout_ms = 5000,
    .crt_bundle_attach = esp_crt_bundle_attach,   /* HTTPS：需 CONFIG_MBEDTLS_CERTIFICATE_BUNDLE */
    .event_handler = http_event_handler,
    .buffer_size = 4096,
};
esp_http_client_handle_t client = esp_http_client_init(&cfg);
esp_err_t err = esp_http_client_perform(client);   /* 阻塞但内部有 timeout_ms 上限 */
if (err == ESP_OK) {
    int status = esp_http_client_get_status_code(client);
    int64_t len = esp_http_client_get_content_length(client);
}
esp_http_client_cleanup(client);
```

```c
/* HTTP server（esp_http_server.h） */
httpd_handle_t server = NULL;
httpd_config_t config = HTTPD_DEFAULT_CONFIG();
config.stack_size = 8192;                /* URI handler 里做 JSON/文件时加大 */
config.max_uri_handlers = 8;
ESP_ERROR_CHECK(httpd_start(&server, &config));
httpd_uri_t uri = { .uri = "/api", .method = HTTP_GET,
                    .handler = api_get_handler, .user_ctx = NULL };
ESP_ERROR_CHECK(httpd_register_uri_handler(server, &uri));
/* httpd_stop(server) */
```

```c
/* MQTT（mqtt_client.h，esp-mqtt 组件） */
esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = "mqtt://broker.example.com",
    .credentials.username = "user",
    .credentials.authentication.password = "pass",
    .session.keepalive = 60,
    .network.disable_auto_reconnect = false,
};
esp_mqtt_client_handle_t mqtt = esp_mqtt_client_init(&mqtt_cfg);
ESP_ERROR_CHECK(esp_mqtt_client_register_event(mqtt, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL));
ESP_ERROR_CHECK(esp_mqtt_client_start(mqtt));
/* 事件里处理 MQTT_EVENT_CONNECTED / DISCONNECTED / DATA，发布用 esp_mqtt_client_publish() */
```

规则：

- 网络凭据**不要硬编码**在源码：用 `CONFIG_*`（Kconfig.projbuild）或 NVS 加密分区读取。
- HTTPS 必须配证书：`esp_crt_bundle_attach`（组件 `esp-tls` + `CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y`），或 `.cert_pem`；不要设 `.skip_cert_common_name_check = true` 上生产。
- MQTT 用事件驱动，重连由库自动处理；业务侧在 `MQTT_EVENT_CONNECTED` 时订阅，不要一启动就 publish。
- HTTP server 的 `httpd_config_t.stack_size` 默认偏小，处理 JSON/TLS 建议 8192+ 并实测高水位（技能 09）。
- 参考：`examples/protocols/esp_http_client`、`http_server/simple`、`https_request`、`https_mbedtls`、`mqtt/tcp`、`mqtt5`、`sockets`。

### 5. ESP-NOW / 配网

- ESP-NOW：`esp_now.h`，`esp_now_init()` → `esp_now_add_peer()` → `esp_now_register_send_cb()` / `esp_now_register_recv_cb()` → `esp_now_send()`；要求双方同信道，STA/SoftAP 都要先启动；`CONFIG_ESP_WIFI_ESPNOW_MAX_ENCRYPT_NUM` 控制加密节点数。参考 `examples/wifi/espnow`。
- 配网：`wifi_provisioning`（`wifi_prov_mgr.h`），支持 BLE / SoftAP 传输 + Security 1/2；或 SmartConfig（`esp_smartconfig.h`）、WPS。**BLE 配网需与技能 07 的 BLE 组件共存，注意二进制体积与分区大小**（`examples/provisioning/wifi_prov_mgr` 的 sdkconfig 备注：Bluedroid 较大需换大 app 分区）。
- mDNS：`mdns.h`（`mdns_init()`, `mdns_hostname_set()`, `mdns_service_add()`），组件 `mdns` 由组件管理器引入。

### 6. 常见故障

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `WIFI_EVENT_STA_DISCONNECTED` 反复 | 密码/加密方式不符、信号弱 | 用 `wifi_config_t.sta.threshold.authmode` 匹配路由器；`esp_wifi_set_max_tx_power`；扫描确认 RSSI |
| 已连上但拿不到 IP | DHCP 未启动/路由器池满 | `esp_netif_dhcpc_start()`；检查 `IP_EVENT_STA_GOT_IP`；必要时静态 IP |
| 网络异常后任务卡死 | socket 无超时 | 全部设 `SO_RCVTIMEO`/`SO_SNDTIMEO` + TWDT 订阅 |
| HTTPS 握手失败 | 缺 CA 证书 / 时间未同步 | `esp_crt_bundle_attach`；先 SNTP 同步时间再 TLS |
| 开启 Wi-Fi 后 ADC2 读数乱 | ADC2 与 Wi-Fi 共用 | 改用 ADC1（技能 05） |
| `esp_netif_create_default_wifi_sta` 返回 NULL | 顺序错/重复创建 | 检查 netif + event loop 已初始化且只创建一次 |

## 示例

```c
/* 等待联网（有界超时，避免无限阻塞） */
EventBits_t bits = xEventGroupWaitBits(s_evt, WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
                                        pdFALSE, pdFALSE, pdMS_TO_TICKS(15000));
if (bits & WIFI_CONNECTED_BIT)      ESP_LOGI(TAG, "connected");
else if (bits & WIFI_FAIL_BIT)     ESP_LOGE(TAG, "connect failed");
else                               ESP_LOGW(TAG, "timeout, will retry later");
```

## 检查清单

- [ ] 初始化顺序为 nvs → netif → event loop → netif_create → wifi_init → 事件 → 配置 → start
- [ ] 重连有上限与退避；等待 IP 用事件组且有超时
- [ ] 所有 socket/HTTP/MQTT 操作设超时，凭据来自配置或 NVS
- [ ] HTTPS 配了 CA 证书，未关闭证书校验
- [ ] 事件处理实例有反注册路径，释放顺序正确
- [ ] 大缓冲（HTTP body / MQTT payload / CSI）用 PSRAM（技能 09）
- [ ] 未对 ADC2 在 Wi-Fi 工作期间做模拟采样（技能 05）
