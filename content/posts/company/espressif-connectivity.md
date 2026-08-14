+++
date = '2026-08-12'
title = '乐鑫 Connectivity：从 Wi-Fi 事件到安全 OTA'
categories = ['company', 'embedded']
tags = ['Espressif', 'ESP-IDF', 'Wi-Fi', 'BLE', 'MQTT', 'TLS', 'OTA']
summary = '以公开 ESP-IDF 5.x API 拆解 esp_event、esp_netif、lwIP、STA 重连、配网、共存、功耗以及 MQTT/TLS/OTA 的工程闭环。'
weight = 10
+++

# 乐鑫 Connectivity：从 Wi-Fi 事件到安全 OTA

## 适用范围与版本边界
本文只使用乐鑫公开文档和公开示例，代码语义以 **ESP-IDF 5.2 的 ESP32 文档** 为固定参照，再指出 5.x 中需要按目标核对的边界。
不同芯片的无线能力不同：ESP32-S2 没有 Bluetooth，ESP32-C3/S3 的 Bluetooth 能力与经典 ESP32 不同，ESP32-C6 还包含 802.15.4，ESP32-H2 不提供 Wi-Fi。
不要把一个 target 的 Kconfig、共存能力、射频校准、睡眠唤醒行为或吞吐数字复制到整个系列。
老项目中的 `tcpip_adapter`、旧 system event API 与新项目的 `esp_netif`、`esp_event` 不能混写；迁移时以所用 IDF release 文档为准。

## 30 秒结论
> ESP-IDF Connectivity 的主线不是“调用 `esp_wifi_connect()`”，而是维护一个跨 Wi-Fi 驱动、事件循环、`esp_netif`、lwIP 与应用协议的连接代次。`WIFI_EVENT_STA_CONNECTED` 只代表二层关联完成，业务应在 `IP_EVENT_STA_GOT_IP` 后建 socket；收到 `WIFI_EVENT_STA_DISCONNECTED` 或 IP 变化时，关闭并作废旧 socket，再退避重连。配网要处理凭据安全和失败回滚，共存与低功耗要量化延迟/吞吐/电流，MQTT、TLS、OTA 则必须把时间、证书、分区、签名和回滚串成完整交付链。

## 核心机制问答
### `esp_event → esp_netif → lwIP` 的链路是什么
常见初始化顺序是：

1. `nvs_flash_init()` 准备 Wi-Fi 与应用持久化所需的 NVS；
2. `esp_netif_init()` 初始化 TCP/IP 适配层和相关 lwIP 上下文；
3. `esp_event_loop_create_default()` 创建默认事件循环；
4. `esp_netif_create_default_wifi_sta()` 创建 STA netif 并安装默认处理器；
5. `esp_wifi_init()`、设模式/配置、`esp_wifi_start()`；
6. 应用事件处理器只转发状态，由连接管理任务执行重连和协议启停。

接收数据路径可概括为：

`Wi-Fi MAC/驱动任务 → esp_netif 驱动 glue → lwIP netif/tcpip_thread → socket → 应用任务`

控制路径则通过 `WIFI_EVENT` 和 `IP_EVENT` 驱动状态机。
默认事件处理器会维护 netif 与 DHCP；应用不要抢在默认处理之前手工伪造“已联网”状态。

### 事件回调里能不能直接做业务
事件回调运行在事件任务上下文，应该做常数时间工作：复制必要字段、更新原子状态、投递队列/事件组，然后返回。
不要在回调中长时间 TLS 握手、等待队列、做 DNS、写大块 flash 或无限重试，否则会延迟同一事件循环上的其他处理器。
如果事件数据只在回调期有效，投递前必须复制；不能把临时指针交给异步任务。

