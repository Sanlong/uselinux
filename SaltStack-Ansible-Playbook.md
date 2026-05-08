# SaltStack Ansible Playbook

本 Ansible Playbook 用于自动化部署 Salt-Master 和 Salt-Minion。

## 目录结构

```
├── salt-master.ansible.yaml     # 主 Playbook 文件
├── salt-master.conf.j2          # Salt-Master 配置模板
├── salt-minion.conf.j2          # Salt-Minion 配置模板
├── hosts-salt.yml              # Inventory 示例文件
└── SaltStack-Ansible-Playbook.md  # 本文档
```

## 功能特性

- 支持 Rocky Linux/AlmaLinux 系统
- 支持 Ubuntu/Debian 系统
- 自动安装必要的依赖和仓库
- 配置 Salt-Master 和 Salt-Minion
- 启动并启用相关服务

## 使用方法

### 1. 配置 Inventory

复制 `hosts-salt.yml` 示例文件并修改为实际环境：

```bash
cp hosts-salt.yml hosts.yml
```

编辑 `hosts.yml` 文件，设置实际的主机信息：

```yaml
all:
  children:
    salt-master:
      hosts:
        salt-master.example.com:
          ansible_host: 192.168.1.10  # 实际的 Salt-Master IP
          ansible_user: root         # 登录用户
          ansible_ssh_pass: your_password  # 登录密码（或使用密钥）
    salt-minion:
      hosts:
        minion1.example.com:
          ansible_host: 192.168.1.11  # 实际的 Minion IP
          ansible_user: root
          ansible_ssh_pass: your_password
```

### 2. 运行 Playbook

#### 部署 Salt-Master：

```bash
ansible-playbook -i hosts.yml salt-master.ansible.yaml --limit salt-master
```

#### 部署 Salt-Minion：

```bash
ansible-playbook -i hosts.yml salt-master.ansible.yaml --limit salt-minion
```

#### 部署所有组件：

```bash
ansible-playbook -i hosts.yml salt-master.ansible.yaml
```

## 变量说明

### Salt-Master 变量：

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| salt_master_interface | 0.0.0.0 | 监听地址 |
| salt_master_publish_port | 4505 | 发布端口 |
| salt_master_ret_port | 4506 | 请求响应端口 |
| salt_master_log_level | info | 日志级别 |
| salt_master_auto_accept | false | 是否自动接受密钥（生产环境建议 false） |
| salt_master_file_roots | base: [/srv/salt] | 工作目录 |
| salt_master_pillar_roots | base: [/srv/pillar] | Pillar 目录 |

### Salt-Minion 变量：

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| salt_master_address | salt-master | Salt-Master 地址 |
| salt_minion_log_level | info | 日志级别 |

## 部署后操作

### 1. 接受 Minion 密钥

在 Salt-Master 上执行：

```bash
# 查看待接受的密钥
salt-key -L

# 接受单个密钥
salt-key -a minion1

# 接受所有密钥（生产环境谨慎使用）
salt-key -A -y
```

### 2. 验证连接

```bash
# 测试所有 Minion
salt '*' test.ping

# 测试单个 Minion
salt minion1 test.ping

# 查看 Minion 信息
salt minion1 grains.items
```

## 高可用配置

如需配置 Salt-Master 高可用，请参考 [SaltStack部署指南.md](SaltStack部署指南.md) 中的高可用方案部分。

## 注意事项

1. 确保目标服务器可以正常访问互联网以下载安装包
2. 生产环境建议使用 SSH 密钥认证，避免在配置文件中存储密码
3. 防火墙需要开放 4505/tcp 和 4506/tcp 端口
4. 建议在测试环境验证后再部署到生产环境

## 故障排除

### 常见问题：

1. **Minion 无法连接 Master**：检查网络连接和防火墙规则
2. **密钥认证失败**：删除并重新接受密钥
3. **服务启动失败**：查看日志文件 `/var/log/salt/master` 或 `/var/log/salt/minion`

### 查看日志：

```bash
# Master 日志
tail -f /var/log/salt/master

# Minion 日志
tail -f /var/log/salt/minion
```
