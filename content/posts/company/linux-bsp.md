+++
date = '2026-08-12'
title = 'Qualcomm / MediaTek Linux BSP：从 Yocto 到板级 Bring-up'
categories = ['company', 'embedded-linux']
tags = ['Linux BSP', 'Yocto', 'Device Tree', 'DMA', 'Power Management', 'Qualcomm Linux', 'MediaTek Genio']
summary = '用公开资料梳理通用 Yocto、启动链、DT、probe、DMA、PM 与 bring-up，并明确 Qualcomm Linux 和 MediaTek Genio 的产品及 NDA 边界。'
weight = 20
+++

# Qualcomm / MediaTek Linux BSP：从 Yocto 到板级 Bring-up

## 适用范围与边界
本文先讲可迁移的 Linux BSP 方法，再映射到两个公开生态：Qualcomm Linux 的 upstream-first / `meta-qcom` 路线，以及 MediaTek Genio 的 IoT Yocto 与 Ubuntu 路线。
平台支持矩阵、内核/Yocto 版本、启动固件和闭源组件随发布变化，必须以目标板和所选 release 的 manifest、release notes、许可证为准。
Android vendor BSP、手机平台经验、旧版厂商交付不能自动套到当前公开 IoT Linux 产品。
本文不猜测未公开寄存器、固件内部实现、私有驱动、客户设计或 NDA 文档。

## 30 秒结论
> BSP 的交付目标不是“内核能启动”，而是让一块板的启动、硬件描述、驱动、根文件系统、升级和电源管理可复现、可诊断、可维护。Yocto 用 layer/recipe/machine/distro/image 固化构建；启动问题按 BootROM 到 userspace 的阶段找最后证据；驱动问题沿 DT match、资源获取、probe/defer、DMA/IOMMU 和 PM 依赖排查。Qualcomm Linux 要区分公开上游内核与 `meta-qcom` 发布，MediaTek 要区分 Genio IoT Yocto 与 Ubuntu 产品线；遇到闭源边界只记录可公开接口、版本、许可证、hash 和健康检查，不臆测内部机制。

## 核心机制问答
### 一个可交付 BSP 包含什么
最小集合通常包括：
- 可复现的源码/metadata manifest、工具链、容器或主机要求；
- machine、distro、image 配置和产品自有 layer；
- bootloader/固件、kernel、DTB/overlay、rootfs 与分区/烧录描述；
- 板级驱动、校准/配置、许可证清单和源代码合规材料；
- 串口救援、升级/回滚、工厂烧录和密钥注入边界；
- 启动、外设、压力、功耗、恢复与量产测试报告。

“能编译”和“能重复生产”之间，差的是版本锁定、产物追踪、自动测试、更新策略和现场恢复。

### Yocto 的 layer、recipe、machine、distro、image 怎样分工
| 对象 | 负责什么 | 不应滥用 |
| --- | --- | --- |
| layer | 按职责组织 metadata，并声明依赖/兼容系列 | 把所有产品修改堆进厂商 layer |
| recipe / `.bbappend` | 获取、配置、编译、安装一个组件或追加定制 | 无边界覆盖整棵源码 |
| machine | SoC/板硬件、内核 provider、DT、启动产物 | 放产品业务策略 |
| distro | init、libc、包格式、安全与全局发行策略 | 绑定单块板的 GPIO |
| image | 选择最终 rootfs 功能和包集合 | 用手工 rootfs 改动替代 recipe |

产品定制放独立 `meta-product`，并用明确 layer priority/override；`local.conf` 只做开发者本地选择，不作为唯一产品规格。
常用证据包括 `bitbake-layers show-layers/show-appends`、`bitbake -e <recipe>`、task 的 `log.do_*` 与 `run.do_*`、`oe-pkgdata-util` 和最终 license manifest。
修补外部源码优先用 `devtool modify/build/finish` 形成可审查 patch，而不是直接改 `tmp/work`。

### 怎样保证构建可复现
锁定所有 layer revision、`SRCREV`、下载镜像、容器/宿主依赖和配置输入；对发布产物记录 manifest、配置摘要、源码许可证、SBOM 与 hash。
sstate 提高速度但不是发布依据；遇到“只在某台机器能编”时，先比较签名和环境，再有针对性地 clean 对应 recipe，不能把清空整个缓存当修复。
CI 至少做 clean 构建抽检、离线源镜像验证、许可证门禁和产物启动测试。

