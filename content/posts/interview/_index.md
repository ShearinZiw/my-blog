+++
title = '知识地图'
description = '围绕嵌入式岗位能力组织的复习、项目追问与投递入口。'
weight = 1
+++

# 嵌入式知识地图

这里不再按“看过哪些八股”组织，而是按**能否解释机制、做出取舍、定位故障**组织。

## 首先从这里开始

- [嵌入式岗位知识地图]({{< relref "embedded-roadmap.md" >}})：能力分层、复习闭环和项目追问模板。
- [五条技术主干]({{< relref "../embedded/_index.md" >}})：C 工具链、MCU、RTOS、Linux / 驱动、lwIP。
- [公司专题路线]({{< relref "../company/_index.md" >}})：乐鑫 Connectivity、高通 / 联发科 Linux BSP。

## 第二阶段主干

| 主干 | 30 秒基础题 | 工程追问 |
| --- | --- | --- |
| [C 工具链]({{< relref "../embedded/c-toolchain.md" >}}) | 源码怎样变成 ELF / 固件 | Map 里 RAM 为什么突然增长 |
| [MCU]({{< relref "../embedded/mcu.md" >}}) | 上电后怎样到 `main` | HardFault 怎样还原现场 |
| [RTOS]({{< relref "../embedded/rtos.md" >}}) | 调度、阻塞、同步的区别 | 偶发超时怎样证明是优先级反转 |
| [Linux / 驱动]({{< relref "../embedded/linux-driver.md" >}}) | device、driver 怎样匹配 | `probe` 不执行怎样分层排查 |
| [lwIP]({{< relref "../embedded/lwip.md" >}}) | raw、netconn、socket 如何选 | 跑几小时断流怎样查 pbuf 与窗口 |

## 基础专题存量

这些文章保留为通用计算机基础和扩展阅读；新的嵌入式主干承担概念边界与稳定入口。

### C 与 C++

- [C 基础与内存]({{< relref "../c/basic.md" >}})
- [C++ 面试速背稿]({{< relref "../cpp/cpp-interview-cheatsheet.md" >}})
- [虚函数与内联]({{< relref "../cpp/virtualcpp.md" >}})
- [C++ 专题导航]({{< relref "cpp-roadmap.md" >}})

### 操作系统与 Linux 应用

- [操作系统面试整理]({{< relref "../os/os.md" >}})
- [进程通信与并发问题]({{< relref "../os/IPC.md" >}})
- [中断与中断上下文]({{< relref "../os/interupt.md" >}})
- [操作系统专题导航]({{< relref "os-roadmap.md" >}})

### 网络

- [计算机网络基础与 TCP]({{< relref "../network/TCP.md" >}})
- [网络专题导航]({{< relref "network-roadmap.md" >}})

## 把答案变成可展示的项目证据

每篇主干文章都准备同一组材料：系统边界图、一次取舍、一次失败、一个量化指标、一条可复现实验。投递时展示解决问题的链路，不展示背了多少名词。
