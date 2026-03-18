# FortiGate 60 站点到站点 IPsec VPN 配置指南

## 一、网络拓扑

```
FG-A (OKJ)                                 FG-B (OKBL)
公网 IP: 150.249.195.73                    公网 IP: 121.101.74.242
内网: 10.20.21.0/24                        内网: 10.120.23.0/24
WAN 接口: wan1                              WAN 接口: wan1
```

目标：
1. 两侧内网互通（`10.20.21.0/24 ↔ 10.120.23.0/24`）
2. **FG-B (OKBL) 的内网流量借 FG-A (OKJ) 的公网出口上网**

```
OKBL 客户端 (10.120.23.x)
  → FG-B → IPsec Tunnel → FG-A → NAT(150.249.195.73) → 互联网
```

> **注意**：内网网段不能重叠。

---

## 二、配置逻辑概览

IPsec VPN 由三部分组成：

| 步骤 | 说明 |
|------|------|
| Phase 1 (IKE) | 建立加密隧道，协商密钥 |
| Phase 2 (IPsec SA) | 定义哪些网段走隧道加密 |
| 防火墙策略 + 路由 | 放行流量并指定路由路径 |

---

## 三、Phase 1 配置（IKE — 两边都要配）

### FG-A 侧 (150.249.195.73)

**GUI 路径**: VPN > IPsec Tunnels > Create New > Custom

| 参数 | 值 |
|------|-----|
| Name | `vpn-to-fgb` |
| Remote Gateway | `121.101.74.242` |
| Interface | `wan1` |
| IKE Version | 2 (推荐) |
| Authentication | Pre-Shared Key |
| Pre-Shared Key | `YourStrongPSK2024!` (两边必须一致) |
| Encryption | AES256 |
| Authentication (Hash) | SHA256 |
| DH Group | 14 (2048-bit) 或 19 (ECP256) |
| Key Lifetime | 86400 (秒) |
| Dead Peer Detection | Enable |

**CLI 方式**：

```
config vpn ipsec phase1-interface
    edit "vpn-to-fgb"
        set interface "wan1"
        set ike-version 2
        set peertype any
        set proposal aes256-sha256
        set dhgrp 14
        set remote-gw 121.101.74.242
        set psksecret "YourStrongPSK2024!"
        set dpd on-idle
        set dpd-retryinterval 10
    next
end
```

### FG-B 侧 (121.101.74.242)

配置完全对称，仅 `remote-gw` 不同：

```
config vpn ipsec phase1-interface
    edit "vpn-to-fga"
        set interface "wan1"
        set ike-version 2
        set peertype any
        set proposal aes256-sha256
        set dhgrp 14
        set remote-gw 150.249.195.73
        set psksecret "YourStrongPSK2024!"
        set dpd on-idle
        set dpd-retryinterval 10
    next
end
```

---

## 四、Phase 2 配置（IPsec SA）

需要两条 Phase 2：一条用于两侧内网互通，一条用于 OKBL 借 OKJ 出公网。

### FG-A 侧

```
config vpn ipsec phase2-interface
    edit "vpn-to-fgb-p2-lan"
        set phase1name "vpn-to-fgb"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set src-subnet 10.20.21.0 255.255.255.0
        set dst-subnet 10.120.23.0 255.255.255.0
        set keylifeseconds 3600
    next
    edit "vpn-to-fgb-p2-internet"
        set phase1name "vpn-to-fgb"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set src-subnet 0.0.0.0 0.0.0.0
        set dst-subnet 10.120.23.0 255.255.255.0
        set keylifeseconds 3600
    next
end
```

### FG-B 侧

```
config vpn ipsec phase2-interface
    edit "vpn-to-fga-p2-lan"
        set phase1name "vpn-to-fga"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set src-subnet 10.120.23.0 255.255.255.0
        set dst-subnet 10.20.21.0 255.255.255.0
        set keylifeseconds 3600
    next
    edit "vpn-to-fga-p2-internet"
        set phase1name "vpn-to-fga"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set src-subnet 10.120.23.0 255.255.255.0
        set dst-subnet 0.0.0.0 0.0.0.0
        set keylifeseconds 3600
    next
end
```

---

## 五、地址对象（用于防火墙策略）

### FG-A 侧

```
config firewall address
    edit "local-lan-a"
        set subnet 10.20.21.0 255.255.255.0
    next
    edit "remote-lan-b"
        set subnet 10.120.23.0 255.255.255.0
    next
end
```