### Wi-Fi STA 的关键状态怎样划分
| 状态/事件 | 已经证明 | 业务动作 |
| --- | --- | --- |
| `WIFI_EVENT_STA_START` | 驱动进入 STA 可启动阶段 | 发起首次连接或唤醒连接管理器 |
| `WIFI_EVENT_STA_CONNECTED` | 已与 AP 完成二层关联 | 等 DHCP/IP，不能宣称云在线 |
| `IP_EVENT_STA_GOT_IP` | netif 获得 IPv4 参数 | 记录地址与连接代次，启动 DNS/TLS/MQTT |
| `WIFI_EVENT_STA_DISCONNECTED` | 当前二层连接失效 | 作废旧 fd，停上层会话，按原因与策略退避重连 |
| `IP_EVENT_STA_LOST_IP` | 地址已失效 | 阻止新业务，并等待恢复或重建会话 |

`GOT_IP` 事件中的 IP change 信息也必须处理。
IP 改变时旧连接的源地址与路由假设不再成立，应关闭并重建 socket，而不是只改一个全局 `connected=true`。

### 为什么断连后的旧 socket 不能继续用
ESP-IDF 的 Wi-Fi 指南明确提示：STA 断连时 lwIP 连接会被清理，应用 socket 处于错误状态。
fd 是小整数，关闭后还可能被新 socket 复用；仅比较 fd 值无法阻止“旧任务写入新连接”。

可靠做法是维护单调递增的 `connection_epoch`：

- `GOT_IP` 创建本代协议任务与 socket；
- `DISCONNECTED` 先增加 epoch，再通知任务取消和关闭 fd；
- 每次异步回调、发送和重试都校验捕获的 epoch；
- 只有当前代且 IP ready 时才允许发布业务数据；
- 旧代清理超时也不能阻塞新一代连接管理器。

### 重连策略为什么不能只有无限立即重试
立即重试会在 AP 不可达、密码错误或认证拒绝时持续占用射频、CPU 和电量，也可能淹没日志。
将断连 reason 分类为瞬态、配置错误、用户主动断开和系统恢复；对瞬态错误使用带抖动的指数退避并设置上限。
密码/认证类连续失败应进入可观测的“需重新配网”，不能一直伪装成网络波动。
记录每轮扫描、关联、DHCP 和云握手耗时，才能知道恢复慢在哪里。

### 配网方案怎样选

公开组件常见路径包括 BLE transport、SoftAP transport、控制台/量产注入，以及目标与版本支持时的其他标准方案。
选择要同时评估手机兼容性、BOM、现场网络、交互、功耗和安全，而不是只比较 demo 成功率。

配网状态机至少包括：

`未配置 → 开启配网 → 收到候选凭据 → 限时联网验证 → 持久化/提交 → 关闭配网`

失败时回到可恢复状态，不能覆盖最后一份已验证凭据。
生产方案应使用 Proof of Possession 或等价身份校验，限制尝试次数和配网窗口，清理临时会话，并评估 NVS encryption、flash encryption 与 secure boot 的组合。
日志不得打印 SSID 对应密码、PoP、token、证书私钥或完整云端凭据。

### Wi-Fi 与 BLE 共存要观察什么

二者共享 2.4 GHz 射频资源时，共存模块进行时隙/优先级协调，但结果仍受芯片、IDF、Wi-Fi 模式、BLE connection interval、包长和空口环境影响。
不要把“共存已开启”当成性能保证，也不要在没有对应 target 文档时硬编码内部仲裁细节。

需要一起记录：

- Wi-Fi RSSI、重传、吞吐、P95/P99 RTT 与断连次数；
- BLE 连接间隔、通知延迟、丢包/重传与配网完成时间；
- 共存相关 Kconfig、任务优先级/绑核、Wi-Fi power save 模式；
- 同信道干扰、AP 信道与测试距离；
- 峰值和平均电流，而不只看平均吞吐。

先通过降低业务 burst、调整 BLE interval/包长和 Wi-Fi 发送节奏验证资源竞争，再决定是否改任务或射频配置。

### 功耗模式怎样影响连接语义

Wi-Fi modem-sleep 尽量保留关联，通过 DTIM/监听窗口在功耗与下行时延之间取舍。
light-sleep 会暂停更多数字逻辑，唤醒延迟和外设状态取决于 target、时钟与电源管理配置。
deep-sleep 更像一次受控重启：RAM、socket、MQTT/TLS 会话通常不能视作仍然有效，应用要设计唤醒后的恢复路径。

