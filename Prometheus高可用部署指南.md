# Prometheus 高可用部署指南

## 1. 概述

在生产环境中，Prometheus 的高可用性至关重要，因为它是监控系统的核心组件。如果 Prometheus 服务不可用，将导致监控数据丢失和告警失效，影响整个系统的可观测性。本指南将详细介绍 Prometheus 高可用的实现方案。

## 2. Prometheus 高可用架构

### 2.1 常见的高可用方案

#### 2.1.1 基于联邦集群的高可用

- **架构**：多个 Prometheus 实例并行运行，通过联邦机制聚合数据
- **优点**：简单易实现，无单点故障
- **缺点**：数据可能存在不一致性

#### 2.1.2 基于 Thanos 的高可用

- **架构**：使用 Thanos 组件实现 Prometheus 高可用和长期存储
- **优点**：支持长期存储，数据一致性好，查询性能高
- **缺点**：架构复杂，部署和维护成本高

#### 2.1.3 基于 Victoria Metrics 的高可用

- **架构**：使用 Victoria Metrics 作为 Prometheus 的长期存储和高可用方案
- **优点**：高性能，高压缩率，支持水平扩展
- **缺点**：生态系统相对较小

## 3. 基于联邦集群的高可用方案

### 3.1 架构设计

```
+---------------------+    +---------------------+
|  Prometheus 实例 1   |    |  Prometheus 实例 2   |
+---------------------+    +---------------------+
|  负责收集区域 A 数据   |    |  负责收集区域 B 数据   |
+---------------------+    +---------------------+
          |                          |
          +--------------------------+
                         |
                 +---------------------+
                 |  联邦 Prometheus    |
                 +---------------------+
                 |  聚合所有数据      |
                 +---------------------+
                         |
                 +---------------------+
                 |      Grafana        |
                 +---------------------+
```

### 3.2 部署步骤

#### 3.2.1 部署多个 Prometheus 实例

按照之前的部署指南，在不同的服务器上部署多个 Prometheus 实例。

#### 3.2.2 配置联邦 Prometheus

创建联邦 Prometheus 配置文件：

```bash
sudo nano /etc/prometheus/prometheus.yml
```

添加以下内容：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "federate"
    scrape_interval: 15s
    honor_labels: true
    metrics_path: "/federate"
    params:
      "match[]":
        - '{job=~".+"}'
    static_configs:
      - targets:
          - "prometheus1:9090"
          - "prometheus2:9090"
```

#### 3.2.3 配置负载均衡

使用 Nginx 或 HAProxy 配置负载均衡：

```bash
sudo nano /etc/nginx/nginx.conf
```

添加以下内容：

```nginx
http {
    upstream prometheus_backend {
        server prometheus1:9090 max_fails=3 fail_timeout=30s;
        server prometheus2:9090 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name prometheus.example.com;

        location / {
            proxy_pass http://prometheus_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_read_timeout 300s;
        }
    }
}
```

## 4. 基于 Thanos 的高可用方案

### 4.1 架构设计

```
+---------------------+    +---------------------+
|  Prometheus 实例 1   |    |  Prometheus 实例 2   |
+---------------------+    +---------------------+
|  配置 remote_write   |    |  配置 remote_write   |
+---------------------+    +---------------------+
          |                          |
          v                          v
+---------------------+    +---------------------+
|  Thanos Sidecar 1    |    |  Thanos Sidecar 2    |
+---------------------+    +---------------------+
          |                          |
          +--------------------------+
                         |
                 +---------------------+
                 |  Thanos Query      |
                 +---------------------+
                 |  聚合查询结果      |
                 +---------------------+
                         |
                 +---------------------+
                 |  Thanos Store      |
                 +---------------------+
                 |  长期存储          |
                 +---------------------+
                         |
                 +---------------------+
                 |  Thanos Compact    |
                 +---------------------+
                 |  压缩和降采样      |
                 +---------------------+
                         |
                 +---------------------+
                 |      Grafana        |
                 +---------------------+
```

### 4.2 部署步骤

#### 4.2.1 安装 Thanos

```bash
# 下载 Thanos
cd /tmp
sudo wget https://github.com/thanos-io/thanos/releases/download/v0.32.0/thanos-0.32.0.linux-amd64.tar.gz

# 解压并安装
sudo tar xvf thanos-0.32.0.linux-amd64.tar.gz
sudo cp thanos-0.32.0.linux-amd64/thanos /usr/local/bin/

# 设置权限
sudo chmod +x /usr/local/bin/thanos
```

#### 4.2.2 配置 Prometheus 远程写入

编辑 Prometheus 配置文件：

```bash
sudo nano /etc/prometheus/prometheus.yml
```

添加以下内容：

```yaml
remote_write:
  - url: "http://localhost:19291/api/v1/receive"
