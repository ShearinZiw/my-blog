+++
date = 2026-08-12
title = "MCU 工程主干：Cortex-M 启动、外设与故障闭环"
categories = ["embedded", "interview"]
tags = ["MCU", "Cortex-M", "CMSIS", "NVIC", "DMA", "HardFault"]
summary = "围绕 Cortex-M 启动、异常、时钟、常用外设、DMA、低功耗和 HardFault 建立可追问的工程知识链。"
weight = 20
+++

## 定位 / 学习目标

MCU 面试的分水岭不是背出寄存器名，而是能把“外设不工作”还原为时钟、引脚、电气、状态机、并发与内存可见性问题。
本文以 Cortex-M 为主线；具体地址、位定义和时序必须回到所用芯片的 datasheet、reference manual 与 errata。

学完应能做到：

- 讲清复位到 `main`、向量表、NVIC 与异常栈帧；
- 正确配置时钟树，并理解 Flash wait state、电压域和外设时钟；
- 从 GPIO/定时器/UART/SPI/I2C 推导驱动状态机；
- 说明 DMA 的所有权、cache、一致性和完成边界；
- 用故障寄存器、栈帧、map/反汇编定位 HardFault；
- 设计可测量的低功耗与看门狗策略。

## 知识链路

```text
上电/复位
  → 取向量表[0]为 MSP、[1]为 Reset_Handler
  → startup 搬运 .data、清零 .bss、初始化系统
  → 时钟树/Flash/电压域稳定
  → GPIO 复用与外设状态机
  → 中断或 DMA 搬运数据
  → RTOS/主循环消费事件
  → fault、trace、功耗与时序证据闭环
```

每条链都要回答三个问题：硬件当前状态是什么、软件拥有什么、用什么观测证明判断。

## 核心面试题（结论 → 原理 → 工程边界）

### 1. Cortex-M 复位后如何走到 `main`？

**结论：** 内核从向量表读取初始 MSP 和复位向量，进入 Reset_Handler；startup 建立 C 运行环境后调用 `main`。

**原理：** 向量表首项不是函数，而是栈顶；异常入口地址最低位表示 Thumb 状态。Reset_Handler 通常复制 `.data`、清零 `.bss`，再做 `SystemInit` 和运行库初始化。

**工程边界：** 向量表基址由芯片启动映射决定；支持 VTOR 的内核可重定位，部分 Cortex-M0 实现没有 VTOR。Bootloader 跳 App 不能机械照搬某一芯片步骤。

### 2. 向量表重定位需要注意什么？

**结论：** 新表必须满足内核要求的地址对齐，表项有效，并在放开中断前完成 VTOR 与栈/状态切换。

**原理：** 异常到来时 NVIC 按异常号索引表项；基址错误或表项 Thumb 位无效会形成 fault。`DSB/ISB` 用于确保系统控制更新在后续取指/异常前生效。

**工程边界：** 对齐值与已实现中断数量/内核版本有关，应读对应 Generic User Guide；带 cache 的器件还需保证新表内容已对取指侧可见。

### 3. NVIC 优先级数字越大越优先吗？

**结论：** Cortex-M 中数值越小优先级越高；只有芯片实现的高若干位有效。

**原理：** priority grouping 将已实现位划分为抢占优先级和子优先级。抢占优先级决定能否打断，子优先级只在同时 pending 时排序。

**工程边界：** CMSIS `NVIC_SetPriority` 接受未左移的逻辑值并负责移位；直接写寄存器时规则不同。不要把 FreeRTOS 的库优先级宏与裸寄存器编码混用。

### 4. PRIMASK、BASEPRI、FAULTMASK 有何边界？

**结论：** PRIMASK 屏蔽可配置优先级异常，BASEPRI 屏蔽数值不高于某阈值的一组中断，FAULTMASK 更强；NMI 不会被这些普通机制屏蔽。

**原理：** BASEPRI 可保留最高紧急中断，适合有上限的临界区；保存并恢复旧值才能正确嵌套。屏蔽只延迟响应，pending 状态可能仍累积。

