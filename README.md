# FRP 内网穿透工具部署文档

> 适用版本：frp v0.52+（推荐 v0.68+）  
> 配置格式：TOML（INI 已弃用，新功能仅支持 TOML/YAML/JSON）  
> 官方文档：https://gofrp.org  
> 项目地址：https://github.com/fatedier/frp

---

## 1. 架构说明

FRP（Fast Reverse Proxy）通过「公网服务端 + 内网客户端」实现内网穿透：

```
互联网用户
    │
    ▼
┌─────────────────────┐
│  云服务器（有公网 IP）  │
│  frps（服务端）       │  ← 监听 7000 等端口，接收外网连接
│  例：1.2.3.4         │
└──────────┬──────────┘
           │ 持久连接（frpc 主动连出）
           ▼
┌─────────────────────┐
│  家里服务器（无公网 IP）│
│  frpc（客户端）       │  ← 把本地服务映射到云端端口/域名
│  例：NAS / 树莓派 / PC │
└──────────┬──────────┘
           │
           ▼
   本地服务（SSH、Web、NAS 等）
```

| 角色  | 程序     | 部署位置         | 作用               |
|-----|--------|--------------|------------------|
| 服务端 | `frps` | 云服务器（有公网 IP） | 接收外网访问，转发到内网     |
| 客户端 | `frpc` | 家里服务器（NAT 后） | 主动连接 frps，注册本地服务 |

---

## 2. 部署前准备

### 2.1 云服务器要求

- 有公网 IPv4（或 IPv6，需两端一致）
- 系统：Linux（本文以 Ubuntu/Debian 为例，CentOS 类似）
- 开放端口（按需）：

| 端口       | 用途                         |
|----------|----------------------------|
| 7000     | frp 控制连接（`bindPort`，可改）    |
| 6000+    | TCP 穿透映射端口（如 SSH 用 6000）   |
| 80 / 443 | HTTP/HTTPS 域名穿透（可选）        |
| 7500     | frps 管理面板（建议仅内网或 SSH 隧道访问） |

### 2.2 家里服务器要求

- 能访问互联网（出站连到云服务器 7000 端口）
- 系统：Linux（NAS、树莓派、旧电脑、Docker 均可）
- 明确要穿透的本地服务及端口（如 SSH 22、Web 8080）

### 2.3 下载二进制

