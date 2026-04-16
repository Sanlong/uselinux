# SaltStack 部署指南（含高可用方案）

## 1. 概述

SaltStack 是一个强大的自动化配置管理和远程执行系统，用于管理服务器基础设施。本指南将详细介绍 SaltStack 的部署过程，包括 Salt-Master（服务器）和 Salt-Minion（客户端）的部署，以及高可用方案。

## 2. 环境准备

### 2.1 系统要求

| 组件 | CPU | 内存 | 存储空间 | 推荐操作系统 |
|------|-----|------|----------|-------------|
| Salt-Master | 2核+ | 4GB+ | 50GB+ | Rocky Linux 9 或 Ubuntu 22.04 LTS |
| Salt-Minion | 1核+ | 512MB+ | 10GB+ | 任意支持的 Linux 发行版 |

### 2.2 网络要求

- 所有服务器之间网络互通
- 开放必要的端口：
  - Salt-Master：4505/tcp（发布端口）、4506/tcp（请求响应端口）
- 确保 DNS 解析正常或配置 /etc/hosts 文件

### 2.3 环境规划示例

```
Salt-Master: 192.168.1.10 (hostname: salt-master)
Salt-Minion1: 192.168.1.11 (hostname: minion1)
Salt-Minion2: 192.168.1.12 (hostname: minion2)
```

配置 /etc/hosts 文件：
```
192.168.1.10 salt-master salt
192.168.1.11 minion1
192.168.1.12 minion2
```

## 3. 部署 Salt-Master

### 3.1 在 Rocky Linux/AlmaLinux 上部署

```bash
# 安装 EPEL 仓库
sudo dnf install epel-release -y

# 导入 SaltStack GPG 密钥
sudo rpm --import https://repo.saltproject.io/py3/redhat/9/x86_64/latest/SALTSTACK-GPG-KEY.pub

# 添加 SaltStack 仓库
curl -fsSL https://repo.saltproject.io/py3/redhat/9/x86_64/latest.repo | sudo tee /etc/yum.repos.d/salt.repo

# 安装 Salt-Master
sudo dnf install salt-master -y

# 启动并启用 Salt-Master 服务
sudo systemctl start salt-master
sudo systemctl enable salt-master

# 检查服务状态
sudo systemctl status salt-master
```

### 3.2 在 Ubuntu/Debian 上部署

```bash
# 更新包列表
sudo apt update

# 安装依赖
sudo apt install curl gnupg -y

# 导入 SaltStack GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://repo.saltproject.io/py3/ubuntu/22.04/amd64/latest/SALTSTACK-GPG-KEY.pub | sudo gpg --dearmor -o /etc/apt/keyrings/salt-archive-keyring.gpg

# 添加 SaltStack 仓库
echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring.gpg arch=amd64] https://repo.saltproject.io/py3/ubuntu/22.04/amd64/latest jammy main" | sudo tee /etc/apt/sources.list.d/salt.list

# 更新包列表并安装 Salt-Master
sudo apt update
sudo apt install salt-master -y

# 启动并启用 Salt-Master 服务
sudo systemctl start salt-master
sudo systemctl enable salt-master

# 检查服务状态
sudo systemctl status salt-master
```

### 3.3 配置 Salt-Master

编辑配置文件：
```bash
sudo nano /etc/salt/master
```

主要配置项：
```yaml
# 监听地址
interface: 0.0.0.0

# 发布端口
publish_port: 4505

# 请求响应端口
ret_port: 4506

# 工作目录
file_roots:
  base:
    - /srv/salt

# Pillar 目录
pillar_roots:
  base:
    - /srv/pillar

# 日志级别
log_level: info

# 自动接受密钥（生产环境不建议）
# auto_accept: True
```

创建工作目录：
```bash
sudo mkdir -p /srv/salt /srv/pillar
sudo chown salt:salt /srv/salt /srv/pillar
```

重启 Salt-Master 服务：
```bash
sudo systemctl restart salt-master
```

## 4. 部署 Salt-Minion

### 4.1 在 Rocky Linux/AlmaLinux 上部署

```bash
# 安装 EPEL 仓库
sudo dnf install epel-release -y

# 导入 SaltStack GPG 密钥
sudo rpm --import https://repo.saltproject.io/py3/redhat/9/x86_64/latest/SALTSTACK-GPG-KEY.pub

# 添加 SaltStack 仓库
curl -fsSL https://repo.saltproject.io/py3/redhat/9/x86_64/latest.repo | sudo tee /etc/yum.repos.d/salt.repo

# 安装 Salt-Minion
sudo dnf install salt-minion -y
```

### 4.2 在 Ubuntu/Debian 上部署

```bash
# 更新包列表
sudo apt update

# 安装依赖
sudo apt install curl gnupg -y

# 导入 SaltStack GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://repo.saltproject.io/py3/ubuntu/22.04/amd64/latest/SALTSTACK-GPG-KEY.pub | sudo gpg --dearmor -o /etc/apt/keyrings/salt-archive-keyring.gpg

# 添加 SaltStack 仓库
echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring.gpg arch=amd64] https://repo.saltproject.io/py3/ubuntu/22.04/amd64/latest jammy main" | sudo tee /etc/apt/sources.list.d/salt.list

# 更新包列表并安装 Salt-Minion
sudo apt update
sudo apt install salt-minion -y
```

### 4.3 配置 Salt-Minion

编辑配置文件：
```bash
sudo nano /etc/salt/minion
```

主要配置项：
```yaml
# Master 地址
master: salt-master

# Minion ID（默认使用主机名）
# id: minion1

# 日志级别
log_level: info
```

