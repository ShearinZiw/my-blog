+++
date = '2026-04-14T05:03:00+08:00'
lastmod = '2026-08-12T00:00:00+08:00'
title = '网络与嵌入式协议栈导航'
categories = ['interview', 'network']
tags = ['八股', '面试', '网络', 'lwIP', '导航']
summary = '从 TCP/IP 协议语义进入 lwIP 实现，再映射到乐鑫 Connectivity 的连接生命周期。'
+++

# 网络与嵌入式协议栈导航

## 推荐顺序

1. [网络分层、封装与 IP 基础]({{< relref "../network/basic.md" >}})
2. [计算机网络基础与 TCP]({{< relref "../network/TCP.md" >}})
3. [lwIP：线程模型、pbuf 与稳定性诊断]({{< relref "../embedded/lwip.md" >}})
4. [乐鑫 Connectivity 专题]({{< relref "../company/espressif-connectivity.md" >}})

## 概念唯一归属

- TCP 握手、可靠性、窗口、重传与关闭语义由网络基础文章解释；
- `tcpip_thread`、PCB、pbuf、netif 与 `lwipopts.h` 由 lwIP 主干解释；
- Wi-Fi STA 状态、配网、重连、共存和 OTA 由乐鑫专题解释。

这样可以沿“协议 → 协议栈 → 产品连接生命周期”追问，又不会维护三份互相漂移的 TCP 答案。

## 项目准备

至少准备一份抓包、一条协议栈统计基线、一次断网/重连故障注入，以及吞吐、延迟、丢包、内存峰值中的两个量化指标。
