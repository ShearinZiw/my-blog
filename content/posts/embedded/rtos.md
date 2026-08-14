+++
date = '2026-08-12T00:00:00+08:00'
title = 'RTOS：调度、同步与实时故障诊断'
categories = ['embedded', 'rtos']
tags = ['RTOS', 'FreeRTOS', '调度', '中断', '实时性', '面试']
summary = '从任务状态、调度和同步原语，到 ISR 边界、优先级反转、栈预算与端到端时延。'
weight = 30
+++

# RTOS：调度、同步与实时故障诊断

适用范围：以抢占式单核 RTOS 和 FreeRTOS 术语为主；其他内核的 API 名称和调度细节可能不同。先确认具体端口、版本与配置，再下实现结论。

## 30 秒总答

> RTOS 的价值不是“同时跑多个函数”，而是在资源受限系统中给并发事件建立可分析的优先级、阻塞与时限。调度器通常选择最高优先级的 Ready 任务；任务等待事件时应进入 Blocked，而不是轮询。ISR 只完成有界工作并通过 `FromISR` 原语唤醒任务。实时性看最坏响应时间，不看平均速度。

## 任务为什么要阻塞而不是轮询

典型状态包括 Running、Ready、Blocked 和 Suspended：

- **Ready**：条件已经满足，只等 CPU；
- **Blocked**：等待队列、通知、信号量或超时；
- **Suspended**：被显式移出调度竞争，时间流逝通常不会自动解除；
- **Running**：当前正在 CPU 上执行。

高优先级任务若不断轮询且从不阻塞，会让低优先级任务饥饿。正确模式是事件到来前阻塞，事件到来后立即成为 Ready。

```c
for (;;) {
    sensor_msg_t msg;
    if (xQueueReceive(sensor_queue, &msg, portMAX_DELAY) == pdPASS) {
        process_sensor(&msg);
    }
}
```

## 抢占、时间片与 Tick 是一回事吗

不是。

- 抢占决定更高优先级任务 Ready 后，是否可以夺取 CPU；
- 时间片决定同优先级 Ready 任务如何轮转；
- Tick 是常见的时间基准和调度触发源，但调度也能由 ISR 唤醒、主动 yield 等事件触发；
- Tickless idle 是在空闲期减少周期 Tick，以降低功耗，不代表取消所有时间管理。

周期任务要避免累计漂移，应基于绝对节拍：

```c
TickType_t next = xTaskGetTickCount();
for (;;) {
    sample_once();
    vTaskDelayUntil(&next, pdMS_TO_TICKS(10));
}
```

## 上下文切换保存什么

上下文切换至少要保存任务继续执行所需的处理器状态，例如栈指针和 ABI 规定的寄存器；FPU、MPU、TrustZone 或架构扩展会改变保存集合。TCB 保存的是内核管理元数据，具体布局属于端口实现，不应背成跨 RTOS 固定结构。

Cortex-M 常用 PendSV 执行任务切换、SysTick 提供节拍、SVC 启动或进入内核服务，但这是常见端口方案，不是所有 MCU/RTOS 的普遍定律。

## 队列、信号量、互斥量、事件组、任务通知怎样选

| 原语 | 传递什么 | 典型用途 | 不适合 |
| --- | --- | --- | --- |
| Queue | 有边界的数据副本/指针 | 多生产者消息、所有权转移 | 只为唤醒单任务时成本偏高 |
| Binary semaphore | 一次事件/令牌 | ISR 完成通知 | 表达资源所有权 |
| Counting semaphore | N 个令牌 | 资源计数、事件累积 | 携带消息内容 |
| Mutex | 所有权 + 互斥，通常带优先级继承 | 任务间保护共享资源 | ISR；跨多个资源的原子事务 |
| Event group | 多个位条件 | 等待任意/全部状态组合 | 排队保存每次事件 |
| Task notification | 面向特定任务的值/位/计数 | 轻量的一对一唤醒 | 多消费者消息队列 |

FreeRTOS 的直接任务通知绑定接收任务，适合 ISR → 单 worker；队列适合携带每个事件的数据。选择依据是语义，而不是只看“哪个快”。

## 中断中能调用哪些 RTOS API

只能调用该 RTOS 明确提供并允许的 ISR-safe API；FreeRTOS 通常以 `FromISR` 后缀区分。它们不会执行可能阻塞的等待，并通过参数报告是否唤醒了更高优先级任务。

```c
void DMA_IRQHandler(void)
{
    BaseType_t should_yield = pdFALSE;

    clear_dma_irq();
    vTaskNotifyGiveFromISR(worker_handle, &should_yield);
    portYIELD_FROM_ISR(should_yield);
}
```

关键边界：在 Cortex-M FreeRTOS 端口中，能调用 RTOS API 的 ISR 优先级必须满足 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 约束。数值越小常代表硬件优先级越高，且实现的优先级位数有限；这是常见配置错误源，不能只凭表面数值比较。

## ISR 应该做多少工作

ISR 的理想职责：确认来源、清/屏蔽中断、搬运最小状态、投递事件、退出。解析协议、日志格式化、等待锁、动态分配和大循环通常延后到任务。

端到端响应时延可拆成：