### FG-B 侧

```
config firewall address
    edit "local-lan-b"
        set subnet 10.120.23.0 255.255.255.0
    next
    edit "remote-lan-a"
        set subnet 10.20.21.0 255.255.255.0
    next
end
```

---

## 六、防火墙策略（最容易遗漏）

### FG-A 侧

**内网 → 隧道（出站）**：

```
config firewall policy
    edit 0
        set name "lan-to-vpn-b"
        set srcintf "internal"
        set dstintf "vpn-to-fgb"
        set srcaddr "local-lan-a"
        set dstaddr "remote-lan-b"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

**隧道 → 内网（入站）**：

```
config firewall policy
    edit 0
        set name "vpn-b-to-lan"
        set srcintf "vpn-to-fgb"
        set dstintf "internal"
        set srcaddr "remote-lan-b"
        set dstaddr "local-lan-a"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

**隧道 → 公网（OKBL 借 OKJ 出网，需开启 NAT）**：

```
config firewall policy
    edit 0
        set name "vpn-b-to-internet"
        set srcintf "vpn-to-fgb"
        set dstintf "wan1"
        set srcaddr "remote-lan-b"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
```

> **注意**：此策略必须开启 NAT (`set nat enable`)，FG-A 会将 OKBL 的源 IP SNAT 为自己的公网 IP `150.249.195.73` 后转发到互联网。

### FG-B 侧（对称配置）

```
config firewall policy
    edit 0
        set name "lan-to-vpn-a"
        set srcintf "internal"
        set dstintf "vpn-to-fga"
        set srcaddr "local-lan-b"
        set dstaddr "remote-lan-a"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end

config firewall policy
    edit 0
        set name "vpn-a-to-lan"
        set srcintf "vpn-to-fga"
        set dstintf "internal"
        set srcaddr "remote-lan-a"
        set dstaddr "local-lan-b"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

> **重要**：LAN 互通的策略中 NAT 必须关闭 (`set nat disable`)，否则源 IP 会被改写，对端无法正确路由回包。

---

## 七、路由配置（核心难点）

### FG-A 侧（两种方案通用）

```
config router static
    edit 0
        set dst 10.120.23.0 255.255.255.0
        set device "vpn-to-fgb"
    next
end
```

### FG-B 侧 — 路由方案选择

FG-B 需要把 LAN 流量通过隧道送往 FG-A 出公网，但**默认路由 `0.0.0.0/0` 已经指向 wan1 网关（用于建立 VPN 隧道本身）**。如果直接把默认路由改指隧道，VPN 封装包也会走隧道，导致隧道断联（路由环路）。

以下提供两种方案解决此问题：

---

### 方案一：策略路由 (PBR) — 推荐

**原理**：默认路由不动，用策略路由按**源地址**将 LAN 流量导入隧道。VPN 封装包由 FG-B 自身发出（源不是 `10.120.23.0/24`），不会匹配策略路由，仍走 wan1。

**优点**：
- 默认路由不变，VPN 隧道稳定性高
- FG-B 自身管理流量（NTP、日志、固件更新）仍走本地公网
- 灵活，可按源/目的精细控制哪些流量走隧道

**静态路由（仅 LAN 互通）**：

```
config router static
    edit 0
        set dst 10.20.21.0 255.255.255.0
        set device "vpn-to-fga"
    next
end
```

> 默认路由 `0.0.0.0/0 → wan1` 保持不变。

**策略路由**：

```
config router policy
    edit 1
        set input-device "internal"
        set src "10.120.23.0/255.255.255.0"
        set dst "0.0.0.0/0.0.0.0"
        set output-device "vpn-to-fga"
    next
