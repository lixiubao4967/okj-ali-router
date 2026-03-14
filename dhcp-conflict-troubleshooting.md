# FortiGate DHCP 故障排查记录

---

## 问题一：VLAN2100 无线用户网关异常，无法上网

### 故障概要

| 项目 | 内容 |
|------|------|
| 发生日期 | 2026-03-13 |
| 影响范围 | 通过 AP 无线连接的用户无法上网 |
| 故障现象 | 用户获取的默认网关从 `10.20.11.254` 变为 `10.20.11.2`，导致无法访问互联网 |

## 网络拓扑

```
Internet
  │
FortiGate (10.20.11.254)   ← 正确的网关，DHCP Server
  │
  ├─ 接口: 10.20.11.254/24，已开启 DHCP
  │
  └─ AP (10.20.11.2)        ← 疑似 rogue DHCP Server
       │
       └─ 无线客户端
```

## 故障时间线

### 正常阶段（数月）

- FortiGate 接口 `10.20.11.254` 开启 DHCP 功能，为该子网提供地址分配
- AP 连接在该接口下，用户通过 AP 无线连接后正常获取 IP
- 获取的网关为 `10.20.11.254`（FortiGate），一切正常

### 故障阶段（2026-03-13 起）

1. 用户突然无法上网
2. 检查发现用户获取的 IP 地址本身没问题（在正确的子网内）
3. **但默认网关变成了 `10.20.11.2`**，而非正确的 `10.20.11.254`
4. 手动在用户端将网关设置为 `10.20.11.254` → **仍然无法到达网关**
5. `ping 10.20.11.2` → **通**
6. `ping 10.20.11.254` → **不通**

## 故障分析

### 核心问题：Rogue DHCP Server（流氓 DHCP 服务器）

AP（`10.20.11.2`）极有可能自身具备 DHCP Server 功能，与 FortiGate 的 DHCP 形成冲突。

#### 为什么之前正常、现在才出问题？

DHCP 采用广播机制，客户端发送 `DHCPDISCOVER` 后，**谁先响应（DHCPOFFER）谁就"赢"**。

- **前期**：FortiGate 的 DHCP 响应速度更快，抢先分配了地址和网关（`10.20.11.254`），AP 的 DHCP 虽然也可能在响应，但被客户端忽略
- **现在**：可能由于以下原因，AP 的 DHCP 响应开始抢先：
  - 客户端原有租约到期，重新发起 DHCPDISCOVER
  - AP 固件更新或重启后 DHCP 响应变快
  - FortiGate 负载变化导致响应变慢
  - 网络拓扑上 AP 离客户端更近（无线直连 vs 经过交换），响应天然更快

#### 为什么手动设置正确网关也不通？

`ping 10.20.11.254` 不通说明问题不仅仅是网关地址错误，可能存在以下情况：

1. **AP 做了 NAT 或隔离**：AP 的 DHCP 可能将无线客户端放在了一个隔离的子网或 NAT 网络中，虽然地址看起来是 `10.20.11.x`，但实际上被 AP 隔离，无法直接到达 FortiGate
2. **AP 启用了客户端隔离（Client Isolation）**：部分 AP 在启用自身 DHCP 后会隔离客户端，阻止其访问同网段的其他设备
3. **二层隔离**：AP 可能在桥接模式和路由模式之间发生了切换，导致流量不再直接桥接到 FortiGate 所在的 VLAN

### 排查进展

通过 FortiGate DHCP 租约表发现可疑设备：

| IP | MAC | VCI | 说明 |
|----|-----|-----|------|
| `10.20.11.24` | `e0:d3:62:e6:f0:16` | `udhcp 1.22.1` | 无 Hostname，确认为 **TP-Link AP** |

- `udhcp` 是 BusyBox 内置 DHCP 客户端，典型的嵌入式设备（AP/路由器）
- 该 AP 之前静态配置为 `10.20.11.2`，后被重置/重启，从 FortiGate DHCP 获取了 `.24` 地址
- 这解释了为什么 `10.20.11.2` 后来 ping 不通了 — AP 已不在该地址

## 解决方案

### 方案一：登录 TP-Link AP 禁用其 DHCP 功能（根治）

AP 当前 IP 为 `10.20.11.24`（MAC: `e0:d3:62:e6:f0:16`），尝试以下方式登录管理界面：

- `http://10.20.11.24` / `https://10.20.11.24`
- `http://10.20.11.24:8080`
- TP-Link 默认管理地址 `http://tplogin.cn`
- 用网线直连 AP 的 LAN 口访问