**工程边界：** Cortex-M0/M0+ 等基线内核通常没有 BASEPRI。临界区不能包围不可控阻塞，也不能误认为关中断会停止 DMA 或外设状态机。

### 5. 异常进入时硬件自动保存哪些内容？

**结论：** 基础栈帧包含 R0-R3、R12、LR、PC、xPSR；R4-R11 通常由软件按需保存。

**原理：** Handler 模式使用 MSP；异常来自 Thread 模式时原先可能使用 MSP 或 PSP。LR 中的 EXC_RETURN 编码返回模式、所用栈和是否存在扩展浮点帧。

**工程边界：** STKALIGN 可能插入对齐字；启用 FPU 后惰性压栈会改变帧形态和最坏中断延迟。HardFault 解析器必须先根据 EXC_RETURN 选 MSP/PSP，再判断扩展帧。

### 6. 中断函数应该多短？

**结论：** ISR 只做确认事件、获取必要数据和通知下半部，长度由系统允许的最坏响应延迟预算决定，而非固定行数。

**原理：** 长 ISR 增大同级/低级中断延迟；频繁抢占还增加栈峰值。Cortex-M 的 tail-chaining 能减少连续异常的进出开销，但不能消除业务执行时间。

**工程边界：** 清标志顺序取决于外设语义；有些读状态再读数据才清除，有些 W1C。照抄“先清中断”可能丢事件，必须按 reference manual。

### 7. 时钟树配置为什么常在切换主频时死机？

**结论：** 安全顺序通常是满足电压/Flash 延迟要求，启动并等待新时钟源稳定，配置分频与 PLL，再切换并确认状态。

**原理：** CPU 频率超过当前 Flash 等待周期或电压能力会取指失败；PLL 有输入范围、倍频范围和锁定时间。外设时钟还可能来自独立 mux。

**工程边界：** 顺序和上限完全依赖芯片。降频时的 Flash/电压调整顺序可能反向；始终检查 clock security、errata 和超时回退，不要无限等待 ready 位。

### 8. 如何确认“主频真的对了”？

**结论：** 不只相信配置变量；读回时钟状态，并用 MCO、定时器翻转 GPIO、DWT cycle counter 或示波器测量。

**原理：** SysTick、UART baud、timer kernel clock 都由实际时钟链决定，分频假设错会产生系统性比例误差。

**工程边界：** 某些 APB prescaler 不为 1 时 timer clock 会倍频，但这是芯片级规则，不是所有 MCU 的通则。

### 9. GPIO 输出为何推荐原子 set/reset 寄存器？

**结论：** 多上下文修改同一输出口时，使用 BSRR/SET/CLR 一类写寄存器可避免输出数据寄存器的读—改—写竞争。

**原理：** `ODR |= bit` 会先读整个端口再写回，ISR 若夹在中间修改其他位会被覆盖。原子写寄存器由硬件按位生效。

**工程边界：** 寄存器名称和写入语义因厂商而异。输入悬空、上下拉、驱动能力、slew rate 与复用冲突是电气问题，软件原子性解决不了。

### 10. 定时器频率如何推导？

**结论：** 对常见向上计数器，更新频率可写为 `f_timer / ((PSC+1)*(ARR+1))`，但先确定 timer kernel clock。

**原理：** prescaler 先分频，计数器从 0 计到 ARR 形成 ARR+1 个 tick。PWM 占空由比较值与计数模式决定。

**工程边界：** center-aligned、重复计数器、预装载与主从同步会改变更新节奏；修改 ARR/CCR 是否立即生效取决于 shadow/preload 配置。

### 11. UART 收发最常见的工程故障是什么？

**结论：** 首查双方帧格式和 baud 误差，再查 RX 过载、错误标志清除顺序与缓冲所有权。

**原理：** 异步通信靠采样时钟容忍有限频差；CPU/ISR 来不及读数据会 overrun。DMA 环形接收常结合 IDLE 或定时快照确定新增区间。

**工程边界：** IDLE 清除方式、FIFO 深度、DMA remaining counter 的一致快照依芯片而定。日志 UART 若在 ISR 中阻塞，会反过来改变时序甚至死锁。

