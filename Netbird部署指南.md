# Netbird 部署指南（含 DERP 服务器配置）

## 1. 概述

Netbird 是一个开源的零信任网络解决方案，使用 WireGuard 协议创建安全的点对点连接。本指南将详细介绍如何部署 Netbird 服务器、配置 DERP 服务器以及安装和配置客户端。

## 2. 环境准备

### 2.1 系统要求

| 组件 | CPU | 内存 | 存储空间 | 推荐操作系统 |
|------|-----|------|----------|-------------|
| Netbird 服务器 | 2核+ | 4GB+ | 50GB+ | Ubuntu 22.04 LTS |
| DERP 服务器 | 1核+ | 2GB+ | 20GB+ | Ubuntu 22.04 LTS |
| 客户端 | 1核+ | 1GB+ | 10GB+ | 支持多种操作系统 |

### 2.2 网络要求

- 所有服务器之间网络互通
- 开放必要的端口：
  - Netbird 服务器：80/tcp（HTTP）、443/tcp（HTTPS）
  - DERP 服务器：443/tcp（HTTPS）、3478/udp（STUN）
- 确保 DNS 解析正常

## 3. 部署 Netbird 服务器

### 3.1 安装 Netbird 服务器

1. **更新系统**：
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **安装依赖**：
   ```bash
   sudo apt install -y curl git
   ```

3. **下载并运行安装脚本**：
   ```bash
   curl -fsSL https://github.com/netbirdio/netbird/releases/latest/download/install.sh | sudo bash
   ```

4. **初始化 Netbird 服务器**：
   ```bash
   sudo netbird up
   ```

5. **获取管理令牌**：
   ```bash
   sudo netbird mgmt status
   ```

### 3.2 配置 Netbird 服务器

1. **编辑配置文件**：
   ```bash
   sudo nano /etc/netbird/config.json
   ```

2. **基本配置**：
   ```json
   {
     "Hostname": "netbird.example.com",
     "DashboardURL": "https://netbird.example.com",
     "AuthOIDC": {
       "Provider": "keycloak",
       "ClientID": "netbird",
       "ClientSecret": "your-client-secret",
       "DiscoveryURL": "https://keycloak.example.com/realms/master",
       "RedirectURI": "https://netbird.example.com/oauth/callback"
     },
     "Datadir": "/var/lib/netbird",
     "Port": 443,
     "LetsEncryptEmail": "your-email@example.com"
   }
   ```

3. **重启 Netbird 服务器**：
   ```bash
   sudo systemctl restart netbird
   ```

## 4. 配置 DERP 服务器

### 4.1 安装 DERP 服务器

1. **下载 DERP 服务器**：
   ```bash
   wget https://github.com/netbirdio/derp/releases/latest/download/derp-server-linux-amd64.tar.gz
   tar -xzf derp-server-linux-amd64.tar.gz
   sudo mv derp-server /usr/local/bin/
   ```

2. **创建配置文件**：
   ```bash
   sudo mkdir -p /etc/derp
   sudo nano /etc/derp/config.json
   ```

3. **配置 DERP 服务器**：
   ```json
   {
     "Stun": {
       "ListenAddr": "0.0.0.0:3478"
     },
     "Derp": {
       "RegionID": "us-east",
       "RegionCode": "use1",
       "RegionName": "US East",
       "ListenAddr": "0.0.0.0:443",
       "TLSCert": "/etc/letsencrypt/live/derp.example.com/fullchain.pem",
       "TLSKey": "/etc/letsencrypt/live/derp.example.com/privkey.pem"
     }
   }
   ```

4. **创建系统服务**：
   ```bash
   sudo nano /etc/systemd/system/derp.service
   ```

5. **添加服务配置**：
   ```ini
   [Unit]
   Description=DERP Server
   After=network.target

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/derp-server --config=/etc/derp/config.json
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

6. **启动 DERP 服务器**：
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start derp
   sudo systemctl enable derp
   ```

### 4.2 配置 Netbird 使用自定义 DERP 服务器

1. **编辑 Netbird 配置文件**：
   ```bash
   sudo nano /etc/netbird/config.json
   ```

2. **添加 DERP 服务器配置**：
   ```json
   {
     "DERPServers": [
       {
         "Name": "us-east",
         "RegionID": "us-east",
         "Host": "derp.example.com",
         "Port": 443,
         "STUNPort": 3478,
         "Protocol": "https"
       }
     ]
   }
   ```

3. **重启 Netbird 服务器**：
   ```bash
   sudo systemctl restart netbird
   ```

## 5. 客户端安装和配置

### 5.1 Windows 客户端

1. **下载安装包**：
   - 访问 https://github.com/netbirdio/netbird/releases/latest
   - 下载 `netbird-windows-amd64.msi`

2. **安装客户端**：
   - 双击安装包并按照提示完成安装

3. **配置客户端**：
   - 启动 Netbird 客户端
   - 点击 "Sign in"
   - 输入 Netbird 服务器地址：`https://netbird.example.com`
   - 按照提示完成登录和配置

### 5.2 Linux 客户端

1. **下载并安装**：
   ```bash
   curl -fsSL https://github.com/netbirdio/netbird/releases/latest/download/install.sh | sudo bash
   ```

2. **配置客户端**：
   ```bash
   sudo netbird up --management-url https://netbird.example.com
   ```