启动并启用 Salt-Minion 服务：
```bash
sudo systemctl start salt-minion
sudo systemctl enable salt-minion

# 检查服务状态
sudo systemctl status salt-minion
```

## 5. 密钥管理和验证

### 5.1 在 Salt-Master 上接受密钥

查看待接受的密钥：
```bash
sudo salt-key -L
```

接受单个密钥：
```bash
sudo salt-key -a minion1
```

接受所有密钥（生产环境谨慎使用）：
```bash
sudo salt-key -A -y
```

删除密钥：
```bash
sudo salt-key -d minion1
```

### 5.2 验证连接

在 Salt-Master 上测试连接：
```bash
# 测试所有 Minion
sudo salt '*' test.ping

# 测试单个 Minion
sudo salt minion1 test.ping

# 查看 Minion 信息
sudo salt minion1 grains.items
```

## 6. Salt-Master 高可用方案

### 6.1 方案一：基于同步配置的高可用

架构设计：
```
                     +-----------------+
                     |  负载均衡器      |
                     +-----------------+
                              |
              +---------------+---------------+
              |                               |
    +------------------+            +------------------+
    |  Salt-Master-1   |            |  Salt-Master-2   |
    +------------------+            +------------------+
              |                               |
              +---------------+---------------+
                              |
                     +-----------------+
                     |   共享存储       |
                     |   (State/Pillar) |
                     +-----------------+
```

部署步骤：
1. 部署至少两个 Salt-Master 节点
2. 配置共享存储（NFS、GlusterFS 或云存储）
3. 使用 rsync 或 Git 同步配置文件
4. 使用负载均衡器分发请求

### 6.2 方案二：基于共享存储的高可用

配置步骤：
```bash
# 在两个 Master 节点上配置共享存储
# 例如使用 NFS 共享 /srv/salt 和 /srv/pillar

# 配置同步脚本
sudo nano /usr/local/bin/sync-salt-config.sh
```

添加以下内容：
```bash
#!/bin/bash
# 同步 State 文件
rsync -avz /srv/salt/ user@salt-master-2:/srv/salt/
# 同步 Pillar 文件
rsync -avz /srv/pillar/ user@salt-master-2:/srv/pillar/
# 同步 Master 配置
rsync -avz /etc/salt/master user@salt-master-2:/etc/salt/
```

设置定时同步：
```bash
sudo chmod +x /usr/local/bin/sync-salt-config.sh
sudo crontab -e
```

添加：
```
*/5 * * * * /usr/local/bin/sync-salt-config.sh
```

### 6.3 配置 HAProxy 负载均衡

安装 HAProxy：
```bash
sudo dnf install haproxy -y  # Rocky Linux
# 或
sudo apt install haproxy -y  # Ubuntu
```

配置 HAProxy：
```bash
sudo nano /etc/haproxy/haproxy.cfg
```

添加以下内容：
```
frontend salt_frontend
    bind *:4505
    bind *:4506
    mode tcp
    default_backend salt_backend

backend salt_backend
    mode tcp
    balance roundrobin
    server salt-master-1 192.168.1.10:4505 check
    server salt-master-1 192.168.1.10:4506 check
    server salt-master-2 192.168.1.20:4505 check
    server salt-master-2 192.168.1.20:4506 check
```

启动 HAProxy：
```bash
sudo systemctl start haproxy
sudo systemctl enable haproxy
```

## 7. 基本使用示例

### 7.1 远程执行命令

```bash
# 在所有 Minion 上执行命令
sudo salt '*' cmd.run 'uptime'

# 在单个 Minion 上执行命令
sudo salt minion1 cmd.run 'df -h'

# 安装软件包
sudo salt 'minion*' pkg.install nginx

# 管理服务
sudo salt minion1 service.start nginx
sudo salt minion1 service.enable nginx
```

### 7.2 创建简单的 State 文件

创建目录结构：
```bash
sudo mkdir -p /srv/salt/nginx
```

创建 State 文件：
```bash
sudo nano /srv/salt/nginx/init.sls
```

添加以下内容：
```yaml
nginx:
  pkg.installed: []
  service.running:
    - enable: True
    - require:
      - pkg: nginx

/etc/nginx/nginx.conf:
  file.managed:
    - source: salt://nginx/nginx.conf
    - require:
      - pkg: nginx
    - watch_in:
      - service: nginx
```

应用 State：
```bash
sudo salt minion1 state.apply nginx
```

## 8. 最佳实践

1. **版本控制**：将 State 和 Pillar 文件纳入版本控制
2. **测试环境**：先在测试环境验证配置
3. **安全配置**：禁用 auto_accept，手动管理密钥
4. **监控告警**：监控 Salt-Master 服务状态
5. **定期备份**：备份配置文件和密钥
6. **日志管理**：配置日志轮转和收集

## 9. 故障排除

### 常见问题

| 问题 | 可能原因 | 解决方法 |
|------|---------|---------|
| Minion 无法连接 Master | 网络问题或防火墙 | 检查网络连接和防火墙规则 |
| 密钥认证失败 | 密钥不匹配 | 删除并重新接受密钥 |
| State 执行失败 | 配置错误 | 检查 State 文件语法 |
| Master 性能问题 | 节点过多 | 考虑高可用方案 |

### 查看日志

```bash
# Master 日志
sudo tail -f /var/log/salt/master

# Minion 日志
sudo tail -f /var/log/salt/minion
```

## 10. 总结

本指南详细介绍了 SaltStack 的部署过程，包括：
- Salt-Master 和 Salt-Minion 的安装配置
- 密钥管理和连接验证
- 两种高可用方案
- 基本使用示例
- 最佳实践和故障排除

通过本指南，您可以快速部署一个功能完善的 SaltStack 自动化管理系统。