在 [GitHub Releases](https://github.com/fatedier/frp/releases) 按架构下载，例如：

```bash
# 云服务器 / 家里服务器均为 Linux amd64 时
wget https://github.com/fatedier/frp/releases/download/v0.68.1/frp_0.68.1_linux_amd64.tar.gz
tar -xzf frp_0.68.1_linux_amd64.tar.gz
cd frp_0.68.1_linux_amd64
```

常见架构对照：

| 设备               | 架构            |
|------------------|---------------|
| 普通云服务器 / x86 家用机 | `linux_amd64` |
| 树莓派 4/5          | `linux_arm64` |
| 旧树莓派 / 部分 NAS    | `linux_arm`   |

---

## 3. 云服务器：安装与配置 frps

### 3.1 安装

```bash
sudo mkdir -p /opt/frp
sudo cp frps /opt/frp/
sudo chmod +x /opt/frp/frps
```

### 3.2 配置文件

创建 `/opt/frp/frps.toml`：

```toml
# frps.toml — 云服务器（服务端）

# frpc 连接端口
bindPort = 7000

# 身份认证（务必修改 token！）
auth.method = "token"
auth.token = "fmy8him+7NQ1zGOWPMBWWWK9gLVp5JLcj9TFUjlzf1I="

# HTTP/HTTPS 域名穿透（不用可注释）
# vhostHTTPPort = 80
# vhostHTTPSPort = 443
# subDomainHost = "frp.example.com"   # 子域名后缀，配合泛解析 *.frp.example.com

# 管理面板（生产环境建议 bind 127.0.0.1，通过 SSH 隧道访问）
webServer.addr = "127.0.0.1"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "8128be5e40bf15b1e20312d9c1879455"

# 限制 frpc 可申请的远程端口范围（安全建议）
allowPorts = [
    { start = 6000, end = 6100 },
    { start = 10000, end = 10100 },
]

# 日志
log.to = "/var/log/frps.log"
log.level = "info"
log.maxDays = 7
```

校验配置：

```bash
/opt/frp/frps verify -c /opt/frp/frps.toml
```

### 3.3 systemd 开机自启

创建 `/etc/systemd/system/frps.service`：

```ini
[Unit]
Description=FRP Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Restart=on-failure
RestartSec=5
ExecStart=/opt/frp/frps -c /opt/frp/frps.toml
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frps
sudo systemctl status frps
```

### 3.4 云服务器防火墙

**ufw（Ubuntu/Debian）：**

```bash
sudo ufw allow 7000/tcp comment 'frp control'
sudo ufw allow 6000/tcp comment 'frp ssh tunnel'
# 若用 HTTP 穿透
# sudo ufw allow 80/tcp
# sudo ufw allow 443/tcp
```

**云厂商安全组：** 在控制台同样放行上述端口。

### 3.5 验证 frps

```bash
# 进程与端口
ss -tlnp | grep frps

# 日志
sudo tail -f /var/log/frps.log
```

管理面板（若已配置）：本机执行 SSH 隧道后访问：

```bash
ssh -L 7500:127.0.0.1:7500 user@云服务器IP
# 浏览器打开 http://127.0.0.1:7500
```

---

## 4. 家里服务器：安装与配置 frpc

### 4.1 安装

```bash
sudo mkdir -p /opt/frp
sudo cp frpc /opt/frp/
sudo chmod +x /opt/frpc/frpc
```

### 4.2 配置文件

创建 `/opt/frp/frpc.toml`（将 `云服务器公网IP` 和 `token` 换成实际值）：

```toml
# frpc.toml — 家里服务器（客户端）

serverAddr = "云服务器公网IP"
serverPort = 7000

auth.method = "token"
auth.token = "与 frps.toml 中完全一致"

# 日志
log.to = "/var/log/frpc.log"
log.level = "info"
log.maxDays = 7

# ── 代理规则 ──────────────────────────────

# 1) SSH 穿透：外网 ssh -p 6000 user@云服务器IP → 家里 22 端口
[[proxies]]
name = "home-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000

# 2) 家里 Web 服务（例：NAS 管理页 5000 端口）
[[proxies]]
name = "home-nas-web"
type = "tcp"
localIP = "192.168.1.100"    # NAS 局域网 IP，若 frpc 跑在 NAS 上则用 127.0.0.1
localPort = 5000
remotePort = 6001

# 3) HTTP 域名穿透（需 frps 开启 vhostHTTPPort + 域名解析）
# [[proxies]]
# name = "home-web"
# type = "http"
# localIP = "127.0.0.1"
# localPort = 8080
# customDomains = ["nas.frp.example.com"]

# 4) 同一台机器多个服务：继续添加 [[proxies]] 块即可
```

校验配置：

```bash
/opt/frp/frpc verify -c /opt/frp/frpc.toml
```

### 4.3 systemd 开机自启

创建 `/etc/systemd/system/frpc.service`：

```ini
[Unit]
Description=FRP Client
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Restart=on-failure
RestartSec=5
ExecStart=/opt/frp/frpc -c /opt/frp/frpc.toml
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frpc
sudo systemctl status frpc
```

### 4.4 验证 frpc

```bash
sudo systemctl status frpc
sudo tail -f /var/log/frpc.log
# 日志中应出现 login success、proxy start success
```

---

## 5. 使用示例

### 5.1 通过穿透访问家里 SSH

```bash
ssh -p 6000 家里用户名@云服务器公网IP
```

等价于：`ssh 家里用户名@家里服务器局域网IP`（经 frp 转发）。

### 5.2 访问家里 Web 服务

浏览器打开：`http://云服务器公网IP:6001`

### 5.3 HTTP 域名穿透（可选）

1. 域名 `*.frp.example.com` 泛解析到云服务器 IP
2. frps 开启 `vhostHTTPPort = 80`、`subDomainHost = "frp.example.com"`
3. frpc 配置 `type = "http"` + `customDomains`
4. 访问 `http://nas.frp.example.com`

---

## 6. 安全建议

| 项            | 建议                                           |
|--------------|----------------------------------------------|
| `auth.token` | 使用 32 位以上随机字符串，两端必须一致                        |
| 管理面板         | `webServer.addr` 设为 `127.0.0.1`，勿直接暴露公网      |
| 端口范围         | `allowPorts` 限制可映射端口，避免 frpc 占用敏感端口          |
| SSH          | 家里 SSH 禁用密码登录，仅用密钥；可考虑将 `remotePort` 改为非常用端口 |
| TLS          | v0.50+ 默认开启 TLS；自签场景可在两端显式配置 `transport.tls` |
| 更新           | 先升级 frps，再升级 frpc（官方混合版本兼容建议）                |

生成随机 token：

```bash
openssl rand -base64 32
```

---

## 7. Docker 部署（可选）

适合家里 NAS（群晖、威联通等）或不想污染宿主机环境时使用。

**云服务器 docker-compose.yml（frps）：**

```yaml
services:
  frps:
    image: snowdreamtech/frps:0.68.0
    container_name: frps
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./frps.toml:/etc/frp/frps.toml:ro
```

**家里服务器 docker-compose.yml（frpc）：**

```yaml
services:
  frpc:
    image: snowdreamtech/frpc:0.68.0
    container_name: frpc
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml:ro
```

`network_mode: host` 可避免部分 NAT 场景下的端口映射问题；配置文件内容与上文 TOML 相同。

---

## 8. 常见问题排查

| 现象                     | 可能原因                            | 处理                                                                             |
|------------------------|---------------------------------|--------------------------------------------------------------------------------|
| frpc 无法连接              | 云安全组/防火墙未放行 7000                | 检查 ufw、iptables、云厂商安全组                                                         |
| `authorization failed` | token 不一致                       | 核对 frps/frpc 的 `auth.token`                                                    |
| `port unavailable`     | `remotePort` 不在 `allowPorts` 范围 | 修改 frps `allowPorts` 或更换 `remotePort`                                          |
| 连接成功但无法访问服务            | `localIP`/`localPort` 错误        | 确认家里服务监听地址，本机用 `curl 127.0.0.1:端口` 测试                                          |
| frpc 频繁断开              | 网络不稳、NAT 超时                     | 在 frpc 增加 `transport.heartbeatInterval = 30`、`transport.heartbeatTimeout = 90` |
| 版本不兼容                  | frps/frpc 版本差距过大                | 两端升级到同一小版本，或参考 [Release 说明](https://github.com/fatedier/frp/releases) 兼容范围     |

**调试命令：**

```bash
# 服务端
sudo journalctl -u frps -f
/opt/frp/frps verify -c /opt/frp/frps.toml

# 客户端
sudo journalctl -u frpc -f
/opt/frp/frpc verify -c /opt/frp/frpc.toml

# 从家里测试能否连上云端 7000
nc -zv 云服务器公网IP 7000
```

---

## 9. 配置模板速查

### 最小可用 frps.toml

```toml
bindPort = 7000
auth.method = "token"
auth.token = "your-secret-token"
```

### 最小可用 frpc.toml

```toml
serverAddr = "1.2.3.4"
serverPort = 7000
auth.method = "token"
auth.token = "fmy8him+7NQ1zGOWPMBWWWK9gLVp5JLcj9TFUjlzf1I="

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
```

---

## 10. 部署检查清单

- [ ] 云服务器已安装 frps，systemd 已 enable
- [ ] 家里服务器已安装 frpc，systemd 已 enable
- [ ] 两端 `auth.token` 一致且足够复杂
- [ ] 云防火墙 + 安全组已放行 7000 及所用 `remotePort`
- [ ] `frps verify` / `frpc verify` 通过
- [ ] frpc 日志有 `login success`
- [ ] 外网可通过 `云服务器IP:remotePort` 访问家里服务
- [ ] 管理面板未直接暴露公网（或已改强密码 + 限制来源 IP）

---

## 附录：目录结构参考

**云服务器：**

```
/opt/frp/
├── frps
└── frps.toml
/etc/systemd/system/frps.service
/var/log/frps.log
```

**家里服务器：**

```
/opt/frp/
├── frpc
└── frpc.toml
/etc/systemd/system/frpc.service
/var/log/frpc.log
```