ESP-IDF 版本和 target 的默认 Wi-Fi power save 可能变化，测试报告必须写出 `esp_wifi_set_ps()` 取值、AP DTIM、流量模型、温度和供电条件。
“平均电流最低”不一定是产品最优；还要比较唤醒到首包、云端重连时间与消息积压。

### MQTT、TLS 与 OTA 如何串成一条安全链

TLS 前置条件包括可信时间策略、信任锚、hostname 校验、随机数与足够握手内存。
不要用“跳过证书校验”解决时间或证书配置问题；测试证书与生产信任链要分开。

MQTT 侧要定义 client ID、会话语义、QoS、LWT、keepalive、离线队列上限与重连退避。
网络恢复不代表 publish 已被业务消费；QoS 与应用幂等、消息 ID/去重需要一起设计。

HTTPS OTA 至少验证：

- 分区表容纳当前与下一镜像，并预留 NVS/phy/回滚需求；
- 下载使用 TLS 校验，镜像使用签名/secure boot 信任链；
- 版本与 anti-rollback 策略不会锁死维修路径；
- 首次启动完成自检后调用公开 API 确认有效，否则 bootloader 能回滚；
- 断电、断网、空间不足、证书轮换和坏镜像都有明确错误码与恢复动作。

flash encryption 保护静态内容，secure boot 验证启动代码来源；二者目的不同，不能互相替代。

## 项目追问

- 为什么以 `GOT_IP` 而不是 `STA_CONNECTED` 启动 MQTT？IPv6 场景又怎样定义 ready？
- 断网 30 秒期间产生的数据存在哪里，容量、淘汰和重放顺序怎样确定？
- connection epoch 如何阻止旧任务误写新 fd；有怎样的故障注入证据？
- BLE 配网与 Wi-Fi 扫描冲突时，哪一个指标先恶化，改动依据是什么？
- 低功耗优化前后，平均电流、P99 下行延迟与云端恢复耗时各是多少？
- TLS 证书过期、设备时间错误和内存不足在日志上怎样区分？
- OTA 在第几阶段允许掉电，如何证明旧版本仍可启动且配置兼容？

## 诊断剧本

### 关联成功但一直没有 `GOT_IP`

1. 区分 `STA_CONNECTED` 与 `GOT_IP`，打印 netif、DHCP 状态及事件时间线。
2. 抓 DHCP Discover/Offer/Request/ACK；无 Discover 时查默认 netif handlers 和任务阻塞。
3. 有 Discover 无 Offer 时查 AP 地址池、隔离/VLAN 和空口，不先改 MQTT。
4. 有 ACK 无事件时查 `esp_netif` 绑定、事件注册生命周期和地址冲突。

### 周期性断连且旧 MQTT 任务不退出

1. 记录 Wi-Fi disconnect reason、epoch、fd 创建/关闭与 MQTT event。
2. 在断连点先失效 epoch，再取消 DNS/TLS/socket；为每一步设置超时。
3. 用 AP 断电、远离 AP、切换 SSID 参数复现，检查是否出现 fd 复用误写。
4. 对齐空口/路由器抓包，区分 AP 踢下线、射频差和应用 keepalive 失败。

### BLE 配网时 Wi-Fi 延迟尖峰

1. 固定 AP 信道、距离、IDF/target 与流量，建立只有 Wi-Fi 的基线。
2. 加入固定 BLE interval/通知负载，记录 Wi-Fi RTT 和 BLE 延迟分位数。
3. 核对共存 Kconfig、功耗模式和任务负载，不依赖单次 RSSI。
4. 逐项改变一个变量，确认改进没有以断连率或电流为代价。

### TLS/OTA 只在部分设备失败