### 12. SPI 的“全双工”意味着什么？

**结论：** 每发一个时钟双方都会移入移出一位；主机想读也必须发送 dummy，想写也必须处理接收侧数据。

**原理：** CPOL/CPHA 定义空闲电平和采样边沿；片选界定从机事务。模式或首位顺序错误常表现为整体移位或固定错位。

**工程边界：** CS 前后延时、连续帧要求、最高 SCK 与信号完整性来自从机手册。DMA 完成只表示内存传输完成，外设 busy 清零后总线最后一位才真正发完。

### 13. I2C 为什么必须开漏和上拉？

**结论：** SDA/SCL 由设备只主动拉低、由电阻拉高，从而支持 ACK、时钟拉伸和多主仲裁而不产生推挽冲突。

**原理：** 高电平上升是 RC 过程，上拉过弱会在高速下达不到采样阈值；过强会增加低电平灌电流。仲裁通过“发送高却读到低”检测丢失。

**工程边界：** 总线卡低先区分从机状态、供电/电平转换与控制器状态机；常见恢复是按规范/器件要求脉冲 SCL 后产生 STOP，但不是所有器件都能被软件解锁。

### 14. DMA 解决了什么，又没解决什么？

**结论：** DMA 减少 CPU 搬运和中断频率，但没有自动解决生命周期、并发所有权、cache 一致性和外设完成语义。

**原理：** CPU 配置源/目的/长度，DMA 作为 bus master 传输；完成/半完成中断只是状态事件。循环 DMA 的生产位置可由计数器快照推导。

**工程边界：** Cortex-M7 等带 D-cache 平台，TX 前需 clean、RX 后需 invalidate，并按 cache line 对齐避免邻接数据被误伤；buffer 所在 RAM 还必须可被该 DMA master 访问。

### 15. 如何正确写 MMIO 寄存器访问？

**结论：** 使用 CMSIS 厂商头文件中的 `volatile` 寄存器结构和掩码，按位语义选择赋值、RMW 或专用 set/clear 操作。

**原理：** `volatile` 保留真实访问；`uint32_t` 明确宽度；`static inline` 封装可保留类型检查且无额外调用。reserved 位通常应保持复位值。

**工程边界：** 对 W1C 位做 `reg |= mask` 可能把其他已置位状态一并清掉；读有副作用、write-only、不同访问宽度、posted write 都必须按手册处理。

```c
static inline void gpio_set(GPIO_TypeDef *gpio, uint32_t pin_mask)
{
    gpio->BSRR = pin_mask;   /* 单次写，避免 ODR 的读改写竞争 */
}
```

### 16. HardFault 应怎样从“死机”变成可定位地址？

**结论：** 保存 SCB 故障寄存器、EXC_RETURN、MSP/PSP 与正确栈帧中的 PC/LR，再用同版 ELF 做符号化和反汇编。

**原理：** `CFSR` 汇总 MemManage/BusFault/UsageFault 子状态，`HFSR.FORCED` 常表示可配置 fault 上卷；`BFAR/MMFAR` 只有对应 valid 位为 1 才可信。

**工程边界：** 异步/不精确总线 fault 的 stacked PC 可能只是后续指令；栈损坏也会让帧不可读。生产设备应把最小 crash record 写入保留 RAM/掉电安全存储，并带固件版本。

### 17. 低功耗不是执行一条 WFI 就结束吗？

**结论：** 低功耗是一套状态契约：选择 sleep/stop/standby，关闭泄漏源，配置唤醒源，进入前后恢复时钟和外设上下文。

**原理：** `WFI` 只是等待中断指令，实际深度由电源控制器和 `SLEEPDEEP` 等配置决定。GPIO 浮空、调试口、稳压器、RAM retention 都影响静态电流。

**工程边界：** 电流表平均值会掩盖唤醒尖峰；应同时测睡眠电流、驻留比例、唤醒延迟和每周期能量。调试连接经常阻止进入最深模式。

### 18. 看门狗如何设计才不会“掩盖故障”？