### 启动链怎样分层
通用视图是：
`BootROM → 厂商/平台早期固件 → 第一/第二阶段 bootloader 或 UEFI → kernel + DTB + initramfs → rootfs → init/systemd → 产品服务`
每个平台的阶段名称、信任链、镜像容器和分区不同，不应背一条“通用 Qualcomm/MediaTek 启动顺序”。
排障先问最后一个可见输出来自哪一阶段，再核对该阶段消费的镜像、地址/分区、签名、版本与下一跳参数。
为每次烧录保存命令、串口日志、分区表和镜像 hash；否则“同一镜像”只是未经证明的假设。

### Device Tree 应描述什么
DT 描述不可探测硬件及其连接关系，不应承载驱动算法或产品业务逻辑。
节点常见属性包括 `compatible`、`reg`、`interrupts`、`clocks`、`resets`、`pinctrl-*`、`dmas`、`iommus`、`power-domains`、`phys`、`supplies` 与 `status`。

新增 binding 先写/复用 YAML schema，再运行 `make dt_binding_check` 与 `make dtbs_check`。
不要因 probe 不成功就随意复制参考板节点；先对照原理图、电源域、地址、IRQ 极性、时钟和 pinmux。
overlay 适合明确的可插拔/部署变体，但 base DT、bootloader fixup 和 overlay 的最终合并结果必须可导出、可验证。

### 驱动为什么没有 probe
按顺序核对：
1. 节点是否在实际加载的 DTB 中且 `status = "okay"`；
2. `compatible` 与驱动 match table 是否一致；
3. 驱动是 built-in 还是 module，module 是否加载及签名通过；
4. bus/controller 父节点是否已注册；
5. regulator、clock、reset、GPIO、IOMMU、DMA 等 provider 是否 ready；
6. probe 是否返回 `-EPROBE_DEFER`、其他错误或成功后在后续阶段解绑。

`-EPROBE_DEFER` 是依赖尚未就绪，不是“忽略错误继续”。
结合 boot log、`/sys/firmware/devicetree/base`、`/sys/bus/*/devices`、`/sys/kernel/debug/devices_deferred`、module 信息与 dynamic debug 建证据链。

### DMA 问题为什么经常表现成随机数据损坏
驱动必须使用 DMA API 建立 CPU 地址到设备可访问地址的映射，不能用 `virt_to_phys()` 猜总线地址。
先设置并检查 DMA mask；区分 coherent allocation 与 streaming mapping；严格配对 map/unmap 或 sync，并保证方向正确。
scatter-gather 映射后的段数可能变化，驱动要使用 DMA API 返回的结果。

IOMMU fault、CMA/连续内存不足、cache 同步遗漏、buffer 提前释放、描述符越界和设备写超时都可能只在压力下暴露。
日志要记录 DMA address/length/direction、描述符代次和完成 IRQ，不打印敏感内容，也避免无界洪泛。
reserved-memory 是明确的系统内存契约，不是所有 DMA 错误的万能修复。

### runtime PM 与系统 suspend 怎样配合
runtime PM 管设备空闲期的局部开关，system suspend 管整机状态迁移；驱动同时受 clock、regulator、reset、power domain、IOMMU 和父子依赖约束。
常见顺序是停止新请求、等待/取消 DMA、保存必要状态、关闭 IRQ/clock/power，恢复时逆向重建；具体 callback 和阶段以子系统规范为准。

遗漏 `pm_runtime_get/put` 配对会导致永不休眠或访问已断电寄存器。
把 wakeup IRQ 配置、`wakeup_source`、autosuspend delay、失败设备与 resume latency 纳入测试。
先用内核 PM 测试级别缩小失败阶段，再看 trace，不要一上来给所有设备加禁止休眠。

### 板级 Bring-up 的推荐闸门
| 闸门 | 目标 | 主要证据 |
| --- | --- | --- |
| 0 电气安全 | 电源、复位、时钟、启动脚符合设计 | 电流限幅、示波器/万用表、原理图复核 |
| 1 最早串口 | 找到稳定早期输出和复位原因 | 带时间戳串口、reset cause |
| 2 启动介质 | 稳定读取正确 boot artifacts | 分区表、hash、启动日志 |
| 3 kernel/DT | 内核启动且硬件描述与板匹配 | cmdline、DTB hash、dtbs_check |
| 4 基础外设 | console、storage、USB/network 可回归 | probe/IRQ/DMA 统计与功能测试 |
| 5 性能与 PM | 温度、电源、压力和 suspend 稳定 | trace、功耗曲线、长稳报告 |
| 6 更新恢复 | 升级失败仍可进入已知状态 | 断电注入、回滚和救援流程 |

每过一闸门固化已知好产物和测试，避免同时改电源、DT、驱动与 rootfs 后失去因果。

