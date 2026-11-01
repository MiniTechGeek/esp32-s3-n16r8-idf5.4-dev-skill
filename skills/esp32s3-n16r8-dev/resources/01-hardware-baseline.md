---
name: esp32s3-n16r8-hardware
description: ESP32-S3-WROOM-1-N16R8 芯片与模组的硬件规格、引脚可用性、电源与上电复位时序、启动配置、存储器映射及全部芯片勘误。当任务涉及芯片选型/型号解读、GPIO 分配与复用冲突、原理图检查、电源设计、下载模式、Flash/PSRAM 接线、开发板引脚或询问「这个脚能不能用」「为什么 ADC 读不准」时加载本技能。
---

# ESP32-S3N16R8 硬件基线与引脚/勘误约束

## 描述

提供 ESP32-S3-WROOM-1-N16R8 的**实证硬件事实**，用于在写代码前判定引脚合法性、电源/时序可行性与勘误规避。所有数据来自资料包 A/B/C 三类 PDF。生成任何涉及 GPIO、外设映射、启动、供电的代码或原理图说明前，必须先加载本技能。

## 使用场景

- 分配 GPIO、排查「引脚没反应 / 上电异常 / 复用冲突」
- 编写或审查原理图、电源、复位、晶振、RF、USB 部分
- 判断下载模式、烧录失败、JTAG 连接问题
- 询问芯片/模组参数、型号含义、工作温度、存储容量
- 需要确认某条勘误是否影响当前设计

## 指令

### 1. 型号解码与关键规格

`ESP32-S3-WROOM-1-N16R8` = 板载 PCB 天线模组，内置 **ESP32-S3R8** 芯片：

| 字段 | 含义 |
| --- | --- |
| `N16` | 16 MB **Quad SPI** flash（模组内，CS0） |
| `R8` | 8 MB **Octal SPI** PSRAM（芯片封装内，CS1） |
| 无 `U` 后缀 | WROOM-1 板载 PCB 天线；WROOM-1U 为外部天线连接器 |
| 无 `V`/`VA` 后缀 | **VDD_SPI = 3.3 V**；仅 `N16R16VA` 为 1.8 V |

硬事实（勿改写）：

- 芯片 QFN56 7×7 mm；模组 18.0 × 25.5 × 3.1 mm，41 个管脚（含 GND/3V3/EN/EPAD）。
- 双核 Xtensa LX7 @ 最高 240 MHz，单精度 FPU，5 级流水，128 位数据总线，SIMD/PIE 向量扩展（AI 加速指令）。
- 384 KB ROM / 512 KB SRAM / 16 KB RTC SRAM（8 KB FAST + 8 KB SLOW，Deep-sleep 保持）/ 4096 bit eFuse（1792 bit 用户可用）。
- ULP 协处理器：ULP-RISC-V 或 ULP-FSM 二选一。
- GDMA：**5 个 RX + 5 个 TX 通道**。
- 外设：3×UART、2×I2C、2×I2S、2×通用 SPI（SPI2/SPI3）、SPI0/1 专供 flash/PSRAM、LCD 接口、DVP 8~16 位摄像头、全速 USB OTG、USB Serial/JTAG、SD/MMC 主机（**仅 1 个，不能作 slave**）、TWAI（CAN 2.0）、LEDC 8 通道、2×MCPWM、RMT、PCNT、2×12 位 SAR ADC 共 20 通道、14 触摸通道、温度传感器、4×54 位通用定时器、52 位系统定时器、3×看门狗。
- Wi-Fi：802.11b/g/n，20/40 MHz，1T1R ≤150 Mbps，4 个虚拟接口，STA/SoftAP/混杂、802.11mc FTM；**STA 扫描时 SoftAP 信道会随之改变**。
- BLE：Bluetooth 5 (LE)、Mesh，125/500 kbps/1/2 Mbps，广播扩展，最高发射 20 dBm。
- 供电 3.0~3.6 V，推荐 3.3 V / **≥500 mA**；R8 系列环境温度 **−40~65 °C**（开启 PSRAM ECC 可到 85 °C，但可用容量减少 1/16）。
- 模组默认 SPI flash 时钟 **≤80 MHz 且不支持自动暂停**；120 MHz 或自动暂停需联系乐鑫定制（模组手册 1.2 注 3）。
- 仅 40 MHz 主晶振被固件支持，精度 **±10 ppm**；可选 32.768 kHz RTC 晶振（ESR ≤70 kΩ，偏置电阻 5~10 MΩ）。