```

#### 4.2.3 启动 Thanos Sidecar

```bash
sudo nano /etc/systemd/system/thanos-sidecar.service
```

添加以下内容：

```ini
[Unit]
Description=Thanos Sidecar
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/thanos sidecar \
    --prometheus.url=http://localhost:9090 \
    --tsdb.path=/var/lib/prometheus \
    --grpc-address=0.0.0.0:10901 \
    --http-address=0.0.0.0:19291

[Install]
WantedBy=multi-user.target
```

#### 4.2.4 启动 Thanos Query

```bash
sudo nano /etc/systemd/system/thanos-query.service
```

添加以下内容：

```ini
[Unit]
Description=Thanos Query
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/thanos query \
    --http-address=0.0.0.0:9090 \
    --store=prometheus1:10901 \
    --store=prometheus2:10901

[Install]
WantedBy=multi-user.target
```

#### 4.2.5 启动 Thanos Store

```bash
sudo nano /etc/systemd/system/thanos-store.service
```

添加以下内容：

```ini
[Unit]
Description=Thanos Store
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/thanos store \
    --http-address=0.0.0.0:19291 \
    --grpc-address=0.0.0.0:10901 \
    --data-dir=/var/lib/thanos/store \
    --objstore.config-file=/etc/thanos/objstore.yml

[Install]
WantedBy=multi-user.target
```

创建对象存储配置文件：

```bash
sudo mkdir -p /etc/thanos
sudo nano /etc/thanos/objstore.yml
```

添加以下内容（以 S3 为例）：

```yaml
type: S3
config:
  bucket: "thanos"
  endpoint: "s3.example.com"
  region: "us-east-1"
  access_key: "your-access-key"
  secret_key: "your-secret-key"
```

#### 4.2.6 启动 Thanos Compact

```bash
sudo nano /etc/systemd/system/thanos-compact.service
```

添加以下内容：

```ini
[Unit]
Description=Thanos Compact
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/thanos compact \
    --http-address=0.0.0.0:19291 \
    --data-dir=/var/lib/thanos/compact \
    --objstore.config-file=/etc/thanos/objstore.yml

[Install]
WantedBy=multi-user.target
```

#### 4.2.7 启动所有服务

```bash
# 启动 Thanos Sidecar
sudo systemctl daemon-reload
sudo systemctl start thanos-sidecar
sudo systemctl enable thanos-sidecar

# 启动 Thanos Query
sudo systemctl start thanos-query
sudo systemctl enable thanos-query

# 启动 Thanos Store
sudo systemctl start thanos-store
sudo systemctl enable thanos-store

# 启动 Thanos Compact
sudo systemctl start thanos-compact
sudo systemctl enable thanos-compact
```

### 4.3 配置 Grafana 数据源

1. 登录 Grafana 界面
2. 点击左侧菜单中的 "Configuration" → "Data sources"
3. 点击 "Add data source"
4. 选择 "Prometheus"
5. 在 "URL" 字段中输入 Thanos Query 的地址，例如：`http://thanos-query:9090`
6. 点击 "Save & Test" 按钮，确保连接成功

## 5. 基于 Victoria Metrics 的高可用方案

### 5.1 架构设计

```
+---------------------+    +---------------------+
|  Prometheus 实例 1   |    |  Prometheus 实例 2   |
+---------------------+    +---------------------+
|  配置 remote_write   |    |  配置 remote_write   |
+---------------------+    +---------------------+
          |                          |
          v                          v
+---------------------+    +---------------------+
|  Victoria Metrics    |    |  Victoria Metrics    |
+---------------------+    +---------------------+
|  主节点 (写入和查询)   |    |  副本节点 (只读)     |
+---------------------+    +---------------------+
          |                          |
          +--------------------------+
                         |
                 +---------------------+
                 |     Load Balancer   |
                 +---------------------+
                         |
                 +---------------------+
                 |      Grafana        |
                 +---------------------+
```

### 5.2 部署步骤

#### 5.2.1 安装 Victoria Metrics

```bash
# 下载 Victoria Metrics
cd /tmp
sudo wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.92.0/victoria-metrics-linux-amd64-v1.92.0.tar.gz

# 解压并安装
sudo tar xvf victoria-metrics-linux-amd64-v1.92.0.tar.gz
sudo cp victoria-metrics-linux-amd64-v1.92.0/victoria-metrics-prod /usr/local/bin/

# 设置权限
sudo chmod +x /usr/local/bin/victoria-metrics-prod
```

#### 5.2.2 配置 Prometheus 远程写入

编辑 Prometheus 配置文件：

```bash
sudo nano /etc/prometheus/prometheus.yml
```

添加以下内容：

```yaml
remote_write:
  - url: "http://victoria-metrics:8428/api/v1/write"
```

#### 5.2.3 启动 Victoria Metrics 主节点

```bash
sudo nano /etc/systemd/system/victoria-metrics.service
```

添加以下内容：

```ini
[Unit]
Description=Victoria Metrics
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/victoria-metrics-prod \
    --storageDataPath=/var/lib/victoria-metrics \
    --retentionPeriod=30d \
    --httpListenAddr=:8428

[Install]
WantedBy=multi-user.target
```

