---
name: tun-fake-ip-diagnose
description: TUN 模式 + fake-ip 模式下 DNS 被劫持到 198.18.0.0/15、ping 假通、TUN 虚拟网卡抢占路由的识别与排查。当用户反馈"所有 ping 都解析到 198.18.x.x"、"网络一切正常但实际是假的"、"装了代理后 ping 都是 0-4ms"等异常时加载此 skill。
metadata:
  version: "2026.6.1"
---

# TUN 模式 + fake-ip 模式诊断

## 适用场景

- `ping baidu.com` / `ping google.com` 等所有域名都解析到 `198.18.0.x`（`198.18.0.0/15` 段）
- ping 显示"0-4ms 通"，但实际根本没连到真实服务器
- `Resolve-DnsName <domain>` 也返回 `198.18.0.x`
- 系统 DNS 服务器是 `198.18.0.2` 或 `198.18.0.3`（不是常见的 `192.168.x.1` / `8.8.8.8`）
- 网络适配器列表里出现 `Meta` / `Clash` / `sing-box` / `Hiddify` / `v2ray` 等名字的 TUN 虚拟网卡
- 路由表里出现 `0.0.0.0/0 → 198.18.0.x` 且 metric 最小

## 关键背景

### 198.18.0.0/15 是什么

`198.18.0.0/15`（即 `198.18.0.0` ~ `198.19.255.255`）是 IANA / IETF **专门保留给 benchmark testing 用的地址段**（RFC 2544），**正常公网 DNS 永远不会返回这个段**。一旦 DNS 解析里出现 `198.18.x.x`，**100% 是有工具在抢答**。

### fake-ip 模式原理

主流代理客户端（Clash Verge / Mihomo / Clash for Windows / v2rayN / sing-box / Hiddify 等）的 **fake-ip 模式**：

1. **本地 DNS 抢答**：在 TUN 网卡上启一个本地 DNS（通常是 `198.18.0.2`），所有 DNS 请求被截到它
2. **返回假 IP**：用一致性哈希从 `198.18.0.0/15` 池子里挑一个 fake IP 返回，**同一域名每次都返回同一个 fake IP**（保证连接复用）
3. **TUN 接管流量**：客户端发包目标 IP 是 fake IP，路由表里 `0.0.0.0/0 → 198.18.0.x` 把流量劫到 TUN
4. **出口再解析**：代理客户端在出口把 fake IP 换回真实 IP（通过本地记录的 fake-IP ↔ 真实域名映射表），再走代理或直连

### 为什么 ping 看起来"通"但实际是假的

```
ping opencode.ai
  → DNS 解析 → 198.18.0.131  (fake)
  → 路由表命中 198.18.0.0/15 段 → 走 Meta Tunnel
  → TUN 拦截到 ICMP 请求 → 假装回一个 0-4ms 的响应
  → 你看到的 "0ms 4ms" 全是 TUN 自导自演，根本没出你的电脑
```

ICMP 在 TUN 里没法像 TCP/UDP 那样被代理到上游，只能本地伪造回包。

## 诊断步骤

按顺序执行，每步独立判断。

### 步骤 1：确认 hosts 文件是否被改

```powershell
Get-Content "$env:WINDIR\System32\drivers\etc\hosts" | Select-String -Pattern 'opencode|baidu|google|github|claude' -SimpleMatch:$true
Get-Item "$env:WINDIR\System32\drivers\etc\hosts" | Select-Object Length, LastWriteTime
```

- 如果 hosts 里没有这些条目 → **不是 hosts 问题**，进入步骤 2
- 如果有 → **hosts 被改了**，直接编辑 hosts 删除对应行

### 步骤 2：看真实 DNS 解析（系统 DNS）

```powershell
Resolve-DnsName opencode.ai -Type A
Resolve-DnsName baidu.com -Type A
```

- 如果返回 `198.18.x.x` → **DNS 服务器被劫持**，进入步骤 3
- 如果返回真实公网 IP（`104.x`、`220.181.x` 等）→ 不是 fake-ip，是其他问题

### 步骤 3：看系统 DNS 服务器

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, ServerAddresses
```

- 看到 `198.18.0.2` 或 `198.18.0.3` → **100% 是 fake-ip 模式的本地 DNS**，不是运营商 DNS
- 看到 `192.168.x.1` / `8.8.8.8` / `1.1.1.1` 等 → 不是 fake-ip

### 步骤 4：看网络适配器有没有 TUN 虚拟网卡

```powershell
Get-NetAdapter | Where-Object { $_.Name -match 'tun|tap|clash|meta|mihomo|sing|v2ray|netch|proxy|http|ethernet|Meta|Hiddify' -or $_.Status -eq 'Up' } | Select-Object Name, InterfaceDescription, Status, LinkSpeed
```

常见 TUN 网卡名：
- `Meta` / `Meta Tunnel` —— Mihomo / Clash Verge 默认
- `Hiddify` —— Hiddify
- `sing-box` / `tun0`
- `Clash` / `Clash TUN`
- `v2rayN` / `v2ray`

### 步骤 5：看路由表

```powershell
route print | Select-String -Pattern '198\.18\.|0\.0\.0\.0' | Select-Object -First 6
```

重点看：

```
0.0.0.0          0.0.0.0      192.168.1.1     192.168.1.88     35    ← 物理网卡默认网关
0.0.0.0          0.0.0.0       198.18.0.2       198.18.0.1      0     ← TUN 默认路由（metric 最小，优先）
```

**第二条 metric=0 比 35 小、优先**，所有没匹配其他路由的流量都被 TUN 截走。

### 步骤 6：找代理进程

```powershell
Get-Process | Where-Object { $_.ProcessName -match 'clash|mihomo|verge|v2ray|nekoray|sing-box|hiddify|surge|quantumult|shadowsocks|trojan|naive|ssr|netch|outline|gost|kool' } | Select-Object Id, ProcessName, Path