### 2. 引脚可用性（生成 GPIO 代码前必须核对）

**绝对禁用（已连接封装内 flash/PSRAM）**

| GPIO | 占用 | 说明 |
| --- | --- | --- |
| 26~32 | SPICS1/SPIHD/SPIWP/SPICS0/SPICLK/SPIQ/SPID | Quad flash 数据线，模组未引出 |
| 33~37 | SPIIO4~SPIIO7 / SPIDQS | **Octal PSRAM 高 4 位数据线与 DQS**；模组引出的 **IO35/IO36/IO37 已连内部 PSRAM，不可用于其他功能** |

**受限使用（需确认复用冲突）**

| GPIO | 默认/约束 | 建议 |
| --- | --- | --- |
| 0 | strapping，弱上拉 = 1；与 GPIO46 共同决定启动模式 | 仅作 BOOT 按键，勿接大电容（会误入下载模式），建议外部上拉 |
| 3 | strapping，浮空，决定 JTAG 信号源，**必须由非高阻外部电路控制** | 普通用途时先确认 JTAG 选择 |
| 45 | strapping，弱下拉 = 0 → VDD_SPI = 3.3 V | 本模组保持默认 3.3 V，勿改电平 |
| 46 | strapping，弱下拉 = 0，参与启动模式与 ROM 日志 | 慎作输出 |
| 19 / 20 | USB **D− / D+**（P1 固定），上电有约 60 µs 毛刺，GPIO19/20 上电各两次高电平毛刺（总时长 3.2 ms / 2 ms） | 用作 USB；若改 GPIO 需显式关闭 USB 功能 |
| 43 / 44 | UART0 TX/RX，默认 ROM 日志与下载口 | 保留给日志/烧录，勿随意复用 |
| 39 / 40 / 41 / 42 | JTAG MTCK/MTDO/MTDI/MTMS | 需 JTAG 调试时不可用 |
| 47 / 48 | SPICLK_P/N 差分；仅 R16V 芯片下为 1.8 V 域 | N16R8 下可作普通 GPIO |
| 15 / 16 | XTAL_32K_P/N，ADC2_CH4/5 | 不接 32.768 kHz 晶振时可作 GPIO |
| 1~14 | RTC GPIO + Touch1~14 + ADC；仅 GPIO14 (TOUCH14) 可驱动屏蔽电极 | Deep-sleep 唤醒只能用 RTC 域 GPIO |

**推荐自由 GPIO（P2，无限制）**：1、2、4~14、17、18、21、38、47、48 中未被上述约束占用的部分。

**模组管脚速查**（WROOM-1，序号→GPIO）：
`1 GND | 2 3V3 | 3 EN | 4 IO4 | 5 IO5 | 6 IO6 | 7 IO7 | 8 IO15 | 9 IO16 | 10 IO17 | 11 IO18 | 12 IO8 | 13 IO19(USB D−) | 14 IO20(USB D+) | 15 IO3 | 16 IO46 | 17 IO9 | 18 IO10 | 19 IO11 | 20 IO12 | 21 IO13 | 22 IO14 | 23 IO21 | 24 IO47 | 25 IO48 | 26 IO45 | 27 IO0 | 28 IO35✗ | 29 IO36✗ | 30 IO37✗ | 31 IO38 | 32 IO39 | 33 IO40 | 34 IO41 | 35 IO42 | 36 RXD0(GPIO44) | 37 TXD0(GPIO43) | 38 IO2 | 39 IO1 | 40 GND | 41 EPAD(GND)`

