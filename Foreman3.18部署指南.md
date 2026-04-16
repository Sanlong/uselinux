# Foreman 3.18 部署指南

## 1. 系统要求

### 1.1 硬件要求

| 组件 | 最小值 | 推荐值 |
|------|-------|-------|
| CPU | 2 核 | 4 核+ |
| 内存 | 4GB | 8GB+ |
| 存储空间 | 50GB | 100GB+ |
| 网络 | 千兆网卡 | 千兆网卡 |

### 1.2 操作系统要求

- **RHEL/CentOS Stream 9**
- **Rocky Linux 9**
- **AlmaLinux 9**
- **Ubuntu 22.04 LTS**

## 2. 安装前准备

### 2.1 系统更新

```bash
# RHEL/Rocky Linux/AlmaLinux
sudo dnf update -y

# Ubuntu
sudo apt update && sudo apt upgrade -y
```

### 2.2 配置主机名

```bash
# 设置主机名
sudo hostnamectl set-hostname foreman.example.com

# 配置 /etc/hosts
sudo nano /etc/hosts
```

添加以下内容：
```
192.168.1.10 foreman.example.com foreman
```

### 2.3 关闭防火墙（可选）

```bash
# RHEL/Rocky Linux/AlmaLinux
sudo systemctl stop firewalld
sudo systemctl disable firewalld

# Ubuntu
sudo ufw disable
```

## 3. 安装 Foreman 3.18

### 3.1 在 RHEL/Rocky Linux/AlmaLinux 上安装

1. **添加 Foreman 仓库**：
   ```bash
   sudo dnf install -y https://yum.theforeman.org/releases/3.18/el9/x86_64/foreman-release.rpm
   sudo dnf install -y epel-release
   sudo dnf makecache
   ```

2. **安装 Foreman 安装器**：
   ```bash
   sudo dnf install -y foreman-installer
   ```

3. **运行安装器**：
   ```bash
   sudo foreman-installer \
     --enable-foreman-plugin-ansible \
     --enable-foreman-plugin-remote-execution \
     --foreman-initial-admin-password=Mao3long
   ```

### 3.2 在 Ubuntu 22.04 上安装

1. **添加 Foreman 仓库**：
   ```bash
   echo "deb http://deb.theforeman.org/ jammy 3.18" | sudo tee /etc/apt/sources.list.d/foreman.list
   echo "deb http://deb.theforeman.org/ plugins 3.18" | sudo tee -a /etc/apt/sources.list.d/foreman.list
   wget -q https://deb.theforeman.org/pubkey.gpg -O- | sudo apt-key add -
   sudo apt update
   ```

2. **安装 Foreman 安装器**：
   ```bash
   sudo apt install -y foreman-installer
   ```

3. **运行安装器**：
   ```bash
   sudo foreman-installer \
     --enable-foreman-plugin-ansible \
     --enable-foreman-plugin-remote-execution \
     --foreman-initial-admin-password=Mao3long
   ```

## 4. 验证安装

### 4.1 检查服务状态

```bash
# 检查 Foreman 服务
sudo systemctl status foreman

# 检查 Apache 服务
sudo systemctl status httpd  # RHEL/Rocky Linux/AlmaLinux
# 或
sudo systemctl status apache2  # Ubuntu

# 检查 PostgreSQL 服务
sudo systemctl status postgresql
```

### 4.2 访问 Foreman Web 界面

打开浏览器，访问：`https://foreman.example.com`

使用以下凭据登录：
- **用户名**：admin
- **密码**：Mao3long

### 4.3 验证插件安装

1. 登录 Foreman Web 界面
2. 点击左侧菜单中的 "Administer" → "About"
3. 检查 "Plugins" 部分，确认 Ansible 和 Remote Execution 插件已安装

## 5. 配置插件

### 5.1 配置 Ansible 插件

1. **生成 Ansible 配置**：
   ```bash
   sudo foreman-rake foreman_ansible:generate_ansible_cfg
   ```