Get-ScheduledTask -ErrorAction SilentlyContinue | Where-Object { $_.TaskName -match 'clash|mihomo|verge|v2ray|sing|hiddify|proxy|tun' } | Select-Object TaskName, State
```

## 解决方案

### 方案 A：临时验证（不改配置）

```powershell
# 让某个应用绕过 TUN，看能否正常解析
# 方法 1：用 nslookup 指定远程 DNS
nslookup opencode.ai 8.8.8.8

# 方法 2：用 curl 带 --resolve 直接连真实 IP
curl -v --resolve opencode.ai:443:<真实IP> https://opencode.ai/
```

如果远程 DNS 能解析到真 IP → 100% 确认是 fake-ip 劫持。

### 方案 B：切回 redir-host 模式（推荐）

不同客户端配置位置：
- **Clash Verge / Mihomo**：`Settings → System Proxy / TUN Mode → DNS Strategy` 改 `redir-host`
- **v2rayN**：关闭 fake IP 改 `UseIP` 模式
- **sing-box**：配置文件里把 `inet4_address` 段从 `198.18.0.0/15` 改成 `9.9.9.9/32` 之类
- **Hiddify**：`Settings → DNS` 改 `Real IP`

### 方案 C：完全关闭 TUN / 改系统代理模式

如果只用浏览器需要代理，可以从 TUN 模式切到 **系统代理模式**（不开 TUN），DNS 就不会被劫持，但只有配置了系统代理的应用才能走代理。

## 重要交互影响

| 流量 | 是否受 TUN / fake-ip 影响 | 说明 |
|------|------------------------|------|
| 浏览器 HTTPS | ✓ 受 fake-ip 影响但**功能正常** | TLS 握手会被 TUN 正确代理到真实 IP |
| `ping` / `tracert` | ✗ **假通** | ICMP 没法被代理，只能本地伪造 |
| `nslookup` / `Resolve-DnsName` | ✗ 被 fake-ip DNS 抢答 | 想看真实 DNS 要带 remote DNS server |
| `127.0.0.1` / loopback | ✗ **不受影响** | TUN 默认不接管 loopback |
| 本地端口（如 cc-switch `127.0.0.1:15721`） | ✗ **不受影响** | 所以本地代理/服务都正常工作 |

### 关键推论

- **loopback 不走 TUN** → 本地起的服务（cc-switch、dev server、数据库）始终正常
- **HTTPS 应用层正常** → 浏览器、claude/codex CLI 的 API 请求都能用，只是 ping/ICMP 工具会骗你
- **要确认"网络真通不通"** → 用 `curl -v <url>` 或 `Test-NetConnection -Port 443`，**别用 ping**

## 关键概念速查

| 概念 | 含义 |
|------|------|
| `198.18.0.0/15` | IANA benchmark 保留段，fake-ip 模式必现 |
| `Meta Tunnel` | Mihomo / Clash Verge TUN 模式的虚拟网卡默认名 |
| `0.0.0.0/0` | 默认路由（匹配所有目标 IP） |
| `metric` | 路由优先级，**数值越小越优先**（TUN 模式通常 metric=0 抢答） |
| TUN 模式 | 在系统层创建虚拟网卡，**接管所有应用的网络流量** |
| fake-ip 模式 | 配合 TUN 用，DNS 返回假 IP 防止污染 + 加速解析 |
| redir-host 模式 | 传统模式，DNS 返回真实 IP，靠 hosts 思路 |
| `127.0.0.1` / loopback | 本机回环，TUN 默认不接管，**本地服务永远正常** |

## 触发关键词

用户提到以下任一关键词时加载此 skill：
- "ping 假通"
- "198.18"
- "所有 DNS 都返回 198"
- "TUN 模式"
- "fake-ip"
- "装了代理后 ping 不正常"
- "Clash Verge" / "Mihomo" / "Hiddify" / "v2ray"
- "网络一切正常但实际是假的"
- "DNS 被劫持"
- "ping 0ms 但没连上"

## 注意事项

1. **不要建议用户"关掉代理"** —— 用户用代理是有需求的，应该定位"哪种模式"而不是"关掉"
2. **优先说"切 redir-host"或"关闭 TUN 改系统代理"**，不要简单说"卸载代理"
3. **loopback 流量不受影响**——这点要先告诉用户，免得他误以为"整个网络都坏了"
4. **HTTPS 业务流量正常**——浏览器、API 调用都没事，只有诊断类工具会骗你
5. **诊断完不要自动改路由表 / 改 DNS** —— 任何网络层修改都先让用户确认

## Skill 信息

- **Skill 名称**: tun-fake-ip-diagnose
- **提示标题**: TUN / fake-ip 模式诊断
- **归属**: 个人 skills
- **关键字**: fake-ip, TUN, 198.18, Clash, Mihomo, DNS 劫持, ping 假通, 路由表
