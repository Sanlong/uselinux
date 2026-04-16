# Prometheus 部署指南（含 Grafana 对接）

## 1. 概述

Prometheus 是一个开源的监控系统，用于收集和存储时间序列数据。Grafana 是一个可视化平台，用于展示 Prometheus 收集的数据。本指南将详细介绍如何部署 Prometheus 并与 Grafana 对接。

## 2. 环境准备

### 2.1 服务器要求

| 组件 | CPU | 内存 | 存储空间 | 网络 |
|------|-----|------|----------|------|
| Prometheus | 2核+ | 4GB+ | 50GB+ | 千兆网卡 |
| Grafana | 2核+ | 2GB+ | 20GB+ | 千兆网卡 |
| 节点导出器 | 1核 | 512MB+ | 10GB+ | 千兆网卡 |

### 2.2 操作系统要求

- CentOS 7/8/9 或 Rocky Linux 8/9
- Ubuntu 18.04 LTS+ 或 Debian 10+
- 系统已更新到最新版本

### 2.3 网络要求

- 所有服务器之间网络互通
- 开放必要的端口：
  - Prometheus：9090/tcp
  - Node Exporter：9100/tcp
  - Grafana：3000/tcp

## 3. 部署 Prometheus

### 3.1 下载并安装 Prometheus

```bash
# 创建 prometheus 用户
sudo useradd -M -s /bin/false prometheus

# 创建目录结构
sudo mkdir -p /etc/prometheus /var/lib/prometheus

# 下载 Prometheus
cd /tmp
sudo wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz

# 解压并安装
sudo tar xvf prometheus-2.45.0.linux-amd64.tar.gz
sudo cp prometheus-2.45.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.45.0.linux-amd64/promtool /usr/local/bin/

# 设置权限
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

### 3.2 配置 Prometheus

创建 Prometheus 配置文件：

```bash
sudo nano /etc/prometheus/prometheus.yml
```

添加以下内容：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
```

### 3.3 创建系统服务

```bash
sudo nano /etc/systemd/system/prometheus.service
```

添加以下内容：

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

### 3.4 启动 Prometheus 服务

```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus

# 检查服务状态
sudo systemctl status prometheus
```

### 3.5 验证 Prometheus

打开浏览器访问：`http://<prometheus-server-ip>:9090`

## 4. 部署 Node Exporter

### 4.1 下载并安装 Node Exporter

```bash
# 创建 node_exporter 用户
sudo useradd -M -s /bin/false node_exporter

# 下载 Node Exporter
cd /tmp
sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz

# 解压并安装
sudo tar xvf node_exporter-1.6.0.linux-amd64.tar.gz
sudo cp node_exporter-1.6.0.linux-amd64/node_exporter /usr/local/bin/

# 设置权限
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### 4.2 创建系统服务

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

添加以下内容：

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

### 4.3 启动 Node Exporter 服务

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

# 检查服务状态
sudo systemctl status node_exporter
```

### 4.4 验证 Node Exporter

打开浏览器访问：`http://<node-exporter-ip>:9100/metrics`

## 5. 部署 Grafana

### 5.1 安装 Grafana

```bash
# CentOS/Rocky Linux
sudo dnf install -y https://dl.grafana.com/oss/release/grafana-9.5.2-1.x86_64.rpm

# Ubuntu/Debian
sudo apt update
sudo apt install -y apt-transport-https software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo apt update
sudo apt install -y grafana
```

### 5.2 启动 Grafana 服务

```bash
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# 检查服务状态
sudo systemctl status grafana-server
```

### 5.3 验证 Grafana

打开浏览器访问：`http://<grafana-server-ip>:3000`

默认登录凭据：
- 用户名：admin
- 密码：admin

首次登录后会要求修改密码。

## 6. 配置 Prometheus 与 Grafana 对接

### 6.1 添加 Prometheus 数据源

1. 登录 Grafana 界面
2. 点击左侧菜单中的 "Configuration" → "Data sources"
3. 点击 "Add data source"
4. 选择 "Prometheus"
5. 在 "URL" 字段中输入 Prometheus 服务器的地址，例如：`http://localhost:9090`
6. 点击 "Save & Test" 按钮，确保连接成功

### 6.2 导入 Grafana 面板

1. 点击左侧菜单中的 "Dashboards"
2. 点击 "Import"
3. 在 "Import via grafana.com" 字段中输入面板 ID：`1860`（Node Exporter Full 面板）
4. 点击 "Load"
5. 选择之前添加的 Prometheus 数据源
6. 点击 "Import"