**DevKitC-1 v1.1**：板载 RGB LED = **GPIO38**（初版为 GPIO48，注意区分）；J3 排针含 TX/RX(GPIO43/44)、GPIO0、GPIO45/46、GPIO19/20、GPIO35~42。

### 3. 启动配置（Strapping & eFuse）

| Strapping | 复位默认 | 作用 |
| --- | --- | --- |
| GPIO0 | 弱上拉 = 1 | 与 GPIO46 决定启动模式 |
| GPIO3 | 浮空 | JTAG 信号源选择（需外部确定电平） |
| GPIO45 | 弱下拉 = 0 | VDD_SPI 电压（0→3.3 V，1→1.8 V；本模组固定 3.3 V） |
| GPIO46 | 弱下拉 = 0 | 启动模式 + ROM 日志打印控制 |

- **SPI Boot（默认）**：GPIO0 = 1，GPIO46 任意。
- **Joint Download Boot**：GPIO0 = 0 **且** GPIO46 = 0，支持 USB-Serial-JTAG / USB-OTG / UART0 下载。自动下载电路靠 DTR/RTS 拉这两个脚。
- eFuse 默认值全 0，**一次性可编程**，烧成 1 不可恢复；`EFUSE_VDD_SPI_FORCE`、`EFUSE_VDD_SPI_TIEH`、`EFUSE_UART_PRINT_CONTROL`、`EFUSE_DIS_USB_SERIAL_JTAG_ROM_PRINT`、`EFUSE_DIS_PAD_JTAG`、`EFUSE_DIS_USB_JTAG`、`EFUSE_STRAP_JTAG_SEL`。
- Strapping 时序：**tSU ≥ 0 ms**（电源稳定到 EN 拉高），**tH ≥ 3 ms**（EN 拉高后保持到锁存采样完成）；strapping 值在复位时被锁存，之后可作普通 IO 读取，但整个上电周期内不变。
- JTAG 源：`EFUSE_STRAP_JTAG_SEL=0` 或 `GPIO3=1` → USB Serial/JTAG；`=1 且 GPIO3=0` → MTDI/MTCK/MTMS/MTDO 管脚；两者都烧 1 → 关闭 JTAG。

### 4. 电源、复位与时钟（硬件检查项）

- 单电源 3.3 V / ≥500 mA；主电源入口加 ESD 二极管 + ≥10 µF；每个数字电源脚就近 0.1 µF。
- 数字电源脚：pin46 `VDD3P3_CPU`、pin20 `VDD3P3_RTC`，范围 3.0~3.6 V。
- **CHIP_PU / EN 不得浮空**。上电时序：`tSTBL ≥ 50 µs`（电源稳定到 EN 拉高）、`tRST ≥ 50 µs`（EN 低于 `VIL_nRST` 的持续时间）；`VIL_nRST = −0.3 ~ 0.25×VDD3P3_RTC V`。
- 推荐 EN 处 RC 延时：**R = 10 kΩ，C = 1 µF**；慢上电/电池充电/频繁上下电场景需加电源监控芯片（阈值约 3.0 V），否则可能启动异常或 flash 擦除偶发失败。
- 晶振：仅 40 MHz，±10 ppm，幅值 >500 mV；XTAL_P 串 24 nH 电感抑制高次谐波；负载电容按 `CL = C1·C4/(C1+C4) + Cstray` 初算并实测校准频偏 ≤±10 ppm。
- RF：50 Ω 阻抗控制，芯片侧 CLC 匹配网络靠近芯片，天线端保留 π 匹配。
- USB：GPIO19=D− / GPIO20=D+，串 22/33 Ω 并预留对地电容；USB D+ 上电会在高低间波动，需稳定初始电平时加外部上拉。
- ADC：每个 ADC 输入对地加 0.1 µF 滤波；**优先 ADC1**（GPIO1~10）。ADC2（GPIO11~20）与 Wi-Fi 共用、且勘误 ADC-183 限制。
- Touch：串 470 Ω~2 kΩ（推荐 510 Ω）抑制耦合噪声并加强 ESD；触摸传感器**未通过传导抗扰度 (CS) 测试**，应用场景受限。
- SDIO：仅 1 个 SD/MMC 主机，所有 SDIO GPIO 加上拉并预留串阻。