**结论：** 只由健康监督点喂狗；各关键任务提交进度，监督者确认系统整体满足时序与数据条件后再喂硬件狗。

**原理：** 若任意任务或周期中断直接喂狗，主业务死锁仍可能持续喂。窗口看门狗还能检测喂得过早，独立时钟看门狗能覆盖主时钟失效。

**工程边界：** 复位前来不及做复杂日志；故障上下文应提前维护，启动后读取 reset cause。调试暂停策略与量产策略需分开配置。

## 项目追问

### 做一个稳定的 UART DMA 环形接收，你怎样划分所有权？

DMA 是 producer，协议线程是 consumer；用单调递增的软件读指针与 DMA 写位置形成区间。
IDLE/半满/满事件只负责唤醒，不能假设一次事件对应一帧；解析器处理回绕、粘包和超长帧。
更新索引前后使用适合平台的临界区/内存屏障，带 cache 平台先完成对应区域维护。

### PWM 更新偶发毛刺，你会追问哪些配置？

- ARR/CCR 是否启用 preload，更新事件何时把 shadow 值装入；
- 多通道写入是否需要同一个同步更新点；
- DMA burst 的宽度、顺序和触发源是否匹配；
- GPIO slew、死区、互补输出与 break 输入是否造成硬件侧变化；
- 用逻辑分析仪关联“寄存器写时刻”和“输出边沿”，不要只打印日志。

### 产品待机电流比目标高十倍，如何拆解？

先做电源轨分区与模式基线，再逐项关闭外设时钟、外部器件、GPIO 上拉和调试接口。
读取唤醒/复位原因，确认没有高频假唤醒；示波器配合电流探头区分静态泄漏与周期尖峰。
最终把睡眠电流、唤醒耗时、工作能量和占空比纳入自动回归，而非只留一次手工截图。

## 故障诊断剧本

| 现象 | 第一证据 | 常见根因 | 闭环验证 |
|---|---|---|---|
| 上电不进 `main` | PC/MSP、复位向量、reset cause | 向量基址、栈顶、startup 或供电复位 | 单步 Reset_Handler 并核对 ELF |
| 中断从不进入 | 外设 flag、NVIC pending/enable | 时钟、触发源、清 flag、屏蔽状态 | 软件置 pending 与真实事件分层验证 |
| 中断风暴 | active/pending 与状态寄存器 | 源未清、清除顺序错、电平条件仍在 | 临时屏蔽源并观测状态变化 |
| UART 偶发丢字节 | ORE/FIFO/DMA 指针 | ISR 延迟、缓冲小、索引竞争 | 压测 + 水位/错误计数器 |
| I2C BUSY 不退 | SDA/SCL 波形与电平 | 从机卡低、漏 STOP、上拉/复用错误 | 总线恢复并重置控制器状态机 |
| DMA 数据旧或撕裂 | 地址、长度、cache line | 所有权、cache、不可达 RAM | 禁 cache 对照并加维护/对齐 |
| HardFault | CFSR/HFSR、stacked PC | 非法地址、未对齐、坏函数指针、栈溢出 | `addr2line` + 反汇编 + 最小复现 |
| 低功耗电流高 | 电流时序与唤醒原因 | GPIO、调试、外设未停、频繁唤醒 | 分模块关断并量化差值 |

HardFault 最小处理框架的重点不是“自动修复”，而是在二次故障前保存证据：

```c
typedef struct {
    uint32_t exc_return, cfsr, hfsr, mmfar, bfar;
    uint32_t r0, r1, r2, r3, r12, lr, pc, xpsr;
} crash_record_t;
```

如果 handler 自己使用损坏的栈、调用格式化日志或写不可靠 Flash，原始故障会被覆盖；实现应尽量短，并预留 CRC 与固件标识。

## 可验证实验 / 命令

### 1. 启动与向量表

```bash
arm-none-eabi-readelf -S -s firmware.elf | less
arm-none-eabi-objdump -s -j .isr_vector firmware.elf
arm-none-eabi-objdump -d -S firmware.elf > firmware.dis
arm-none-eabi-nm -n firmware.elf | grep -E 'Reset_Handler|_sdata|_sbss'
```