3. **登录**：
   ```bash
   sudo netbird login
   ```
   按照提示完成登录。

### 5.3 macOS 客户端

1. **下载安装包**：
   - 访问 https://github.com/netbirdio/netbird/releases/latest
   - 下载 `netbird-macos-amd64.pkg`

2. **安装客户端**：
   - 双击安装包并按照提示完成安装

3. **配置客户端**：
   - 启动 Netbird 客户端
   - 点击 "Sign in"
   - 输入 Netbird 服务器地址：`https://netbird.example.com`
   - 按照提示完成登录和配置

## 6. 验证部署

### 6.1 检查服务器状态

```bash
# 检查 Netbird 服务器状态
sudo netbird status

# 检查 DERP 服务器状态
sudo systemctl status derp
```

### 6.2 检查客户端连接

```bash
# 查看客户端状态
netbird status

# 查看 peers
netbird peers
```

### 6.3 测试连接

1. **在一个客户端上 ping 另一个客户端**：
   ```bash
   ping <peer-ip>
   ```

2. **测试 SSH 连接**：
   ```bash
   ssh user@<peer-ip>
   ```

## 7. 高级配置

### 7.1 配置 OIDC 认证

Netbird 支持多种 OIDC 提供商，如 Keycloak、Auth0、Google 等。以下是使用 Keycloak 的配置示例：

1. **在 Keycloak 中创建客户端**：
   - 登录 Keycloak 管理界面
   - 创建一个新的客户端，客户端 ID 为 "netbird"
   - 设置 Redirect URI 为 "https://netbird.example.com/oauth/callback"
   - 启用 "Standard Flow" 和 "Direct Access Grants"

2. **更新 Netbird 配置**：
   ```json
   {
     "AuthOIDC": {
       "Provider": "keycloak",
       "ClientID": "netbird",
       "ClientSecret": "your-client-secret",
       "DiscoveryURL": "https://keycloak.example.com/realms/master",
       "RedirectURI": "https://netbird.example.com/oauth/callback"
     }
   }
   ```

### 7.2 配置多区域 DERP 服务器

为了提高全球访问速度，可以配置多个区域的 DERP 服务器：

```json
{
  "DERPServers": [
    {
      "Name": "us-east",
      "RegionID": "us-east",
      "Host": "derp-us.example.com",
      "Port": 443,
      "STUNPort": 3478,
      "Protocol": "https"
    },
    {
      "Name": "eu-west",
      "RegionID": "eu-west",
      "Host": "derp-eu.example.com",
      "Port": 443,
      "STUNPort": 3478,
      "Protocol": "https"
    },
    {
      "Name": "asia-east",
      "RegionID": "asia-east",
      "Host": "derp-asia.example.com",
      "Port": 443,
      "STUNPort": 3478,
      "Protocol": "https"
    }
  ]
}
```

### 7.3 配置访问控制

Netbird 支持基于用户和组的访问控制：

1. **创建访问控制规则**：
   ```bash
   sudo netbird access add --name "Allow Engineering" --users "user1@example.com,user2@example.com" --peers "*"
   ```

2. **查看访问控制规则**：
   ```bash
   sudo netbird access list
   ```

## 8. 故障排除

### 8.1 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 客户端无法连接 | 网络问题 | 检查网络连接和防火墙规则 |
| DERP 服务器无法访问 | 端口未开放 | 检查防火墙规则，确保 443 端口开放 |
| 认证失败 | OIDC 配置错误 | 检查 OIDC 提供商配置和 Netbird 配置 |
| 点对点连接失败 | NAT 穿透问题 | 检查 DERP 服务器配置和网络环境 |

### 8.2 查看日志

```bash
# Netbird 服务器日志
sudo journalctl -u netbird

# DERP 服务器日志
sudo journalctl -u derp

# 客户端日志
netbird logs
```

## 9. 维护

### 9.1 备份

```bash
# 备份 Netbird 配置和数据
sudo tar -czf netbird-backup.tar.gz /etc/netbird /var/lib/netbird

# 备份 DERP 配置
sudo tar -czf derp-backup.tar.gz /etc/derp
```

### 9.2 更新

```bash
# 更新 Netbird 服务器
sudo netbird update

# 更新 DERP 服务器
wget https://github.com/netbirdio/derp/releases/latest/download/derp-server-linux-amd64.tar.gz
tar -xzf derp-server-linux-amd64.tar.gz
sudo systemctl stop derp
sudo mv derp-server /usr/local/bin/
sudo systemctl start derp
```

### 9.3 重启服务

```bash
# 重启 Netbird 服务器
sudo systemctl restart netbird

# 重启 DERP 服务器
sudo systemctl restart derp
```

## 10. 总结

本指南详细介绍了如何部署 Netbird 服务器、配置 DERP 服务器以及安装和配置客户端。通过以下步骤，您可以构建一个安全、可靠的零信任网络：

1. 准备系统环境
2. 部署 Netbird 服务器
3. 配置 DERP 服务器
4. 安装和配置客户端
5. 验证部署
6. 配置高级功能

Netbird 提供了一种简单、安全的方式来构建零信任网络，适用于企业内部网络、远程办公和 IoT 设备管理等场景。通过合理配置 DERP 服务器，可以提高网络连接的可靠性和性能。