### 5. 存储器映射（TRM 第 4 章）

| 地址范围 | 用途 |
| --- | --- |
| `0x3C00_0000 ~ 0x3DFF_FFFF` | 32 MB 外部存储器**数据**虚地址（DCache，64 KB 页映射 PSRAM，支持 8/16/32/128 位读写） |
| `0x3FC8_8000 ~ 0x3FCF_FFFF` | 480 KB 内部 SRAM（GDMA 可访问区） |
| `0x4000_0000 ~ 0x4005_FFFF` | 384 KB ROM（指令） |
| `0x4037_0000 ~ 0x403D_FFFF` | 448 KB 内部 SRAM（指令/数据） |
| `0x4200_0000 ~ 0x43FF_FFFF` | 32 MB 外部存储器**指令**虚地址（ICache，映射 flash/PSRAM） |
| `0x5000_0000 ~ 0x5000_1FFF` | 8 KB RTC FAST / SLOW 存储器（ULP 可访问，Deep-sleep 保持） |
| `0x6000_0000 ~ 0x600F_FFFF` | 836 KB 外设寄存器空间 |

- 外部 flash 与片外 RAM 各最大 1 GB，通过 Cache + MMU 以 **64 KB 块**映射；片外 RAM 支持 8/16/32/128 位读写，flash 只读数据空间仅支持读。
- Cache：指令 cache 16/32 KB（4/8 路组相连），数据 cache 32/64 KB（4 路），块 16/32 B，支持 pre-load / lock / critical-word-first。
- **GDMA 约束**：内部 RAM 可访问区 `0x3FC88000~0x3FCFFFFF`；外部 RAM 可访问区 `0x3C000000~0x3DFFFFFF` 且**只能突发模式**（块 16/32/64 B）。DMA 描述符地址必须落在内部 RAM，否则触发 `GDMA_ETXMEM_REJECT_INT`。
- 支持基于 XTS-AES 的片外 flash/PSRAM 硬件加解密。

### 6. 芯片勘误（ESP32-S3 Errata v1.3，全部 8 条，影响 v0.0/v0.1/v0.2）

| 编号 | 现象 | 对 v5.4 开发的要求 |
| --- | --- | --- |
| CACHE-126 | Cache 回写期间同 cache line 被访问可能返回错误数据 | ESP-IDF **v4.4.6+ / v5.0.4+ / v5.1.1+ / v5.2+ 已自动规避**（关中断 + cache freeze）；v5.4 无需处理，但禁止自行改写 cache 回写流程。无硬件修复计划 |
| RTC-126 | Light-sleep 关闭 RTC 外设后唤醒，CPU 读 RTC 域寄存器可能出错 | **不要在 Light-sleep 下掉电 RTC 外设**；v4.4+ 已规避，不影响功耗 |
| ANALOG-160 | `BIAS_SLEEP=0` 且 `PD_CUR=1` 会**永久损坏芯片**（ULP/触摸在睡眠中使用时可能触发） | 禁止该模拟电源配置；v4.4.2+/v5.0+ 已禁用，**不要手动改寄存器绕过** |
| LCD-239 | RGB 模式 `LCD_CLK_EQU_SYSCLK=1` 时 PCLK 无法下降沿触发且连续帧会串数据；I8080 模式 `ahead_cycle ≤ 2` 时首数据错误 | **RGB 不要把 `LCD_CAM_LCD_CLK_EQU_SYSCLK` 置 1**；I8080 保证 `ahead_cycle > 2`；v4.4.5+/v5.0.3+/v5.1+ 已规避 |
| USBOTG-4289 | Date Code < 2219 / PW < PW-2022-06-XXXX 的批次 `EFUSE_DIS_USB_OTG_DOWNLOAD_MODE` 默认烧 1，USB-OTG 下载不可用 | **优先用 USB-Serial-JTAG 下载**（GPIO19/20）或 UART0；v0.2 部分批次已修复；不需要该功能时烧 `EFUSE_DIS_USB_OTG_DOWNLOAD_MODE` 以避免 ROM 初始化外部 PHY 改变 IO 状态 |
| RMT-176 | RMT 连续 TX 停止后 idle 电平由回卷数据决定而非 end-marker | 设 `RMT_IDLE_OUT_EN_CHn=1`，只用寄存器控制 idle；**v5.0 起（含 v5.4）驱动已配置**，勿自行关闭 |
| TOUCH-100 | 前两次 `TOUCH_SCAN_DONE_INT` 中断的原始数据未定义 | **跳过前两次中断的 raw data**，然后关闭该中断；驱动层未完全封装，应用层要自行过滤 |
| ADC-183 | SAR ADC2 的数字控制器（DIG ADC2 / DMA）可能收到伪采样使能而卡死 | **ADC2 不要用连续(DMA)模式**，改用 RTC 控制器；优先把模拟输入放到 **ADC1 (GPIO1~10)** |