## 两条厂商公开路线
### Qualcomm Linux：upstream-first 与 `meta-qcom`
公开主线内核中，Qualcomm SoC/板级支持位于标准 arm64 DTS、clock/reset/interconnect/remoteproc 等相应子系统；上游邮件列表、kernel docs 和 git history 是机制依据。
Qualcomm Linux 公开提供 Yocto 兼容的 `meta-qcom` BSP layer 及关联仓库；具体 machine、发行分支、kernel provider 和 capability overlay 必须查选定 release。

“upstream-first”应落实为：优先复用上游 binding/子系统接口，维护可审查 patch，持续跟踪上游差距，避免无期限私有 fork。
它不表示目标板所有功能都已在主线可用，也不表示 firmware、模型、校准数据或某些增值组件具有同一许可证和获取方式。
不要把 CodeLinaro/旧 QTI BSP、Android vendor kernel 与当前 `qualcomm-linux/meta-qcom` 混成一个版本体系。

### MediaTek Genio：IoT Yocto 与 Ubuntu
MediaTek 公共 Genio 资源将 IoT Yocto 和 Ubuntu 作为两条可见软件路线，且支持的 SoC/EVK 随 release 更新。
IoT Yocto 通过公开开发指南、release 文档和 Genio GitLab metadata 组织镜像、BSP 与工具；开始项目前先锁定目标板对应 release 与 manifest。
Ubuntu on Genio 是与 Canonical 生态结合的产品路线，包管理、更新、认证/支持周期和镜像组成不能按 Yocto 自建发行版假设。

选型时比较可定制深度、启动/存储预算、包更新模型、安全维护、认证、量产工具和目标硬件支持，而不是笼统问“Ubuntu 还是 Yocto 更好”。
上游内核的 MediaTek 驱动与 DTS 可作为通用机制参考，但不自动证明某一 Genio 发布镜像启用了相同功能。

### NDA 与闭源组件怎么处理
公开知识库可以记录：公开 API/ABI、设备树 binding、版本、许可证、来源 URL、二进制 hash、配置输入、加载结果、超时和健康检查。
不得收录：内部寄存器表、未公开源码/patch、客户原理图、私有日志、签名密钥、固件解密内容或受限文档截图。
遇到二进制组件，只描述可观测输入输出和故障隔离边界；“从现象推测内部算法”不能当事实。
简历和面试也用脱敏后的指标与方法，不用泄露料号组合、客户数据或内部路径证明能力。

## 项目追问
- 你的 manifest 怎样保证半年后能重建同一镜像；哪些输入仍来自外部服务？
- BootROM 到 systemd 的每个产物由谁生成、谁验证、失败后停在哪里？
- 为什么这个硬件信息放 DT 而不是驱动；binding 和原理图如何互证？
- 一次 `-EPROBE_DEFER` 最终依赖哪个 provider，怎样证明顺序已经闭环？
- DMA buffer 的 CPU/device 所有权、IOMMU 映射和 cache 同步时间线是什么？
- suspend 功耗超标时，哪些 rail/clock/wakeup source 提供了反证？
- 厂商 layer 与产品 layer 的 patch 如何升级、回归并逐步上游化？
- 闭源组件出错时，团队能观测和控制的最小边界是什么？

## 诊断剧本
### 板子没有进入 kernel
1. 找最后一条可信串口输出，给它归属 BootROM、固件还是 bootloader。
2. 核对实际分区/加载地址、镜像 header、签名状态与 hash，不只核对文件名。
3. 检查 bootargs、kernel/DTB 选择和内存地址是否冲突。
4. 换回上一份已知好产物时只改变一个阶段，定位首个坏输入。

### DT 节点存在但驱动不工作
1. 从运行中系统导出实际 DT，确认不是改错源文件或加载错 DTB。
2. 检查 match、module、父 bus 与 deferred devices。
3. 逐项验证 regulator/clock/reset/pinctrl/IRQ/DMA/IOMMU 返回值。
4. 用 dynamic debug、ftrace 和错误路径日志定位 probe 的最后一步。

### DMA 压测偶发花屏/坏包
1. 禁止并发复用 buffer，给描述符和提交/完成加代次。
2. 审核 mask、direction、map/sync/unmap 及错误返回，捕获 IOMMU fault。
3. 对比 coherent 测试缓冲与 streaming 路径，缩小 cache/生命周期问题。
4. 改变负载、温度和内存压力，验证不是用延时碰巧掩盖竞态。

### suspend 后设备无法恢复
1. 记录 suspend 阶段、最后失败设备、wakeup sources 和完整 dmesg。
2. 用 `pm_test`/trace 缩小到 freezer、devices、platform、processors 或 core 阶段。
3. 检查在途 DMA/IRQ、runtime PM 引用、父子电源域和 resume 顺序。
4. 恢复成功后做多轮循环、唤醒源矩阵和低电压/高温回归。