2. **同步 Ansible 角色**：
   ```bash
   sudo foreman-rake foreman_ansible:roles:sync
   ```

3. **在 Web 界面中配置**：
   - 点击左侧菜单中的 "Configure" → "Ansible"
   - 配置 Ansible 相关设置

### 5.2 配置 Remote Execution 插件

1. **在 Web 界面中配置**：
   - 点击左侧菜单中的 "Administer" → "Settings" → "Remote Execution"
   - 配置远程执行相关设置

2. **设置 SSH 密钥**：
   - 点击左侧菜单中的 "Hosts" → "All Hosts"
   - 选择主机，点击 "Edit"
   - 在 "Remote Execution" 标签页中配置 SSH 密钥

## 6. 基本使用

### 6.1 添加主机

1. 点击左侧菜单中的 "Hosts" → "Create Host"
2. 填写主机信息：
   - **Name**：主机名
   - **Organization**：选择组织
   - **Location**：选择位置
   - **Host Group**：选择主机组
3. 在 "Network" 标签页中配置网络信息
4. 在 "Operating System" 标签页中选择操作系统
5. 点击 "Submit" 创建主机

### 6.2 使用 Ansible 管理主机

1. **为主机分配 Ansible 角色**：
   - 点击左侧菜单中的 "Hosts" → "All Hosts"
   - 选择主机，点击 "Edit"
   - 在 "Ansible Roles" 标签页中分配角色

2. **运行 Ansible 播放**：
   - 点击左侧菜单中的 "Hosts" → "All Hosts"
   - 选择主机，点击 "Run Ansible Playbook"
   - 选择要运行的播放簿

### 6.3 使用 Remote Execution 执行命令

1. **执行远程命令**：
   - 点击左侧菜单中的 "Hosts" → "All Hosts"
   - 选择主机，点击 "Run Command"
   - 输入要执行的命令
   - 选择执行方式（SSH）
   - 点击 "Submit"

## 7. 故障排除

### 7.1 查看日志

```bash
# Foreman 日志
sudo tail -f /var/log/foreman/production.log

# Apache 日志
sudo tail -f /var/log/httpd/error_log  # RHEL/Rocky Linux/AlmaLinux
# 或
sudo tail -f /var/log/apache2/error.log  # Ubuntu

# PostgreSQL 日志
sudo tail -f /var/lib/pgsql/data/log/postgresql-*.log  # RHEL/Rocky Linux/AlmaLinux
# 或
sudo tail -f /var/log/postgresql/postgresql-*.log  # Ubuntu
```

### 7.2 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 无法访问 Web 界面 | 防火墙阻止 | 关闭防火墙或开放 443 端口 |
| 插件安装失败 | 依赖问题 | 检查依赖并重新安装 |
| 远程执行失败 | SSH 配置错误 | 检查 SSH 密钥和权限 |
| Ansible 同步失败 | 网络问题 | 检查网络连接和 Ansible 配置 |

## 8. 维护

### 8.1 备份

```bash
# 备份 Foreman 数据库
sudo foreman-rake db:dump

# 备份配置文件
sudo tar -czf foreman-backup.tar.gz /etc/foreman /var/lib/foreman
```

### 8.2 更新

```bash
# RHEL/Rocky Linux/AlmaLinux
sudo dnf update foreman*

# Ubuntu
sudo apt update && sudo apt upgrade foreman*
```

### 8.3 重启服务

```bash
sudo systemctl restart foreman httpd postgresql
```

## 9. 总结

本指南详细介绍了如何部署 Foreman 3.18 并启用 Ansible 和 Remote Execution 插件。通过以下步骤，您可以构建一个功能强大的基础设施管理系统：

1. 准备系统环境
2. 安装 Foreman 3.18
3. 启用所需插件
4. 配置插件设置
5. 开始使用 Foreman 管理基础设施

Foreman 3.18 结合 Ansible 和 Remote Execution 插件，为您提供了一个完整的基础设施管理解决方案，能够自动化配置管理、远程执行命令和管理主机生命周期。