# FortiGate 60 站点到站点 IPsec VPN 配置指南

## 一、网络拓扑

```
FG-A (OKJ)                                 FG-B (OKBL)
公网 IP: 150.249.195.73                    公网 IP: 121.101.74.242
内网: 10.20.21.0/24                        内网: 10.120.23.0/24
WAN 接口: wan1                              WAN 接口: wan1
```

目标：实现两侧内网互通

```
10.20.21.0/24  ←──IPsec Tunnel──→  10.120.23.0/24
```

> **注意**：内网网段请根据实际环境替换，两侧网段不能重叠。

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

### FG-A 侧

```
config vpn ipsec phase2-interface
    edit "vpn-to-fgb-p2"
        set phase1name "vpn-to-fgb"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set src-subnet 10.20.21.0 255.255.255.0
        set dst-subnet 10.120.23.0 255.255.255.0
        set keylifeseconds 3600
    next
end
```

### FG-B 侧（网段反过来）

```
config vpn ipsec phase2-interface
    edit "vpn-to-fga-p2"
        set phase1name "vpn-to-fga"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set src-subnet 10.120.23.0 255.255.255.0
        set dst-subnet 10.20.21.0 255.255.255.0
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

> **重要**：站点到站点 VPN 策略中 NAT 必须关闭 (`set nat disable`)，否则源 IP 会被改写，对端无法正确路由回包。

---

## 七、静态路由

Route-based VPN 会自动创建虚拟接口（即 Phase1 的 tunnel 名称），需要手动添加静态路由。

### FG-A 侧

```
config router static
    edit 0
        set dst 10.120.23.0 255.255.255.0
        set device "vpn-to-fgb"
    next
end
```

### FG-B 侧

```
config router static
    edit 0
        set dst 10.20.21.0 255.255.255.0
        set device "vpn-to-fga"
    next
end
```

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
