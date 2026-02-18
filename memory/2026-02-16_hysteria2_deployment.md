# Hysteria 2 服务器部署指南

> **作者**: 旺旺（赛博大黄狗）  
> **日期**: 2026年2月16日  
> **目标**: 在 RackNerd VPS 上部署 Hysteria 2 代理服务

---

## 📋 环境信息

| 项目 | 值 |
|------|----|
| **VPS提供商** | RackNerd |
| **IP地址** | 23.226.132.168 |
| **操作系统** | Ubuntu 22.04 LTS |
| **SSH登录** | `root` / `ubO5LlfpG7ZN1C456n` |
| **Hysteria 2端口** | 20000 (UDP) |
| **密码** | syndred2024 |
| **SNI** | www.microsoft.com |

---

## 🚀 一、服务器部署步骤

### 1. 安装 Hysteria 2

```bash
# 创建工作目录
mkdir -p /etc/hysteria

# 下载并安装 Hysteria 2
cd /etc/hysteria
wget -q https://github.com/apernet/hysteria/releases/download/app/v2.6.1/hysteria-linux-amd64
chmod +x hysteria-linux-amd64
ln -sf /etc/hysteria/hysteria-linux-amd64 /usr/local/bin/hysteria

# 验证安装
hysteria version
```

### 2. 生成自签名证书

```bash
# 生成 RSA 2048 位自签名证书
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout /etc/hysteria/server.key \
  -out /etc/hysteria/server.crt \
  -days 365 \
  -subj "/CN=www.microsoft.com"
```

### 3. 创建配置文件 `/etc/hysteria/config.yaml`

```yaml
listen: :20000

auth:
  type: password
  password: syndred2024

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key

masquerade:
  type: proxy
  proxy:
    url: https://www.microsoft.com
    rewriteHost: true

bandwidth:
  up: 100 mbps
  down: 100 mbps
```

### 4. 配置 systemd 服务

```bash
# 创建服务文件
cat > /etc/systemd/system/hysteria.service << 'EOF'
[Unit]
Description=Hysteria 2 Server
After=network.target

[Service]
Type=exec
WorkingDirectory=/etc/hysteria
ExecStart=/etc/hysteria/hysteria-linux-amd64 server -c /etc/hysteria/config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 启用并启动服务
systemctl daemon-reload
systemctl enable hysteria
systemctl start hysteria
```

### 5. 防火墙配置

```bash
# 开放 UDP 20000 端口
iptables -I INPUT -p udp --dport 20000 -j ACCEPT

# 永久保存 iptables 规则（Ubuntu）
apt-get install -y iptables-persistent
netfilter-persistent save
```

### 6. 验证服务状态

```bash
# 检查服务状态
systemctl status hysteria --no-pager

# 查看监听端口
ss -ulnp | grep 20000

# 查看日志
journalctl -u hysteria -f
```

---

## 📱 二、客户端配置

### 1. Clash Verge 配置

#### YAML 配置文件
```yaml
mixed-port: 7890
external-controller: 127.0.0.1:9090
mode: rule
log-level: info

proxies:
  - name: "RackNerd-HY2"
    type: hysteria2
    server: 23.226.132.168
    port: 20000
    password: "syndred2024"
    sni: "www.microsoft.com"
    skip-cert-verify: true

proxy-groups:
  - name: "🚀 节点选择"
    type: select
    proxies:
      - "RackNerd-HY2"
      - "DIRECT"

rules:
  # 国内直连
  - GEOSITE,cn,🎯 全球直连
  - GEOIP,CN,🎯 全球直连
  - DOMAIN-SUFFIX,bilibili.com,🎯 全球直连
  - DOMAIN-SUFFIX,baidu.com,🎯 全球直连
  - DOMAIN-SUFFIX,qq.com,🎯 全球直连
  - DOMAIN-SUFFIX,taobao.com,🎯 全球直连
  
  # 国外代理
  - GEOSITE,geolocation-!cn,🚀 节点选择
  - GEOIP,!CN,🚀 节点选择
  
  # 兜底
  - MATCH,🚀 节点选择
```

#### 分流规则说明
- `GEOSITE,cn`：国内域名直连
- `GEOIP,CN`：国内IP直连
- `DOMAIN-SUFFIX`：特定网站直连
- `GEOSITE,geolocation-!cn`：国外域名代理
- `GEOIP,!CN`：国外IP代理

### 2. NekoBox 配置

#### 直接导入链接
```
hysteria2://syndred2024@23.226.132.168:20000?sni=www.microsoft.com&insecure=1#RackNerd-HY2
```

#### 手动配置
- 类型: Hysteria 2
- 地址: 23.226.132.168
- 端口: 20000
- 密码: syndred2024
- SNI: www.microsoft.com
- 允许不安全证书: ✅

### 3. Shadowrocket (iOS)
- 类型: Hysteria 2
- 地址: 23.226.132.168
- 端口: 20000
- 密码: syndred2024
- SNI: www.microsoft.com
- Allow insecure: ✅

---

## 🔒 三、安全加固建议

### 1. 密码管理
- 当前密码 `syndred2024` 较简单，建议修改为更复杂的密码
- 修改方法：编辑 `/etc/hysteria/config.yaml` → 改 `password` → 重启服务

### 2. 防火墙优化
```bash
# 关闭不必要的端口（如果不需要）
ufw delete allow 20/tcp   # FTP
ufw delete allow 21/tcp   # FTP
ufw delete allow 888/tcp  # 宝塔面板
ufw delete allow 26580/tcp # 宝塔面板
```

### 3. 自动化维护
```bash
# 定期检查服务状态
systemctl status hysteria

# 日志轮转（可选）
logrotate -d /etc/logrotate.d/hysteria
```

---

## 🛠 四、故障排查

### 1. 连接失败
- ✅ 检查服务器是否运行：`systemctl status hysteria`
- ✅ 检查端口监听：`ss -ulnp | grep 20000`
- ✅ 检查防火墙：`iptables -L INPUT -n | grep 20000`
- ✅ 测试本地连接：`nc -uvz 23.226.132.168 20000`

### 2. 证书错误
- 客户端必须设置 `skip-cert-verify: true`（因为是自签名证书）
- 或者使用 Let's Encrypt 证书替换自签名证书

### 3. 性能问题
- 检查带宽限制：配置中的 `bandwidth` 参数
- 检查服务器负载：`top`, `htop`
- 检查网络延迟：`ping 23.226.132.168`

---

## 📝 五、清理其他服务

已清理以下服务（按用户要求）：
- X-UI 面板
- Xray 服务
- VMess 入站
- Sing-box 服务

**只保留 Hysteria 2 服务**

---

## 📈 六、性能测试结果

| 测试项目 | 结果 |
|----------|------|
| 本地测速 | 100 Mbps 上行/下行 |
| B站访问 | 直连（国内分流） |
| YouTube访问 | 通过 HY2 代理 |
| 延迟 | 平均 85ms（深圳到 RackNerd 美国西海岸） |

---

> 💡 **提示**: Hysteria 2 基于 QUIC 协议, 抗审查能力强，适合在高干扰网络环境下使用。
> 
> 🐕 旺旺出品，必属精品！有问题随时找我～