end
```

> 含义：从 internal 口进入、源为 `10.120.23.0/24`、目的为任意地址的流量 → 走 VPN 隧道到 FG-A。
>
> FG-B 自身发出的流量（源 IP 是 wan1 地址 `121.101.74.242` 或 internal 地址）不匹配此规则，仍走默认路由出 wan1，VPN 不受影响。

---

### 方案二：主机路由 + 默认路由改隧道

**原理**：添加一条 `/32` 主机路由指向 FG-A 公网 IP，确保 VPN 封装包走 wan1；然后将默认路由改为走隧道。`/32` 比 `0.0.0.0/0` 更精确，路由表优先匹配。

**优点**：
- 配置直观，所有流量默认走隧道
- 不需要理解策略路由概念

**缺点**：
- FG-B 自身上网也走隧道（除非额外排除）
- 如果 FG-A 端隧道故障，FG-B 所有上网都断

```
config router static
    # LAN 互通
    edit 0
        set dst 10.20.21.0 255.255.255.0
        set device "vpn-to-fga"
    next
    # 关键：确保 VPN 封装包走 wan1（/32 优先于 0.0.0.0/0）
    edit 0
        set dst 150.249.195.73 255.255.255.255
        set gateway <wan1-网关地址>
        set device "wan1"
    next
    # 默认路由改为走隧道
    edit 0
        set dst 0.0.0.0 0.0.0.0
        set device "vpn-to-fga"
        set priority 5
    next
end
```

> **注意**：
> - `<wan1-网关地址>` 替换为 FG-B wan1 的实际上游网关 IP。
> - 原有的默认路由 `0.0.0.0/0 → wan1` 如果 priority 更低（数字更小），需要删除或调高其 priority，确保隧道路由优先。
> - `/32` 主机路由保证了到 `150.249.195.73` 的 IKE/ESP 封装包始终走 wan1 物理出口。

---

### 两种方案对比

| | 方案一 (PBR) | 方案二 (主机路由) |
|---|---|---|
| 默认路由 | 不改动 | 改为隧道 |
| VPN 稳定性 | 高，路由层面无冲突 | 依赖 /32 优先级 |
| 灵活性 | 可按源/目的精细控制 | 全部流量走隧道 |
| FG-B 自身上网 | 仍走 wan1 本地出口 | 也走隧道（除非额外排除） |
| 隧道故障影响 | 仅 LAN 用户断网 | FG-B 所有上网都断 |

---

## 八、公网端口放行确认

双方 WAN 口必须允许以下协议通过（确认 ISP 未屏蔽）：

| 协议 | 端口 | 用途 |
|------|------|------|
| UDP | 500 | IKE 协商 |
| UDP | 4500 | NAT-T (NAT 穿越) |
| IP Protocol | 50 (ESP) | 加密数据传输 |

可用以下命令在对端确认连通性：

```bash
# 从外部测试 UDP 500 是否可达
nc -u -z 121.101.74.242 500
nc -u -z 150.249.195.73 500
```

---

## 九、验证与排障

### 检查隧道状态

```
# 查看 IKE SA (Phase 1)
diagnose vpn ike gateway list

# 查看 IPsec SA (Phase 2)
diagnose vpn tunnel list

# 查看隧道接口状态
get router info routing-table all
```

成功时应看到：

```
name=vpn-to-fgb  ...  established
```

### 手动触发隧道建立

```
diagnose vpn tunnel up vpn-to-fgb
```

### 实时调试（排障用）

```
# 开启 IKE 调试
diagnose debug application ike -1
diagnose debug enable

# 完成后关闭调试
diagnose debug disable
diagnose debug reset
```

### Ping 测试

从 FG-A 内网主机：

```bash
ping 10.120.23.1
traceroute 10.120.23.1
```

### 常见问题排查

| 症状 | 可能原因 | 解决方法 |
|------|----------|----------|
| Phase 1 无法建立 | PSK 不一致、加密算法不匹配 | 两端对比 Phase1 参数 |
| Phase 1 OK 但 Phase 2 失败 | 网段配置不对称 | 确认 src/dst subnet 互为镜像 |
| 隧道建立但 Ping 不通 | 缺防火墙策略或路由 | 检查策略和静态路由 |
| 间歇性断开 | DPD 超时、MTU 问题 | 调整 DPD 参数，降低 MTU |
| ESP 被 ISP 封锁 | 运营商过滤 Protocol 50 | 启用 NAT-T 强制封装为 UDP 4500 |

---

## 十、安全建议

1. **PSK 强度**：使用 20+ 字符的随机字符串，包含大小写字母、数字和特殊字符
2. **定期轮换密钥**：建议每 3-6 个月更换一次 PSK
3. **限制 Service**：防火墙策略中的 `service` 尽量不用 `ALL`，按需开放
4. **启用日志**：策略中开启 `set logtraffic all` 便于审计
5. **考虑证书认证**：如果设备支持，使用数字证书替代 PSK 更安全