登录后操作：

1. **关闭 DHCP Server 功能**
2. **将工作模式设为桥接模式（Bridge/AP Mode）**，而非路由模式（Router Mode）
3. 保存并重启 AP

### 方案二：FortiGate 开启 DHCP Snooping（防复发）

即使 AP 以后被重置导致 DHCP 重新开启，FortiGate 的 DHCP Snooping 会在交换层面拦截非授权的 DHCP Offer，确保客户端只收到 FortiGate 的正确响应：

```
config system interface
    edit "VLAN2100"
        set dhcp-snooping enable
    next
end
```

同时将 AP 的 MAC 绑定固定 IP，方便后续管理：

```
config system dhcp server
    edit 3
        config reserved-address
            edit 1
                set ip 10.20.11.2
                set mac e0:d3:62:e6:f0:16
                set description "TP-Link-AP-VLAN2100"
            next
        end
    next
end
```

> **方案一是根治，方案二是防复发。建议两个都做。**

## 验证步骤

修复后执行以下检查：

```bash
# 1. 确认获取的网关地址
ipconfig getifaddr en0        # macOS 获取 IP
ipconfig getoption en0 router # macOS 获取网关

# 2. 确认网关可达
ping -c 4 10.20.11.254

# 3. 确认互联网可达
ping -c 4 8.8.8.8
curl -s https://httpbin.org/ip

# 4. （可选）抓包确认 DHCP 来源
sudo tcpdump -i en0 -n port 67 or port 68
# 观察 DHCP Offer 的 server-id 是否为 10.20.11.254
```

### 结论

本次故障的根本原因高度疑似 **AP 内置的 DHCP Server 与 FortiGate DHCP 冲突**（Rogue DHCP）。前期因 FortiGate 抢先响应而未暴露，租约到期后 AP 抢先响应导致客户端获取了错误的网关。修复方向是**禁用 AP 的 DHCP 功能并确保 AP 工作在桥接模式**。

---

## 问题二：VLAN2201 新设备无法自动获取 IP 地址

### 故障概要

| 项目 | 内容 |
|------|------|
| 发生日期 | 2026-03-14 |
| 影响范围 | `10.20.13.0/24`（VLAN2201）网段新接入的设备 |
| 故障现象 | 新设备无法通过 DHCP 自动获取 IP 地址 |

### 原因分析

FortiGate DHCP Server（edit 5）针对 VLAN2201 接口配置了 **MAC 地址白名单机制**：

```
config system dhcp server
    edit 5
        set mac-acl-default-action block    ← 关键配置：默认拒绝
        set interface "VLAN2201"
        ...
        config reserved-address
            edit 1  → okjppc001  3c:2c:30:e2:12:74
            edit 2  → okjppc002  3c:2c:30:e2:10:1a
            edit 3  → okjppc006  3c:2c:30:e2:16:60
            ...（共 14 台设备）
        end
    next
end
```

`mac-acl-default-action block` 表示：**只有 `reserved-address` 列表中登记了 MAC 地址的设备才能获取 IP，所有未登记的新设备都会被 DHCP 拒绝**。

当前白名单中仅有 14 台设备（okjppc001-012、okjsvr101、okjwinsv01），任何新设备的 MAC 不在列表中，DHCP 请求直接被丢弃。

### 解决方案

#### 方案一：将新设备 MAC 加入白名单（推荐，保持安全管控）

```
config system dhcp server
    edit 5
        config reserved-address
            edit 15
                set ip 10.20.13.15
                set mac xx:xx:xx:xx:xx:xx
                set description "新设备名称"
            next
        end
    next
end
```

> 将 `xx:xx:xx:xx:xx:xx` 替换为新设备的实际 MAC 地址。

#### 方案二：取消 MAC 白名单限制（允许所有设备获取地址）

```
config system dhcp server
    edit 5
        set mac-acl-default-action assign
    next
end
```

> **注意**：该网段连接的是生产 PC（okjppc 系列），`block` 策略可能是有意为之的安全管控。取消前请确认是否符合安全策略。

### 验证步骤

```bash
# 在 FortiGate 上查看 DHCP 分配日志
diagnose debug application dhcps -1
diagnose debug enable

# 新设备端重新获取地址
# Windows:
ipconfig /release && ipconfig /renew

# macOS:
sudo ipconfig set en0 DHCP

# 确认获取到 10.20.13.x 地址和正确网关 10.20.13.254
```