#### 5.2.4 启动 Victoria Metrics 副本节点

在另一台服务器上执行相同的安装步骤，然后创建服务文件：

```bash
sudo nano /etc/systemd/system/victoria-metrics-replica.service
```

添加以下内容：

```ini
[Unit]
Description=Victoria Metrics Replica
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/victoria-metrics-prod \
    --storageDataPath=/var/lib/victoria-metrics \
    --retentionPeriod=30d \
    --httpListenAddr=:8428 \
    --replicationFactor=2 \
    --dedup.minScrapeInterval=10s

[Install]
WantedBy=multi-user.target
```

#### 5.2.5 配置负载均衡

使用 Nginx 或 HAProxy 配置负载均衡：

```bash
sudo nano /etc/nginx/nginx.conf
```

添加以下内容：

```nginx
http {
    upstream victoria_metrics {
        server victoria-metrics-1:8428 max_fails=3 fail_timeout=30s;
        server victoria-metrics-2:8428 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name victoria-metrics.example.com;

        location / {
            proxy_pass http://victoria_metrics;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_read_timeout 300s;
        }
    }
}
```

#### 5.2.6 配置 Grafana 数据源

1. 登录 Grafana 界面
2. 点击左侧菜单中的 "Configuration" → "Data sources"
3. 点击 "Add data source"
4. 选择 "Prometheus"
5. 在 "URL" 字段中输入负载均衡器的地址，例如：`http://victoria-metrics.example.com`
6. 点击 "Save & Test" 按钮，确保连接成功

## 6. 高可用方案比较

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 联邦集群 | 简单易实现，成本低 | 数据一致性差，查询性能一般 | 小型环境，预算有限 |
| Thanos | 数据一致性好，支持长期存储 | 架构复杂，维护成本高 | 大型环境，需要长期存储 |
| Victoria Metrics | 高性能，高压缩率 | 生态系统相对较小 | 对性能要求高的环境 |

## 7. 故障转移和监控

### 7.1 故障检测

- 使用 Prometheus 监控自身和其他 Prometheus 实例
- 配置告警规则，当实例不可用时触发告警
- 使用健康检查工具（如 Consul）监控服务状态

### 7.2 自动故障转移

- 使用负载均衡器的健康检查功能
- 配置自动故障转移策略
- 实现服务发现机制，动态更新可用实例

### 7.3 监控高可用状态

创建专门的监控面板，监控：
- 各 Prometheus 实例的状态
- 数据采集和存储状态
- 告警触发情况
- 系统资源使用情况

## 8. 最佳实践

### 8.1 部署建议

- **避免单点故障**：所有组件都应该有冗余
- **网络隔离**：将监控系统放在独立的网络中
- **数据备份**：定期备份配置和数据
- **安全加固**：配置防火墙，限制访问

### 8.2 配置建议

- **合理设置采集间隔**：根据业务需求调整 `scrape_interval`
- **配置数据保留策略**：根据存储容量和需求设置保留时间
- **启用压缩**：减少存储占用
- **配置合理的告警规则**：避免告警风暴

### 8.3 维护建议

- **定期更新**：保持软件版本最新
- **监控系统本身**：确保监控系统的健康状态
- **性能优化**：定期检查和优化系统性能
- **文档更新**：及时更新部署和维护文档

## 9. 故障排除

### 9.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 数据不一致 | 网络延迟或配置错误 | 检查网络连接，确保配置正确 |
| 服务不可用 | 资源耗尽或配置错误 | 检查系统资源，查看日志 |
| 查询性能差 | 数据量过大或查询复杂 | 优化查询，增加资源 |
| 告警不触发 | 配置错误或网络问题 | 检查告警规则，测试通知渠道 |

### 9.2 日志查看

```bash
# Prometheus 日志
sudo journalctl -u prometheus

# Thanos 日志
sudo journalctl -u thanos-sidecar
sudo journalctl -u thanos-query
sudo journalctl -u thanos-store
sudo journalctl -u thanos-compact

# Victoria Metrics 日志
sudo journalctl -u victoria-metrics
```

## 10. 总结

本指南介绍了三种 Prometheus 高可用方案：

1. **基于联邦集群的高可用**：简单易实现，适合小型环境
2. **基于 Thanos 的高可用**：功能强大，支持长期存储，适合大型环境
3. **基于 Victoria Metrics 的高可用**：高性能，适合对性能要求高的环境

选择合适的高可用方案需要考虑：
- 环境规模和预算
- 数据一致性要求
- 长期存储需求
- 性能要求

通过实施高可用方案，可以确保 Prometheus 在生产环境中的稳定性和可靠性，为系统监控提供持续的保障。

---

**注意**：本指南基于 Prometheus 2.45.0、Thanos 0.32.0 和 Victoria Metrics 1.92.0 版本，实际部署时请使用最新版本。