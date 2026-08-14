+++
date = '2026-08-12'
title = 'lwIP：线程模型、缓冲区与网络故障闭环'
categories = ['embedded', 'network']
tags = ['lwIP', 'TCP/IP', 'pbuf', 'netif', 'DHCP', 'DNS', '网络诊断']
summary = '从 NO_SYS、raw/netconn/socket 与 tcpip_thread，到 pbuf 所有权、窗口和内存池，再用抓包与 stats 闭环网络故障。'
weight = 50
+++

# lwIP：线程模型、缓冲区与网络故障闭环

## 适用范围

本文以 **上游 lwIP 2.1.x** 的公开接口和配置项为基线。
芯片厂商通常会修改网卡驱动、OS 适配层、默认宏和调试出口；结论落地前必须保存 `lwipopts.h`、移植层版本与编译产物。
这里的 raw API 指 lwIP 的回调式 TCP/UDP API，不是 POSIX raw socket。

## 30 秒结论

> lwIP 的第一原则是先确定“谁拥有协议栈核心”和“谁拥有每个 pbuf”。`NO_SYS=1` 没有 OS 抽象，只能在同一执行上下文驱动 raw API；`NO_SYS=0` 通常由 `tcpip_thread` 串行执行核心代码，其他线程走 netconn/socket、消息投递或 core locking。性能问题不能只调大堆：要同时看 pbuf/内存池、TCP 发送与接收窗口、应用消费速度和网卡描述符。故障定位按“链路—地址—路由/DNS—传输—应用”分层，并把串口时间线、lwIP stats 与双端抓包对齐。

## 核心机制问答

### `NO_SYS` 到底决定什么

| 模式 | 核心执行方式 | 可用应用 API | 必做约束 |
| --- | --- | --- | --- |
| `NO_SYS=1` | 主循环/中断外的单一上下文主动轮询 | raw API | 周期调用 `sys_check_timeouts()`；不能使用 netconn/socket |
| `NO_SYS=0` | OS 适配层提供线程、邮箱、信号量，常由 `tcpip_thread` 拥有核心 | raw、netconn、socket | ISR 只收最小状态并投递；跨线程调用遵守端口的核心保护规则 |

`NO_SYS=1` 不等于“所有东西都在 ISR 里跑”。网卡 ISR 应确认来源、回收/投递描述符后退出，协议处理放在主循环。
`NO_SYS=0` 也不等于所有 lwIP 函数天然可重入；必须查函数属于 core API、线程安全封装还是移植层扩展。

### raw、netconn、socket 怎样选

| API | 模型 | 优点 | 代价与边界 |
| --- | --- | --- | --- |
| raw | 回调驱动，回调运行在核心上下文 | 内存和切换成本低，可精细控制复制 | 回调不能阻塞；PCB 与 pbuf 生命周期必须自己守住 |
| netconn | 顺序式 API，通过邮箱请求核心线程 | 比 raw 易写阻塞式任务 | 需要 OS；同一连接跨线程并发并非默认安全 |
| socket | POSIX 风格，通常建立在 netconn 之上 | 便于移植现有网络程序，可用 `select/poll` | 多一层描述符与邮箱；仍受 lwIP 配置和资源上限约束 |

选择依据不是“哪个更高级”，而是应用并发模型、RAM 预算、兼容性和可维护性。
即使 socket 对“调用线程到核心线程”做了保护，也不要让多个任务无协议地同时 `send/recv/close` 同一 fd；采用单所有者任务或明确启用并验证端口支持的 full-duplex 语义。

### `tcpip_thread` 和 core locking 是一回事吗

不是。常见安全路径有两类：

1. `tcpip_callback()`、`tcpip_try_callback()`、`netifapi_*()` 等把工作排到核心线程；
2. 启用 `LWIP_TCPIP_CORE_LOCKING` 后，任务先取得核心锁，再调用明确允许的 core API。

两者都要服从具体移植层。不要在 ISR 取得核心锁，也不要把长耗时业务包在核心锁内。
调试版可启用端口支持的 `LWIP_ASSERT_CORE_LOCKED()`，把偶发链表损坏尽早变成可定位断言。

### 一个包如何穿过 `netif` 与 `pbuf`

典型接收路径是：

`DMA 描述符 → 网卡驱动 → pbuf → netif->input → Ethernet/IP → TCP/UDP → 回调或邮箱 → 应用`

发送路径反向经过路由选择、协议头构造、`netif->output/output_ip6` 和驱动 `linkoutput`。
`netif` 表示一个网络接口，维护地址、MTU、状态、链路状态、输出函数和统计信息。
“接口 up”是管理状态，“link up”是物理/二层状态；两者都满足后 DHCP 才有成功前提。

### `pbuf` 的 `len`、`tot_len` 与类型怎样理解

