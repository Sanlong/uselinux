# 容器化 Prometheus 配置与使用指南

## 1. 概述

容器化部署 Prometheus 是现代云原生环境中的常见做法，它提供了更灵活、可扩展的监控方案。本指南将详细介绍如何使用 Docker 和 Kubernetes 部署和配置 Prometheus，以及如何使用它监控容器化应用。

## 2. 使用 Docker 部署 Prometheus

### 2.1 基本部署

```bash
# 创建数据目录
mkdir -p /prometheus/data

# 运行 Prometheus 容器
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /prometheus/data:/prometheus \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus:v2.45.0
```

### 2.2 基本配置文件

创建 `prometheus.yml` 配置文件：

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
      - targets: ["node-exporter:9100"]
```

### 2.3 使用 Docker Compose 部署完整监控栈

创建 `docker-compose.yml` 文件：

```yaml
version: '3'
services:
  prometheus:
    image: prom/prometheus:v2.45.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    restart: always

  node-exporter:
    image: prom/node-exporter:v1.6.0
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: always

  grafana:
    image: grafana/grafana:9.5.2
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    restart: always

volumes:
  prometheus_data:
  grafana_data:
```

启动服务：

```bash
docker-compose up -d
```

## 3. 使用 Kubernetes 部署 Prometheus

### 3.1 使用 Helm 部署

```bash
# 添加 Prometheus 仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 安装 Prometheus Stack
helm install prometheus prometheus-community/kube-prometheus-stack
```

### 3.2 自定义配置

创建 `values.yaml` 文件：

```yaml
alertmanager:
  enabled: true
  ingress:
    enabled: true
    hosts:
      - alertmanager.example.com

prometheus:
  enabled: true
  ingress:
    enabled: true
    hosts:
      - prometheus.example.com
  service:
    type: ClusterIP

grafana:
  enabled: true
  ingress:
    enabled: true
    hosts:
      - grafana.example.com
  service:
    type: ClusterIP
  adminPassword: "your-secure-password"
```

使用自定义配置安装：

```bash
helm install prometheus prometheus-community/kube-prometheus-stack -f values.yaml
```

### 3.3 手动部署（不使用 Helm）

创建 Prometheus 配置文件 `prometheus-configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: "kubernetes-nodes"
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - source_labels: [__address__]
          regex: '(.*):10250'
          replacement: '${1}:9100'
          target_label: __address__

      - job_name: "kubernetes-pods"
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          regex: true
          action: keep
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          regex: (.+)
          target_label: __metrics_path__
          replacement: $1
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
```

创建 Prometheus 部署文件 `prometheus-deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
        - name: prometheus-storage
          mountPath: /prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        persistentVolumeClaim:
          claimName: prometheus-pvc
```

创建服务文件 `prometheus-service.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
  type: ClusterIP
```

应用部署：

```bash
kubectl create namespace monitoring
kubectl apply -f prometheus-configmap.yaml
kubectl apply -f prometheus-deployment.yaml
kubectl apply -f prometheus-service.yaml
```

## 4. 配置详解

### 4.1 全局配置

```yaml
global:
  scrape_interval: 15s  # 抓取间隔
  evaluation_interval: 15s  # 评估间隔
  external_labels:  # 外部标签
    monitor: 'codelab-monitor'
```

### 4.2 抓取配置

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "docker"
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: "kubernetes-apiservers"
    kubernetes_sd_configs:
    - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: default;kubernetes;https
```

### 4.3 告警规则配置

创建 `alerts.yml` 文件：

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

在 `prometheus.yml` 中引用：

```yaml
rule_files:
  - "alerts.yml"
```

## 5. 基本使用

### 5.1 访问 Prometheus UI

- **Docker 部署**：`http://localhost:9090`
- **Kubernetes 部署**：
  ```bash
  # 端口转发
  kubectl port-forward svc/prometheus 9090:9090 -n monitoring
  ```
  然后访问 `http://localhost:9090`

### 5.2 执行查询

在 Prometheus UI 的 "Graph" 标签页中执行查询：

- 查看 CPU 使用率：
  ```
  100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  ```

- 查看内存使用率：
  ```
  (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
  ```

- 查看容器 CPU 使用率：
  ```
  sum(rate(container_cpu_usage_seconds_total{name!=""}[1m])) by (name)
  ```