1. 分类为 DNS、TCP、TLS alert、证书/时间、HTTP、flash 写入、镜像校验或启动回滚。
2. 记录 target、IDF commit、分区表 hash、证书 bundle 版本和最大空闲连续堆。
3. 对坏网络、错误时间、证书轮换、低电压和下载中断做矩阵测试。
4. 禁止在正式固件中通过关闭 hostname/证书校验绕过失败。

## 可验证实验

### 实验一：STA 生命周期与旧 socket

建立持续 TCP echo，定时关闭 AP 再恢复；验收为每次断连旧 epoch 全部退出，只在新 `GOT_IP` 后创建 fd，且无跨代数据。

### 实验二：配网提交与回滚

输入错误密码、断电、再输入正确密码；证明最后一份已验证凭据不会被候选值提前覆盖，配网窗口超时后不再暴露入口。

### 实验三：共存压测

同时跑固定速率 iperf 与 BLE notification，扫描多个 connection interval；输出 Wi-Fi/BLE 延迟分位数、吞吐、丢包和电流曲线，并保存 sdkconfig。

### 实验四：功耗状态机

对 no-sleep、modem-sleep、light-sleep 和 deep-sleep 分别测 24 小时能耗、下行首包、重连时间与消息完整性，明确唤醒源和 AP DTIM。

### 实验五：OTA 故障注入

在 DNS、TLS、下载 30%、写完未切换、首次启动自检等阶段断网/断电；验收为设备总能进入旧版、新版或受控恢复模式，且可读取失败阶段。

## 易错纠偏

- 错：“`STA_CONNECTED` 就能创建云连接。” 对：它只证明二层关联，IPv4 业务通常等待 `IP_EVENT_STA_GOT_IP`。
- 错：“自动重连后旧 socket 会恢复。” 对：断连/IP 变化后关闭并重建 socket，用 epoch 隔离旧异步任务。
- 错：“事件回调里直接做 TLS 最及时。” 对：回调应快速转发，耗时状态机放专用任务。
- 错：“BLE 配网只是换一种 UI。” 对：它涉及无线共存、身份校验、凭据存储、超时和回滚。
- 错：“开启省电只影响吞吐。” 对：还会影响 DTIM 下行延迟、连接恢复、定时器与外设唤醒。
- 错：“HTTPS OTA 已经安全，所以不用镜像签名。” 对：传输安全、启动真实性与静态数据保护是不同边界。
- 错：“ESP32 系列的无线能力都一样。” 对：先锁定芯片 target、revision、IDF release 与 sdkconfig。

## 交叉链接

- [lwIP：线程模型、缓冲区与网络故障闭环]({{< relref "../embedded/lwip.md" >}})：socket、pbuf、DHCP/DNS 与抓包主干。
- [RTOS：调度、同步与实时故障诊断]({{< relref "../embedded/rtos.md" >}})：事件任务、队列、优先级和 ISR 边界。
- [MCU：启动、中断、DMA 与低功耗]({{< relref "../embedded/mcu.md" >}})：睡眠、唤醒、flash 与硬件证据。
- [公司专题路线]({{< relref "_index.md" >}})：专题的公开资料与 NDA 边界。
- [嵌入式岗位知识地图]({{< relref "../interview/embedded-roadmap.md" >}})：把专题改写成项目证据。

## 官方资料

- [ESP-IDF 5.2 Wi-Fi 驱动与 STA 场景](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-guides/wifi.html)
- [ESP-NETIF 编程指南](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-reference/network/esp_netif.html)
- [ESP Event Loop Library](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-reference/system/esp_event.html)
- [Wi-Fi Provisioning Manager](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-reference/provisioning/wifi_provisioning.html)
- [Wi-Fi 与 Bluetooth 共存](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-guides/coexist.html)
- [ESP-IDF 低功耗模式](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-guides/low-power-mode.html)
- [ESP-MQTT 文档](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-reference/protocols/mqtt.html)
- [ESP HTTPS OTA](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-reference/system/esp_https_ota.html)
- [ESP-IDF OTA API 与回滚](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/api-reference/system/ota.html)
- [ESP-IDF Security Guides](https://docs.espressif.com/projects/esp-idf/en/release-v5.2/esp32/security/index.html)