### Yocto 构建结果漂移
1. 比较 manifest、layer branch/revision、`bitbake -e` 关键变量和 task hash。
2. 检查浮动 `SRCREV`、未镜像下载、本地文件与 host contamination。
3. 只清理受影响 recipe 并重建，保留失败 task 日志用于根因分析。
4. 把修复落到 layer/lockfile/CI，不能以“删 tmp 后好了”结案。

## 可验证实验
### 实验一：从零复现镜像
在干净容器或受控主机按 manifest 构建两次，比较关键产物 hash、包清单、许可证与启动测试；对允许变化的时间戳单独解释。

### 实验二：DT/probe 故障注入
在测试分支分别制造错误 `compatible`、缺失 clock 与 deferred provider；要求脚本能从实际 DT、deferred list 和日志准确判断三类故障。

### 实验三：DMA/IOMMU 所有权
为每个 buffer 记录 `CPU → mapped → device → completed → CPU` 状态，叠加 I/O 与内存压力；验收为无非法迁移、IOMMU fault、数据校验错和泄漏。

### 实验四：PM 循环
做不少于数百轮 runtime suspend/resume 与 system suspend/wakeup，覆盖网络、存储和指定 GPIO 唤醒；记录失败轮次、功耗分位数和恢复时延。

### 实验五：升级与掉电

在写 bootloader 配置、kernel、rootfs 和首次启动迁移等边界注入掉电；证明设备进入旧版、新版或受控救援，不落入不可观测砖态。

## 易错纠偏

- 错：“BSP 就是驱动集合。” 对：它还包含可复现构建、启动/固件、DT、rootfs、升级、许可证和测试。
- 错：“设备树节点写了就会 probe。” 对：还要匹配实际 DTB、driver、父 bus 和所有 provider 依赖。
- 错：“DMA 地址等于物理地址。” 对：必须走 DMA API，IOMMU 和总线窗口会改变设备视角。
- 错：“`-EPROBE_DEFER` 多等一会就行。” 对：要找到未满足依赖，循环 defer 也可能是 DT/驱动错误。
- 错：“禁用 runtime PM 能修复恢复问题。” 对：它只能做隔离实验，最终要修引用、顺序与硬件状态。
- 错：“Qualcomm/MediaTek BSP 都是一个固定栈。” 对：先锁定产品线、板、release、manifest 和许可证。
- 错：“公开仓库存在就代表所有加速器都主线上游且全开源。” 对：逐组件核对代码、firmware、许可和目标支持。

## 交叉链接

- [C 工具链：从源码到可启动固件]({{< relref "../embedded/c-toolchain.md" >}})：ELF、链接、ABI 和调试产物。
- [Linux / 驱动主干]({{< relref "../embedded/linux-driver.md" >}})：设备模型、并发、IRQ 和用户态接口。
- [MCU：启动、中断、DMA 与低功耗]({{< relref "../embedded/mcu.md" >}})：与板级电源、DMA、启动证据互证。
- [公司专题路线]({{< relref "_index.md" >}})：公司线的范围和公开资料原则。
- [嵌入式岗位知识地图]({{< relref "../interview/embedded-roadmap.md" >}})：将 bring-up 结果整理为可追问证据。

## 官方资料

- [Yocto Project Overview and Concepts Manual](https://docs.yoctoproject.org/overview-manual/index.html)
- [Yocto `devtool` Quick Reference](https://docs.yoctoproject.org/ref-manual/devtool-reference.html)
- [Yocto BSP Developer's Guide](https://docs.yoctoproject.org/bsp-guide/index.html)
- [Linux Devicetree Usage Model](https://docs.kernel.org/devicetree/usage-model.html)
- [Linux DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html)
- [Linux Device Power Management Basics](https://docs.kernel.org/driver-api/pm/devices.html)
- [Qualcomm Linux 产品入口](https://www.qualcomm.com/developer/software/qualcomm-linux)
- [Qualcomm Linux `meta-qcom`](https://github.com/qualcomm-linux/meta-qcom)
- [Linux arm64 Qualcomm DTS](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/boot/dts/qcom)
- [MediaTek Genio 软件资源入口](https://mediatek.gitlab.io/)
- [MediaTek IoT Yocto Developer Guide](https://mediatek.gitlab.io/aiot/doc/aiot-dev-guide/master/)
- [Ubuntu on Genio 文档](https://genio.mediatek.com/doc/ubuntu/index.html)