- `len` 是当前节点有效长度，`tot_len` 是从当前节点到链尾的总长度；解析协议头不能假设数据连续。
- `PBUF_RAM` 的头和载荷通常动态分配，适合需要改写或拼头的缓冲。
- `PBUF_POOL` 来自固定池，常用于接收；把它长期扣在应用层会直接饿死后续收包。
- `PBUF_ROM`/`PBUF_REF` 引用外部存储，外部数据必须活到协议栈不再引用为止。
- `pbuf_ref()` 增加引用，`pbuf_free()` 减少引用并可能释放一整段链；返回值不是字节数。

读跨节点字段优先使用 `pbuf_copy_partial()` 或逐段处理，不要直接把 `payload` 强转成完整报文结构。

### 零拷贝真正省了什么，谁负责释放

零拷贝省的是载荷复制，不会消除描述符、缓存一致性、引用计数和对齐成本。
RX 常用 `pbuf_custom` 包装 DMA buffer：驱动把所有权交给协议栈，只有 custom free 回调触发后才能把描述符还给 DMA。
TX 若使用“不复制”语义，应用缓冲必须保持不变，直到相应 API/回调明确表示协议栈不再引用；不能函数返回就复用栈数组。

评审零拷贝方案必须回答：

- buffer 起点、长度和 cache line 是否满足 DMA 约束；
- CPU 与 DMA 交接前后由谁 clean/invalidate；
- 错误、重传、分片和接口 down 时谁回收；
- 引用耗尽时是丢包、复制降级还是施加背压；
- 统计项能否证明“少复制”没有换来描述符饥饿。

### 内存池、TCP 窗口与吞吐为什么要一起看

`MEM_SIZE` 只代表 lwIP heap 的一个维度，常见瓶颈还包括 `MEMP_NUM_TCP_SEG`、PCB 数、API message、超时项、`PBUF_POOL_SIZE` 和网卡 RX/TX 描述符。
TCP 发送侧重点看 `TCP_SND_BUF`、`TCP_SND_QUEUELEN` 和 segment 池；接收侧看 `TCP_WND`、应用读取速度以及 raw API 是否及时调用 `tcp_recved()` 归还窗口。

带宽时延积约为 `吞吐目标 × RTT`。
窗口明显小于带宽时延积时，链路再快也会周期性等待 ACK；盲目放大窗口又会抬高峰值 RAM、队列时延和丢包恢复成本。
先记录峰值、失败计数和最慢消费路径，再按证据改配置。

### DHCP 和 DNS 成功分别证明了什么

DHCP 成功说明接口完成了租约交换并取得一组地址参数，不证明互联网、DNS 或业务端口可达。
排查时记录 Discover/Offer/Request/ACK、租约时间、地址/掩码/网关和 DNS server。
静态地址切换前后要按端口要求停止 DHCP，并处理旧路由、ARP 与已有连接。

raw DNS 查询可能同步命中缓存并返回 `ERR_OK`，也可能返回 `ERR_INPROGRESS` 后在回调给结果；回调参数和查询上下文必须活到完成或取消。
域名失败时用“直接访问已知 IP”区分 DNS 与路由/TCP/TLS 问题，并核对 DHCP 是否真的下发了可用 DNS。

### 抓包和 stats 应该怎样配合

打开调试构建中的 `LWIP_STATS`、`LINK_STATS` 及端口显示入口，至少采集：

- link 层收发、丢弃和错误；
- pbuf、heap 与各 memp 池的 current/max/error；
- TCP 重传、RST、超时和 PCB 数；
- 网卡描述符不足、DMA 错误与驱动队列水位。

设备日志用单调时钟，记录 netif、连接代次、四元组、错误码和关键 seq/ack。
抓包尽量同时取 AP/交换机侧与服务器侧；只抓设备外部看不到“驱动收到但协议栈丢弃”，只看 stats 又看不到对端窗口、重传和 RST。

## 项目追问

- 为什么项目选择 socket 而不是 raw；切换后 RAM、吞吐和维护成本各变化多少？
- 网卡 ISR、驱动任务、`tcpip_thread` 和业务任务的边界怎样画，哪个队列提供背压？
- 一块 RX buffer 从 DMA 到应用再回到 DMA 的所有权时间线是什么？
- 最大并发连接、峰值 pbuf、最坏 RTT 和发送窗口是怎样量出来的？
- Wi-Fi/以太网断链后，旧 PCB、DNS 缓存和业务状态怎样失效？
- 线上“偶发超时”靠哪些日志、统计和 pcap 被缩小到具体一层？

## 诊断剧本

### 接口有链路但拿不到地址

