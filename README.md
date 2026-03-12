# OKJ Ali Router - 网络问题记录与解决方案

## 问题：调用阿里云 API 时 VPC 相关接口超时

### 现象

执行阿里云 ECS 巡检脚本时，`vpc.ap-northeast-1.aliyuncs.com` 的 API 调用间歇性超时：

```
ERROR: Post "https://vpc.ap-northeast-1.aliyuncs.com/...": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

- ECS 接口有时正常，VPC 接口基本必超时
- 导致所有实例的 PublicIP 显示为 `None`

---

### 根因分析

#### 网络拓扑

```
Mac 本机
  └─ WiFi 网关 10.20.21.254（FortiGate 防火墙）
       ├─ Primary DNS:   1.1.1.1（公网）
       ├─ Secondary DNS: 172.20.0.10（阿里云内网 DNS，用于 k8s 域名解析）
       └─ 静态路由：
            172.20.0.0/16 → 10.20.18.105（via General-LAN/wan2，走 OpenVPN 到阿里内网）
            10.241.0.0/24 → 10.20.18.105（via General-LAN/wan2）
```

#### 问题根因

FortiGate 的 Primary/Secondary DNS **并非严格顺序模式**，而是**同时查询两个 DNS，谁先响应用谁**。

| DNS 服务器 | 路径 | 延迟 |
|-----------|------|------|
| `1.1.1.1`（Primary） | 出公网 | 较高 |
| `172.20.0.10`（Secondary） | 走 VPN 内网路由，直连 | 极低（几毫秒） |

因此内网 DNS `172.20.0.10` 经常抢先响应。

而阿里云内网 DNS 对 `vpc.ap-northeast-1.aliyuncs.com` 返回的是阿里云内部 IP `100.100.1.231`（`100.100.0.0/16` 是阿里云 VPC 内部的 metadata 路由段，仅 ECS 实例内部可访问）。

FortiGate 路由表中没有 `100.100.0.0/16` 的路由，流量到这个 IP 直接黑洞，导致超时。

#### 对比验证

```bash
# 内网 DNS 解析结果（错误）
$ dig vpc.ap-northeast-1.aliyuncs.com
;; ANSWER SECTION:
vpc.ap-northeast-1.aliyuncs.com. → 100.100.1.231  ❌ 阿里内网 IP，外部不可达

# 公网 DNS 解析结果（正确）
$ dig @8.8.8.8 vpc.ap-northeast-1.aliyuncs.com
;; ANSWER SECTION:
vpc.ap-northeast-1.aliyuncs.com. → 47.91.8.8      ✅ 公网 IP，正常访问
```

---

### 解决方案：FortiGate 配置 DNS 条件转发（Split DNS）

不让所有域名都查询 `172.20.0.10`，只把 k8s 内部域名转给它，其余走公网 DNS。

#### FortiGate CLI 配置

```
# 只将 k8s 内部域名转发给内网 DNS
config system dns-database
    edit "k8s-internal"
        set domain "cluster.local"     # 替换为实际的 k8s 内部域名后缀
        set type forward
        set forwarder "172.20.0.10"
    next
end

# 全局 DNS 只保留公网，去掉 172.20.0.10
config system dns
    set primary 1.1.1.1
    set secondary 8.8.8.8
end
```

#### 效果

| 域名类型 | 走向 |
|---------|------|
| `*.cluster.local`（k8s 内部） | → `172.20.0.10` 解析 ✅ |
| `*.aliyuncs.com` 等公网域名 | → `1.1.1.1` 解析，返回公网 IP ✅ |

---

### Mac 本机端解决方案：/etc/resolver 配置分域 DNS

在无法修改防火墙配置时，可在 Mac 本机通过 `/etc/resolver/` 目录为 `aliyuncs.com` 单独指定 DNS，只影响该域名的解析，其他域名不受影响，且**持久有效（重启、网络切换均不丢失）**：

```bash
sudo mkdir -p /etc/resolver
echo "nameserver 8.8.8.8" | sudo tee /etc/resolver/aliyuncs.com
```

验证生效：

```bash
scutil --dns | grep -A5 aliyuncs
```

应看到：

```
resolver #X
  domain   : aliyuncs.com
  nameserver[0] : 8.8.8.8
```

验证解析结果正确：

```bash
dig vpc.ap-northeast-1.aliyuncs.com
# 应返回公网 IP（如 47.91.x.x），而不是 100.100.x.x
```

效果：`*.aliyuncs.com` 域名解析强制走 `8.8.8.8`，k8s 内部域名解析不受影响。

> **为什么不用 scutil 交互模式？**
> `scutil` 写入 `State:` 键是内存态，网络切换（重连 WiFi、VPN 重连、DHCP 刷新）后即丢失。`/etc/resolver/` 文件由 mDNSResponder 直接读取，持久有效。

---

### 临时绕过方法

将本机所有 DNS 改为公网（会影响 k8s 内部域名解析）：

```bash
sudo networksetup -setdnsservers Wi-Fi 8.8.8.8 1.1.1.1
```

注意：此方法在网络切换（重连 WiFi、DHCP 刷新）后可能失效，需重新设置。
