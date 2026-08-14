+++
date = '2026-08-12T00:00:00+08:00'
title = 'Linux 与驱动：从设备树到可观测的硬件服务'
categories = ['embedded', 'linux', 'driver']
tags = ['Linux', '驱动', '设备树', '中断', 'DMA', 'BSP', '面试']
summary = '围绕启动、设备模型、probe、中断、DMA、用户态接口与 PM，建立 Linux 驱动排障主线。'
weight = 40
+++

# Linux 与驱动：从设备树到可观测的硬件服务

适用范围：现代 Linux platform driver 和设备树平台；内核版本、总线类型和子系统会改变具体 API。优先采用目标内核文档与现有子系统范式。

## 30 秒总答

> Linux 驱动不是“读写几个寄存器”，而是把固件描述或可枚举硬件接入设备模型，按资源依赖完成 probe，为内核子系统或用户态提供受控接口，并在并发、DMA、电源和卸载路径上保持生命周期正确。排障先确认设备是否存在、是否匹配、资源是否就绪，再进入寄存器和业务逻辑。

## 从上电到驱动 probe 的链路

典型嵌入式 Linux 启动链：

`Boot ROM → 多级固件/Bootloader → Kernel 解压与早期初始化 → initramfs/rootfs → init/systemd → 用户服务`

Bootloader 通常向内核传递 DTB 和启动参数。设备树描述**硬件事实与连接关系**，驱动描述控制逻辑。不要把策略、运行状态或板上不存在的设备塞进 DTS。

platform 设备通常由 DT/ACPI/板级代码枚举；platform driver 提供 `probe()` 和卸载/关闭相关回调。匹配成功不等于 probe 一定成功，资源供应者可能尚未就绪。

## device、driver、bus、class 各是什么

- `device`：一个设备实例和它的生命周期、父子关系、资源；
- `driver`：能绑定某类设备的实现；
- `bus`：定义设备与驱动如何匹配；
- `class`：按用户可见功能组织设备，不等于物理总线。

驱动注册与设备出现的先后通常都可以，driver core 在双方可用时尝试匹配。`compatible` 不是用户随意命名的字符串，应遵循 binding，并保持从具体到兼容项的顺序。

## `probe()` 不执行怎样排查

按层收证据：

1. 最终 DT 中节点是否存在且 `status = "okay"`；
2. 运行中 `/proc/device-tree` 或 `/sys/firmware/devicetree/base` 是否与预期 DTB 一致；
3. 驱动是否被 Kconfig 选中、编进内核或模块已加载；
4. 设备与驱动的 modalias / compatible 是否匹配；
5. `dmesg` 是否有 probe 错误或 deferred probe；
6. clock、regulator、reset、GPIO、IOMMU 等 supplier 是否就绪；
7. probe 是否返回了错误但日志丢失了错误码。

`-EPROBE_DEFER` 的含义是依赖资源暂不可用，driver core 稍后重试；它不是通用“先忽略错误”。若 binding、资源名字或 provider 永远不成立，设备会一直推迟。

## probe 中如何管理资源

典型顺序：解析/取得资源 → 启用供电和时钟 → 解除复位 → 初始化硬件 → 申请 IRQ/DMA → 注册上层接口。失败路径必须按逆序撤销。

`devm_*` 资源由 device 生命周期管理，能简化 probe 失败和 remove，但不意味着所有顺序问题消失：硬件停机可能必须早于某些资源自动释放，仍需显式 action 或回调。

不要把 `probe()` 写成吞掉所有错误的巨型函数。保留原始 errno，并在资源边界加可定位的日志。

## MMIO 为什么不用普通 `volatile *`

驱动通过 `devm_ioremap_resource()` 等映射资源，并用 `readl()/writel()` 家族访问。原因包括地址空间、访问宽度、端序、编译器与架构所需的 I/O 顺序语义。