`硬件触发 → IRQ 进入 → ISR 工作 → 任务成为 Ready → 调度 → 任务处理完成`

只有拆开测，才知道瓶颈是关中断过久、ISR 过长、高优先级任务占用，还是 worker 自身处理慢。

## 什么是优先级反转

高优先级 H 等待低优先级 L 持有的互斥量，同时中优先级 M 抢占 L，使 H 间接被 M 阻塞，这才是经典优先级反转。

优先级继承会临时提升持锁者，缩短阻塞，但不是万能解：

- 必须使用具有所有权语义的 mutex，而非普通二值信号量；
- 嵌套锁和多资源仍需分析；
- 临界区必须有界；
- 实现策略取决于具体内核。

最有效的第一步通常是缩小临界区、统一锁顺序、避免在持锁时做 I/O。

## 死锁、丢事件和队列满怎样区分

- **死锁**：任务形成等待环，相关计数不再增长；
- **丢事件**：生产计数增长，消费计数缺口出现，但系统仍运行；
- **队列满**：发送 API 返回失败或阻塞，队列水位长期触顶；
- **通知合并**：位或二值语义只表示“至少发生一次”，不保存每次事件；
- **优先级饥饿**：某任务长期 Ready 却没有运行时间。

每次发送都检查返回值，并记录生产、消费、丢弃和最大水位；没有计数就很难区分这些现象。

## 栈和堆怎样做预算

任务栈取决于最深调用链、局部变量、库函数、FPU 上下文和中断/异常模型。不要只用“运行没崩”作为依据：

1. 静态检查大局部数组、递归和格式化函数；
2. 压力覆盖最深路径；
3. 读取 high-water mark / canary；
4. 留出可解释余量；
5. 分清任务栈与中断使用的栈。

动态分配并非绝对禁止，但长期运行固件必须回答确定性、碎片、失败处理和观测问题。能静态创建的核心任务、队列和控制块通常优先静态创建；变长业务数据可使用定长块池或受控分配器。

## 软件定时器回调为什么不能堵塞

许多 RTOS 在同一个 timer service task 中串行执行软件定时器回调。某个回调阻塞或做重活，会拖延其他定时器。回调只更新状态或投递工作；业务处理交给专门任务。

## 项目追问

### 为什么这样划分任务

不要回答“一模块一个任务”。给出任务边界的依据：截止期、阻塞源、共享资源、故障隔离和栈成本。过多任务增加栈和切换成本，过少任务会让慢 I/O 阻塞实时路径。

### 优先级怎样定

先按截止期和阻塞依赖列任务表，再测 WCET/执行时间分布、周期和共享资源。不要把“重要”直接等同于“最高优先级”，也不要靠不断调数字消除症状。

### 怎样证明系统满足实时性

至少给出最坏或高分位端到端时延、测试负载、采样方式、丢事件数、栈水位和连续运行时间。平均时延无法证明 deadline 一定满足。

## 偶发超时诊断剧本

1. 明确定义哪两个观测点之间超时，用 GPIO 或 trace 打时间戳；
2. 采集任务状态、运行时间、队列水位、ISR 次数和最长关中断时间；
3. 看超时时 worker 是 Blocked、Ready 但未运行，还是正在执行；
4. 若 Ready 未运行，找所有更高优先级执行流和长临界区；
5. 若 Blocked，检查等待对象、发送返回值和所有权；
6. 若正在执行，拆分算法与外设等待时间；
7. 修复后用最高输入速率、同时中断、内存压力和长稳测试回归。

## 最小实验

创建三个任务重现优先级反转：L 持 mutex，H 随后请求 mutex，M 执行 CPU 密集循环。分别使用 mutex 与普通二值信号量，记录 H 的阻塞时间和 L 的有效优先级。实验应同时验证你的具体 RTOS 是否实现、怎样实现优先级继承，而不是只复述教材。

## 易错纠偏

- `volatile` 不提供任务同步和原子性；
- 关调度器不一定等于关中断，关中断也不等于全系统并发都安全；
- 二值信号量表达事件，mutex 表达所有权，不能只因 API 类似就互换；
- 高优先级不是“运行更多”，而是 Ready 时更先获得 CPU；
- `vTaskDelay()` 不是精密绝对周期源；
- 看门狗复位是最后防线，不是并发设计的修复方案。

## 交叉链接

- 前置：[MCU 的异常与 NVIC]({{< relref "mcu.md" >}})
- 实现落点：[C 工具链、ABI 与启动]({{< relref "c-toolchain.md" >}})
- 对照：[Linux 中断、锁与 workqueue]({{< relref "linux-driver.md" >}})
- 网络线程边界：[lwIP 的 tcpip_thread 与 API]({{< relref "lwip.md" >}})

## 官方资料

- [FreeRTOS Kernel 文档](https://www.freertos.org/Documentation/02-Kernel/01-About-the-FreeRTOS-kernel)
- [FreeRTOS 直接任务通知](https://www.freertos.org/Documentation/02-Kernel/02-Kernel-features/03-Direct-to-task-notifications/01-Task-notifications)
- [Arm CMSIS-Core：NVIC](https://arm-software.github.io/CMSIS_6/latest/Core/group__NVIC__gr.html)