1. 确认 `netif_is_up()` 与 `netif_is_link_up()` 的状态变化顺序。
2. 抓 DHCP 四步，若没有 Discover，查超时驱动、核心上下文和 UDP/pbuf 分配失败。
3. 有 Discover 无 Offer，查 VLAN、AP 隔离、服务器地址池；不要先改 TCP 参数。
4. 有 ACK 但应用仍报离线，打印实际地址、掩码、网关、DNS 和默认 netif。

### TCP 跑一段时间后停顿

1. 对齐应用写入/读取速率、socket 超时、TCP 重传和零窗口事件。
2. 看 `PBUF_POOL`、`MEMP_TCP_SEG`、发送队列和网卡描述符的峰值/error。
3. 抓包区分对端零窗口、本端未 ACK、路径丢包和主动 RST。
4. 检查 raw 回调是否阻塞、是否遗漏 `tcp_recved()`、是否跨线程碰 PCB。

### 压测后内存不回落

1. 在连接建立、关闭、错误回调和接口 down 处记录 PCB/pbuf 计数。
2. 对每个 `pbuf_ref`、custom pbuf 和异步 DNS 上下文画成对释放路径。
3. 区分“缓存保留高水位”和持续增长；重复固定轮次比较稳定平台。
4. 注入 connect 失败、对端 RST、拔线与队列满，覆盖非正常回收分支。

## 可验证实验

### 实验一：证明核心线程边界

在 raw 回调、socket 任务与驱动接收点打印线程 ID；从错误线程故意调用带核心断言的测试函数，确认调试固件可立即捕获违规。

### 实验二：证明 pbuf 所有权

给每个 custom RX buffer 加编号和状态机 `DMA_OWNED → STACK_OWNED → DMA_OWNED`，压测同时断链；验收条件是无重复归还、无悬挂且池高水位稳定。

### 实验三：窗口与吞吐曲线

在固定 RTT/丢包环境下阶梯修改窗口和发送缓冲，记录吞吐、P99 时延、峰值 RAM 与重传；用带宽时延积解释拐点，而不是只展示最好数字。

### 实验四：DHCP/DNS 故障注入

依次阻断 DHCP Offer、配置错误 DNS、让域名返回失败、直接访问 IP；保存设备日志、stats 和 pcap，验证分层告警是否准确。

## 易错纠偏

- 错：“`NO_SYS=1` 就在中断中直接跑协议栈。” 对：它只表示没有 OS 层，仍应限制 ISR 工作并在单一上下文驱动核心。
- 错：“socket API 线程安全，所以一个 fd 可随便被多任务关闭和收发。” 对：默认只保证到核心的调用路径，连接级并发仍需所有权协议。
- 错：“`PBUF_POOL` 越大越好。” 对：它会占静态 RAM，也可能掩盖应用长期持包；先找所有权和背压问题。
- 错：“零拷贝等于不管理内存。” 对：零拷贝把复制成本换成更严格的生命周期和 cache/DMA 协议。
- 错：“收到 DHCP ACK 就代表业务在线。” 对：还要分别验证路由、DNS、TCP/TLS 与应用鉴权。
- 错：“抓到重传就是 lwIP 有 bug。” 对：重传只是现象，要结合 ACK、窗口、双端抓包和本地丢弃计数定位责任层。

## 交叉链接

- [RTOS：调度、同步与实时故障诊断]({{< relref "rtos.md" >}})：理解邮箱、核心线程和 ISR 边界。
- [MCU：启动、中断、DMA 与低功耗]({{< relref "mcu.md" >}})：补齐网卡 DMA、cache 与描述符所有权。
- [Linux / 驱动主干]({{< relref "linux-driver.md" >}})：对照 Linux NAPI、socket 与驱动诊断方法。
- [乐鑫 Connectivity 专题]({{< relref "../company/espressif-connectivity.md" >}})：把 lwIP 放进 Wi-Fi 事件生命周期。
- [嵌入式岗位知识地图]({{< relref "../interview/embedded-roadmap.md" >}})：按项目证据组织复习。

## 官方资料

- [lwIP 2.1.x 模块索引](https://www.nongnu.org/lwip/2_1_x/modules.html)
- [lwIP raw 回调式 API](https://www.nongnu.org/lwip/2_1_x/group__callbackstyle__api.html)
- [lwIP 线程与核心锁说明](https://www.nongnu.org/lwip/2_1_x/multithreading.html)
- [lwIP pbuf API](https://www.nongnu.org/lwip/2_1_x/group__pbuf.html)
- [lwIP netif API](https://www.nongnu.org/lwip/2_1_x/group__netif.html)
- [lwIP DHCPv4](https://www.nongnu.org/lwip/2_1_x/group__dhcp4.html)
- [lwIP DNS](https://www.nongnu.org/lwip/2_1_x/group__dns.html)
- [lwIP stats 配置项](https://www.nongnu.org/lwip/2_1_x/group__lwip__opts__stats.html)