### 5.3 配置 Grafana 数据源

1. 访问 Grafana UI：`http://localhost:3000`
2. 登录（默认用户名/密码：admin/admin）
3. 点击左侧菜单中的 "Configuration" → "Data sources"
4. 点击 "Add data source"
5. 选择 "Prometheus"
6. 在 "URL" 字段中输入 Prometheus 地址，例如：`http://prometheus:9090`
7. 点击 "Save & Test" 按钮，确保连接成功

### 5.4 导入 Grafana 面板

1. 点击左侧菜单中的 "Dashboards"
2. 点击 "Import"
3. 在 "Import via grafana.com" 字段中输入面板 ID：
   - `1860`：Node Exporter Full
   - `12633`：Docker Container Monitoring
   - `3119`：Kubernetes Cluster Monitoring
4. 点击 "Load"
5. 选择之前添加的 Prometheus 数据源
6. 点击 "Import"

## 6. 高级配置

### 6.1 持久化存储

**Docker 部署**：

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /prometheus/data:/prometheus \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus:v2.45.0
```

**Kubernetes 部署**：

创建 PVC：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: standard
```

### 6.2 高可用配置

**Docker 部署**：使用 Docker Compose 部署多个 Prometheus 实例，配合负载均衡器

**Kubernetes 部署**：使用 StatefulSet 部署多个 Prometheus 实例

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceName: prometheus
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.45.0
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
        - name: prometheus-storage
          mountPath: /prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
  volumeClaimTemplates:
  - metadata:
      name: prometheus-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 50Gi
      storageClassName: standard
```

### 6.3 远程存储

配置 Prometheus 使用远程存储：

```yaml
remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"

remote_read:
  - url: "http://victoriametrics:8428/api/v1/read"
```

## 7. 监控容器化应用

### 7.1 监控 Docker 容器

1. 部署 cAdvisor：

```bash
docker run -d \
  --name cadvisor \
  -p 8080:8080 \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  gcr.io/cadvisor/cadvisor:v0.47.0
```

2. 在 Prometheus 配置中添加 cAdvisor 抓取：

```yaml
scrape_configs:
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```

### 7.2 监控 Kubernetes 应用

1. 为应用添加 Prometheus 注解：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        ports:
        - containerPort: 8080
```

2. 配置 Prometheus 抓取 Kubernetes Pod：

```yaml
scrape_configs:
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      regex: true
      action: keep
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      regex: (.+)
      target_label: __metrics_path__
      replacement: $1
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
```

## 8. 故障排除

### 8.1 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| Prometheus 无法启动 | 配置文件错误 | 检查配置文件语法：`docker exec prometheus promtool check config /etc/prometheus/prometheus.yml` |
| 无法抓取目标 | 网络问题 | 检查网络连接和防火墙规则 |
| 数据丢失 | 存储配置错误 | 检查持久化存储配置 |
| 性能问题 | 抓取配置不当 | 调整抓取间隔和评估间隔 |

### 8.2 查看日志

**Docker 部署**：
```bash
docker logs prometheus
```

**Kubernetes 部署**：
```bash
kubectl logs deployment/prometheus -n monitoring
```

## 9. 最佳实践

1. **合理配置抓取间隔**：根据应用特点调整抓取间隔，避免过度抓取
2. **使用持久化存储**：确保数据不会丢失
3. **配置告警规则**：及时发现和处理问题
4. **使用 Grafana 可视化**：直观展示监控数据
5. **定期备份配置**：确保配置可以快速恢复
6. **监控 Prometheus 本身**：确保监控系统的可靠性
7. **使用服务发现**：自动发现和监控新的目标
8. **优化存储**：配置适当的存储保留时间

## 10. 总结

容器化部署 Prometheus 是现代云原生环境中的最佳实践，它提供了灵活、可扩展的监控方案。通过本指南，您可以：

1. 使用 Docker 或 Kubernetes 部署 Prometheus
2. 配置 Prometheus 监控容器化应用
3. 与 Grafana 集成实现数据可视化
4. 实现 Prometheus 高可用
5. 排查常见问题

通过合理配置和使用 Prometheus，您可以构建一个强大的监控系统，确保容器化应用的稳定运行。