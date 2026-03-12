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
