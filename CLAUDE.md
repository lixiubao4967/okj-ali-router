# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **documentation-only repository** recording a DNS split-horizon networking issue and its FortiGate firewall solution. There is no executable code, build system, or test framework.

## Content Overview

The `README.md` documents:

1. **Problem**: Alibaba Cloud VPC API (`vpc.ap-northeast-1.aliyuncs.com`) intermittently timing out, causing `PublicIP` to show as `None` in ECS inspection scripts.

2. **Root Cause**: FortiGate queries Primary DNS (`1.1.1.1`) and Secondary DNS (`172.20.0.10`, Alibaba internal) simultaneously. The internal DNS responds faster via VPN and returns `100.100.1.231` — an Alibaba VPC metadata IP only reachable from inside ECS instances. FortiGate has no route for `100.100.0.0/16`, so traffic black-holes.

3. **Solution**: FortiGate DNS split-horizon — forward only `*.cluster.local` (Kubernetes internal) to the internal DNS `172.20.0.10`; route all other domains (including `*.aliyuncs.com`) to public DNS.

4. **Workaround**: On macOS, manually set DNS to public servers via `networksetup -setdnsservers Wi-Fi 8.8.8.8 1.1.1.1` (resets on network reconnect).

## Site-to-Site IPsec VPN + 异地公网出口

`fortigate-site-to-site-ipsec-vpn.md` 文档记录了 OKJ 与 OKBL 两个办公点之间的 IPsec VPN 方案，核心技术要点如下：

### 需求

OKBL（FG-B）的内网用户需要通过 VPN 隧道，借 OKJ（FG-A）的公网 IP 出口访问互联网。

### 核心难点：路由环路

站点到站点 VPN 本身依赖公网默认路由（`0.0.0.0/0 → wan1`）建立隧道。如果把默认路由改指隧道以实现"所有流量走对端出网"，VPN 封装包也会被送入隧道，导致隧道断联——这就是经典的 **VPN 路由环路**问题。

### 两种解决方案

1. **策略路由 (PBR)**（推荐）：默认路由不动，利用 FortiGate 策略路由按**源地址 + 入接口**匹配，仅将 LAN 用户流量导入隧道。FG-B 自身发出的 VPN 封装包（源 IP 非 LAN 网段）不匹配策略路由，仍走 wan1，隧道不受影响。

2. **主机路由 /32**：为对端公网 IP 添加一条 `/32` 精确路由指向 wan1，再将默认路由改为走隧道。`/32` 比 `0.0.0.0/0` 优先级高，VPN 封装包走 `/32` 出 wan1，其余流量走隧道。缺点是 FG-B 自身管理流量也走隧道，隧道故障则全断。

### 关键认知：策略路由 vs 普通路由

- **普通路由**：仅根据目的地址做转发决策
- **策略路由**：可同时匹配源地址、目的地址、入接口、协议等多维条件，且优先级高于普通路由表

### 配置要点

- Phase 2 需要两条 SA：一条 LAN 互通，一条 `0.0.0.0/0 ↔ 远端 LAN`（出网用）
- FG-A（出口侧）需要 VPN → WAN 防火墙策略并**开启 NAT**（SNAT 为自身公网 IP）
- FG-B 侧需要 `internal → vpn-to-fga (dstaddr all)` 策略放行出网流量（PBR 只管路由，防火墙策略管放行）
- LAN 互通策略必须**关闭 NAT**
- 所有隧道策略配置 TCP MSS Clamping（1360）防止 MTU 黑洞
- link-monitor 监测隧道健康，故障时自动撤回路由实现回退

## Network Topology

```
Mac Client
  └─ WiFi Gateway 10.20.21.254 (FortiGate)
       ├─ Primary DNS:   1.1.1.1
       ├─ Secondary DNS: 172.20.0.10 (Alibaba Cloud internal, k8s DNS)
       └─ Static routes:
            172.20.0.0/16 → 10.20.18.105 (via OpenVPN to Alibaba Cloud)
            10.241.0.0/24 → 10.20.18.105
```