烧录后在 Reset_Handler 首条指令断点，核对 MSP 是否落在有效 RAM、PC 是否落在可执行 Flash，再观察 `.data/.bss` 前后内容。

### 2. 中断延迟与占用

在 ISR 入口/出口用独立 GPIO set/reset，逻辑分析仪测到达延迟、执行宽度和抖动；避免 RMW 翻转引入竞争。
再逐级增加高优先级负载，得到最坏情况，而不是只测空闲平均值。

### 3. 时钟与串口误差

把选定内部时钟输出到 MCO 或用 timer 输出固定分频，实测频率；同时抓取 UART 0x55 波形，计算 bit time 与误差。
改变温度或供电条件后重复，验证内部 RC 是否仍满足链路预算。

### 4. DMA 一致性

准备跨 cache-line 的已知图样，分别在执行/不执行 clean、invalidate 时做 TX/RX 回环；记录地址对齐、长度与 cache 配置。
该实验要在真实带 cache 的目标上执行，主机模拟无法证明总线一致性。

### 5. 主动制造可控 Fault

在 Debug 专用固件中分别触发未定义指令、非法地址访问和除零（先确认 trap 配置），验证 crash record 能还原到确切源码行。
实验必须隔离量产配置，并确认不会写坏持久化数据或驱动危险执行器。

常用定位命令：

```bash
arm-none-eabi-addr2line -e firmware.elf -a -f -C 0x08001234
arm-none-eabi-objdump -d -S --start-address=0x08001200 \
  --stop-address=0x08001280 firmware.elf
arm-none-eabi-gdb firmware.elf
# (gdb) info registers
# (gdb) x/8wx $sp
# (gdb) monitor reset halt
```

## 易错纠偏

- **误区：** 优先级数字越大越紧急。**纠偏：** NVIC 数值越小越高，且只实现部分高位。
- **误区：** 关全局中断后外设停止。**纠偏：** DMA、timer 和总线仍运行，只是 CPU 延迟响应。
- **误区：** ISR 里清标志总是写 0。**纠偏：** 常见有 W1C、读序列清除等不同语义。
- **误区：** DMA complete 等于 SPI 线空闲。**纠偏：** 还需等待外设移位器/BSY 完成再释放 CS。
- **误区：** `volatile` buffer 就能解决 DMA cache。**纠偏：** 它不 clean/invalidate cache，也不建立硬件所有权。
- **误区：** I2C 上拉越小越好。**纠偏：** 要同时满足上升时间和器件灌电流规格。
- **误区：** HardFault 地址寄存器永远有效。**纠偏：** 先检查 MMARVALID/BFARVALID，区分精确与不精确错误。
- **误区：** WFI 必然进入最低功耗。**纠偏：** 深度由电源配置决定，pending 中断还可能令其立即返回。
- **误区：** 独立任务各自喂狗更安全。**纠偏：** 这会掩盖局部或主流程失活，应集中做健康仲裁。

## 交叉链接

- [嵌入式 C 工具链：从源码到可启动镜像](/posts/embedded/c-toolchain/)
- [RTOS：调度、同步与实时性证据](/posts/embedded/rtos/)
- [Linux 与驱动：从中断到设备模型](/posts/embedded/linux-driver/)
- [lwIP：协议栈线程模型与网卡收发](/posts/embedded/lwip/)

## 官方资料

- [CMSIS-Core documentation](https://arm-software.github.io/CMSIS_6/latest/Core/index.html)
- [CMSIS-Core register and intrinsic reference](https://arm-software.github.io/CMSIS_6/latest/Core/group__Core__Register__gr.html)
- [Arm Cortex-M generic user guides](https://developer.arm.com/documentation/dui0552/latest/)
- [Arm ABI specifications](https://github.com/ARM-software/abi-aa)
- [GNU Binutils Documentation](https://sourceware.org/binutils/docs/)

最后的工程准则是：CMSIS 给出内核与访问接口，芯片手册给出实例化细节，errata 给出已知例外；三者都不能由博客或 SDK 示例替代。