### 6.3 创建自定义面板

1. 点击左侧菜单中的 "Dashboards"
2. 点击 "New dashboard"
3. 点击 "Add new panel"
4. 在 "Query" 选项卡中选择 Prometheus 数据源
5. 输入查询语句，例如：`node_cpu_seconds_total{mode="idle"}`
6. 点击 "Apply"
7. 点击 "Save dashboard" 保存面板

## 7. 配置防火墙

### 7.1 开放必要的端口

```bash
# CentOS/Rocky Linux
sudo firewall-cmd --permanent --add-port=9090/tcp  # Prometheus
sudo firewall-cmd --permanent --add-port=9100/tcp  # Node Exporter
sudo firewall-cmd --permanent --add-port=3000/tcp  # Grafana
sudo firewall-cmd --reload

# Ubuntu/Debian
sudo ufw allow 9090/tcp
sudo ufw allow 9100/tcp
sudo ufw allow 3000/tcp
sudo ufw reload
```

## 8. 监控其他节点

### 8.1 在其他节点上安装 Node Exporter

在需要监控的其他服务器上重复执行 4.1-4.3 步骤安装 Node Exporter。

### 8.2 更新 Prometheus 配置

编辑 Prometheus 配置文件：

```bash
sudo nano /etc/prometheus/prometheus.yml
```

更新 `scrape_configs` 部分：

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100", "192.168.1.101:9100", "192.168.1.102:9100"]
```

重启 Prometheus 服务：

```bash
sudo systemctl restart prometheus
```

## 9. 高级配置

### 9.1 配置告警规则

创建告警规则文件：

```bash
sudo nano /etc/prometheus/alerts.yml
```

添加以下内容：

```yaml
groups:
- name: node_alerts
  rules:
  - alert: HighCPUUsage
    expr: (100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
      description: "CPU usage is above 80% for 5 minutes"

  - alert: HighMemoryUsage
    expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      description: "Memory usage is above 80% for 5 minutes"
```

更新 Prometheus 配置文件，添加告警规则：

```yaml
rule_files:
  - "alerts.yml"
```

重启 Prometheus 服务：

```bash
sudo systemctl restart prometheus
```

### 9.2 配置 Grafana 告警

1. 登录 Grafana 界面
2. 点击左侧菜单中的 "Alerting" → "Alert rules"
3. 点击 "New alert rule"
4. 配置告警规则和通知渠道

## 10. 维护和监控

### 10.1 定期备份

```bash
# 备份 Prometheus 配置
sudo cp /etc/prometheus/prometheus.yml /backup/

# 备份 Grafana 配置
sudo cp -r /etc/grafana /backup/
```

### 10.2 监控 Prometheus 本身

- 使用 Prometheus 监控自己的指标
- 在 Grafana 中创建 Prometheus 自身的监控面板

### 10.3 性能优化

- 调整 `scrape_interval` 和 `evaluation_interval`
- 配置合适的存储保留时间
- 启用压缩和降采样

## 11. 故障排除

### 11.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Prometheus 无法启动 | 配置文件错误 | 检查配置文件语法：`promtool check config /etc/prometheus/prometheus.yml` |
| Node Exporter 无法连接 | 防火墙阻止 | 检查防火墙规则，确保 9100 端口开放 |
| Grafana 无法连接 Prometheus | 网络问题 | 检查网络连接和 Prometheus 服务状态 |
| 指标数据不显示 | 采集间隔问题 | 调整 `scrape_interval` 参数 |

### 11.2 日志查看

```bash
# Prometheus 日志
sudo journalctl -u prometheus

# Node Exporter 日志
sudo journalctl -u node_exporter

# Grafana 日志
sudo journalctl -u grafana-server
```

## 12. 总结

本指南详细介绍了 Prometheus 和 Grafana 的部署过程，包括：

1. Prometheus 的安装和配置
2. Node Exporter 的安装和配置
3. Grafana 的安装和配置
4. Prometheus 与 Grafana 的对接
5. 监控多节点配置
6. 告警规则配置
7. 维护和故障排除

通过这些步骤，您可以构建一个完整的监控系统，实时监控服务器的状态和性能。

---

**注意**：本指南基于 Prometheus 2.45.0 和 Grafana 9.5.2 版本，实际部署时请使用最新版本。