普通 `volatile` 只约束 C 抽象机中的访问优化，不等于设备访问 API、CPU 屏障或 DMA 一致性。还要注意：

- 某些总线写是 posted write，需要读回或规定方法确保到达；
- 对 W1C/W1S 寄存器做通用读改写可能清错位；
- 寄存器可被 IRQ 和任务并发访问时，要保护软件层不变量；
- 时钟/电源域未开启时，读寄存器可能返回假值或触发总线错误。

## hardirq、threaded IRQ 与 workqueue 怎样选

hardirq 上下文不能睡眠，工作必须有界。典型顶半部确认/清中断、采样状态，再唤醒线程化中断或 workqueue。

- **threaded IRQ**：处理与该 IRQ 紧密相关、允许睡眠的后续工作；
- **workqueue**：通用异步进程上下文，可排队和合并工作；
- **NAPI**：网络高包率下从逐包中断切换到有预算的轮询；
- tasklet 是历史机制，新设计应优先参考目标子系统当前范式。

互斥锁可睡眠，不能在 hardirq/atomic context 使用；自旋锁不会睡眠，但临界区必须短，并正确处理本地 IRQ 与同一数据的关系。

## 中断风暴怎样定位

1. 对比 `/proc/interrupts` 计数与真实设备事件；
2. 检查触发类型（边沿/电平、极性）与 DT；
3. 确认先读状态还是先清中断，以及硬件规定的 ack 顺序；
4. 对电平中断，退出前确认源条件已经消失；
5. 测 handler 执行时长和重入间隔；
6. 共享 IRQ 必须判断是不是本设备并正确返回；
7. 用示波器/逻分确认引脚，而不是只读软件计数。

## DMA 地址为什么不等于 CPU 地址

至少区分：CPU 虚拟地址、CPU 物理地址和设备看到的 DMA 地址。IOMMU、总线窗口与架构会让它们不同，不能把 `virt_to_phys()` 结果直接交给设备。

- coherent 分配适合描述符等长期共享控制结构，但仍需遵守所有权/顺序；
- streaming mapping 适合有明确传输方向和 map/unmap 或 sync 边界的数据；
- direction 必须准确，DMA mask 必须在 probe 设置和检查；
- 成功提交前 CPU 写完，完成通知后 CPU 才重新接管 buffer；
- 不能在设备仍拥有 buffer 时释放、复用或随意读取。

## IOMMU fault 的诊断顺序

先记录 fault IOVA、访问方向、stream/device 标识，再对照描述符和 mapping 生命周期：地址/长度是否越界、SG 是否终止、direction 是否一致、是否过早 unmap、设备是否用了旧描述符。只增大 reserved-memory 通常掩盖不了生命周期 bug。

## 字符设备怎样支持阻塞、非阻塞与 poll

驱动应维护一个真实条件（例如 RX 队列非空），而不是把“被唤醒”当作数据已经存在。

```c
if (file->f_flags & O_NONBLOCK) {
    if (!data_ready(dev))
        return -EAGAIN;
} else {
    ret = wait_event_interruptible(dev->readq, data_ready(dev));
    if (ret)
        return ret;
}
```

`poll()` 注册同一 wait queue，并根据当前条件返回 mask。若条件未消费却一直报告 readable，用户态 epoll 会忙循环。`copy_to_user/from_user` 可能失败，返回值必须处理；用户指针不能直接解引用。

`mmap` 能减少复制，但会放大权限、缓存属性、生命周期和设备拔出问题，应优先使用子系统已有接口。

## Runtime PM 的常见陷阱

Runtime PM 用 usage count 和依赖关系控制设备按需开关。常见错误包括 get/put 不配对、autosuspend 时仍有 DMA、resume 恢复顺序错误、系统 suspend 与 runtime suspend 状态机互相覆盖。

资源顺序常是：resume 时供电 → clock → reset → 寄存器恢复 → IRQ/DMA；suspend 时反向执行。具体顺序由硬件决定，应通过 trace 和寄存器基线验证。