> 结论：ESP-IDF v5.4 已自动规避 CACHE-126 / RTC-126 / ANALOG-160 / LCD-239 / RMT-176；**仍需应用侧注意的只有 ADC-183（ADC2 DMA）、TOUCH-100（前两帧）、USBOTG-4289（下载方式）**。

### 7. 输出规则

- 生成引脚相关代码前，先输出一张「引脚 → 功能 → 冲突检查」表，确认无黑名单引脚。
- 涉及硬件结论时标注来源章节（如「C 模组手册 3.2 表 3-1 注 b」）。
- 若用户描述与上述事实冲突（例如要求把 GPIO35 当普通输出），必须明确指出并给出替代引脚，不得默默照做。

## 示例

```c
/* 合法示例：N16R8 上用 ADC1 + 一个自由 GPIO 驱动 LED */
#define LED_GPIO    GPIO_NUM_38   /* P2 自由脚（DevKitC-1 v1.1 板载 RGB LED，注意复用） */
#define ADC1_PIN    GPIO_NUM_4    /* ADC1_CH3，RTC GPIO，Touch4，P1 固定功能 */
/* 非法：GPIO_NUM_35 / 36 / 37 —— 已连内部 Octal PSRAM，禁止作为 GPIO */
/* 非法：GPIO_NUM_26 ~ 32 —— 封装内 Quad flash 数据线 */
```

```c
/* 启动模式判定（用于解释自动下载电路） */
/* GPIO0=1 → SPI Boot；GPIO0=0 且 GPIO46=0 → Joint Download Boot
 * 下载通道优先级：USB-Serial-JTAG (GPIO19/20) > UART0 (GPIO43/44)
 * 原因：勘误 USBOTG-4289 使早期批次 USB-OTG 下载不可用 */
```

## 检查清单

- [ ] 所有 GPIO 不在 26~37 黑名单内，且已核对 strapping/USB/UART0/JTAG 复用
- [ ] 模拟输入优先使用 ADC1；ADC2 未使用 DMA/连续模式
- [ ] 供电 3.3 V / ≥500 mA，EN 有 RC(10 kΩ + 1 µF)，tSTBL ≥ 50 µs、tRST ≥ 50 µs、tH ≥ 3 ms
- [ ] 晶振 40 MHz ±10 ppm，RF 50 Ω，USB 串阻与 ADC 滤波电容已预留
- [ ] sdkconfig 中 VDD_SPI/Flash/PSRAM 配置与 N16R8（3.3 V / 16 MB Quad / 8 MB Octal）一致
- [ ] 8 条勘误已逐条评估，特别是 ADC-183、TOUCH-100、USBOTG-4289