## 项目追问

### 从参考板移植到新板，最小改动是什么

先列硬件差异：电源、时钟、pinmux、reset、地址/IRQ、外设连接和时序。优先改板级 DTS/overlay 与 binding 允许的属性，通用驱动通过匹配数据表达 SoC 差异，避免在驱动里散落 board-name 判断。

### 为什么使用中断加 DMA，而不是轮询

给出数据率、CPU 占用、批量大小、最坏延迟、缓存维护和错误恢复。DMA 不是“零 CPU”，提交、完成、中断和一致性都有成本；低速小数据轮询可能更简单。

### 驱动怎样面向量产排障

至少提供稳定错误码、关键状态统计、dynamic debug/tracepoint、超时与恢复次数、版本信息和持久化崩溃日志；不能依赖不断打印高频日志。

## 板级 Bring-up 诊断剧本

1. 建立正常启动基线：串口从最早阶段到用户态的时间点；
2. 新板先确认电源、复位、boot strap、参考时钟和可读存储；
3. 打开 earlycon，区分“内核没启动”和“console 没配置”；
4. 确认实际加载的 DTB/overlay 和 kernel/module 版本；
5. 按 provider → consumer 检查 regulator、clock、reset、pinctrl；
6. probe 成功后再查 IRQ、DMA 和用户态接口；
7. 用单变量实验替换模块或禁用功能，形成反证；
8. 修复后回归冷启动、热重启、suspend、压力和异常断电。

## 常用取证工具

| 工具/入口 | 回答的问题 |
| --- | --- |
| `dmesg -w` / dynamic debug | probe、错误码、状态变化 |
| sysfs / debugfs | 绑定、资源、子系统内部状态 |
| `/proc/interrupts` | IRQ 是否到达、分布是否异常 |
| ftrace / trace-cmd | 函数、IRQ、调度与 PM 时间线 |
| lockdep | 锁顺序和原子上下文误用 |
| KASAN/KCSAN/UBSAN | 越界、竞态和未定义行为线索 |
| pstore/ramoops | 重启后保留 panic/oops |
| `devmem` | 仅受控实验；不能替代正确驱动 |

## 最小实验

为一个虚拟或真实 platform device 建立最小驱动：只完成 DT 匹配、资源映射和一条 probe 日志。依次故意改错 compatible、关闭 provider、让 probe 返回 `-EPROBE_DEFER`，观察 sysfs 和 dmesg。这个实验能把“设备不存在、驱动未加载、匹配失败、依赖未就绪”四类问题区分开。

## 易错纠偏

- 用户态进入系统调用不等于一定发生进程上下文切换；
- `probe` 成功不代表用户态数据通路正确；
- `volatile` 不替代 `readl/writel`、锁、屏障或 DMA API；
- workqueue 在进程上下文执行，但回调仍可能并发，且取消/flush 有生命周期要求；
- `devm_*` 简化释放，不自动修复停机顺序；
- DTS 不是驱动配置项垃圾场；
- 不要用私有 ioctl 重造已有标准子系统 ABI。

## 交叉链接

- 前置：[C 工具链、ELF 与链接]({{< relref "c-toolchain.md" >}})
- 硬件对照：[MCU 外设、中断与 DMA]({{< relref "mcu.md" >}})
- 并发对照：[RTOS 调度与 ISR 边界]({{< relref "rtos.md" >}})
- 平台专题：[高通 / 联发科 Linux BSP]({{< relref "../company/linux-bsp.md" >}})

## 官方资料

- [Linux：Platform Devices and Drivers](https://docs.kernel.org/driver-api/driver-model/platform.html)
- [Linux and the Devicetree](https://docs.kernel.org/devicetree/usage-model.html)
- [Linux DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html)
- [Linux Workqueue](https://docs.kernel.org/core-api/workqueue.html)
- [Linux Driver Basics](https://docs.kernel.org/driver-api/basics.html)
