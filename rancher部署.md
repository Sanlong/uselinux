# Rancher 部署指南（Kubernetes 集群部署方式）

## 一、环境准备

### 1.1 前置条件
- **Kubernetes 集群**: 已部署好的 Kubernetes 集群（建议 v1.25+）
- **kubectl**: 已配置好 kubectl 命令行工具，能正常访问目标集群
- **Helm**: 已安装 Helm 3（推荐 v3.10+）
- **StorageClass**: 集群需有可用的 StorageClass（用于持久化存储）
- **Ingress Controller**: 建议提前部署 NGINX Ingress Controller

### 1.2 集群资源要求
| 组件 | 最小要求 | 推荐配置 |
|------|---------|---------|
| CPU | 4 核 | 8 核 |
| 内存 | 8GB | 16GB |
| 磁盘 | 50GB | 100GB |

### 1.3 安装 Helm（如未安装）

```bash
# 下载 Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 验证安装
helm version
```

## 二、添加 Rancher Helm 仓库

```bash
# 添加 Rancher Helm 仓库
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

# 更新仓库索引
helm repo update
```

## 三、创建 Rancher 命名空间

```bash
kubectl create namespace cattle-system
```

## 四、安装 Cert-Manager（用于证书管理）

### 4.1 安装 cert-manager CRDs

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.crds.yaml
```

### 4.2 添加 cert-manager Helm 仓库

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

### 4.3 安装 cert-manager

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.0
```

### 4.4 验证 cert-manager 安装

```bash
kubectl get pods -n cert-manager
```

等待所有 Pod 状态变为 **Running**。

## 五、安装 Rancher

### 5.1 安装 Rancher（使用 Let's Encrypt 证书）

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set replicas=3 \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com
```

**参数说明：**
- `hostname`: Rancher 访问域名
- `replicas`: 副本数（高可用建议 3）
- `ingress.tls.source`: 证书来源（letsEncrypt、secret、rancher）
- `letsEncrypt.email`: Let's Encrypt 注册邮箱

### 5.2 使用自签名证书安装（测试环境）

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set replicas=3 \
  --set ingress.tls.source=rancher
```

### 5.3 使用自定义证书安装

1. 创建证书 Secret：
```bash
kubectl -n cattle-system create secret tls tls-rancher-ingress \
  --cert=tls.crt \
  --key=tls.key
```

2. 安装 Rancher：
```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set replicas=3 \
  --set ingress.tls.source=secret
```

### 5.4 验证 Rancher 安装

```bash
# 查看 Pod 状态
kubectl get pods -n cattle-system

# 查看 Service
kubectl get svc -n cattle-system

# 查看 Ingress
kubectl get ingress -n cattle-system
```

等待所有 Rancher Pod 状态变为 **Running**。

## 六、配置 Rancher

### 6.1 访问 Rancher UI

打开浏览器，访问：`https://<您的域名>`（如 https://rancher.example.com）

### 6.2 登录并设置密码

1. **获取初始密码**：
```bash
kubectl get secret -n cattle-system bootstrap-secret \
  -o go-template='{{.data.bootstrapPassword|base64decode}}'
```

2. 使用初始密码登录
3. 设置新的管理员密码
4. 设置 Rancher Server URL（使用部署时指定的域名）

### 6.3 添加集群

#### 6.3.1 添加自定义集群

1. 在 Rancher UI 中点击 **☰ > Cluster Management**
2. 点击 **Create**
3. 选择 **Custom**
4. 输入集群名称（如 `my-cluster`）
5. 配置集群选项：
   - **Kubernetes 版本**: 选择目标版本
   - **Networking**: 选择网络插件（Calico、Flannel 等）
   - **Authentication**: 配置认证方式
6. 点击 **Next**
7. 选择节点角色（etcd、Control Plane、Worker）
8. 复制生成的注册命令

#### 6.3.2 在目标节点上执行注册命令

```bash
# 在目标节点上执行（根据 Rancher UI 生成的命令）
sudo docker run -d --privileged --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/rancher:/var/lib/rancher \
  rancher/rancher-agent:v2.8.0 \
  --server https://rancher.example.com \
  --token abcdef1234567890 \
  --ca-checksum abc123def456
```

#### 6.3.3 等待集群就绪

在 Rancher UI 中查看集群状态，等待所有节点变为 **Active** 状态。

### 6.4 导入现有集群

1. 在 Rancher UI 中点击 **☰ > Cluster Management**
2. 点击 **Import**
3. 输入集群名称
4. 复制生成的 YAML 文件内容
5. 在目标集群上执行：
```bash
kubectl apply -f rancher-cluster.yaml
```

## 七、配置 Ingress Controller

### 7.1 安装 NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### 7.2 验证 Ingress Controller

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## 八、部署应用示例

### 8.1 创建 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
```

### 8.2 创建 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f nginx-service.yaml
```

### 8.3 创建 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - nginx.example.com
    secretName: tls-nginx
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

```bash
kubectl apply -f nginx-ingress.yaml
```

## 九、升级 Rancher

### 9.1 查看当前版本

```bash
helm list -n cattle-system
```

### 9.2 更新 Helm 仓库

```bash
helm repo update
```

### 9.3 升级 Rancher

```bash
helm upgrade rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --version <目标版本>
```

### 9.4 验证升级

```bash
kubectl get pods -n cattle-system
```

## 十、备份与恢复

### 10.1 备份 etcd（Rancher 管理的集群）

```bash
# 在 Rancher UI 中进入集群 > 更多操作 > 备份

# 或使用命令行（针对下游集群）
kubectl -n kube-system exec -it etcd-<节点名> -- \
  ETCDCTL_API=3 etcdctl snapshot save /tmp/backup.db
```

### 10.2 备份 Rancher 配置

```bash
# 导出 Rancher 配置
export KUBECONFIG=~/.kube/config
helm get values rancher -n cattle-system -o yaml > rancher-values-backup.yaml

# 备份 CRD
kubectl get crds -o yaml > rancher-crds-backup.yaml
```

## 十一、卸载 Rancher

### 11.1 删除 Rancher Helm 发布

```bash
helm uninstall rancher -n cattle-system
```

### 11.2 删除相关资源

```bash
# 删除命名空间
kubectl delete namespace cattle-system

# 删除 CRD（谨慎操作）
kubectl delete crds -l app=rancher
```

## 十二、常见问题

### 12.1 Rancher Pod 无法启动

```bash
# 查看 Pod 日志
kubectl logs -n cattle-system rancher-<pod-name>

# 检查资源限制
kubectl describe pods -n cattle-system rancher-<pod-name>
```

### 12.2 Ingress 无法访问

```bash
# 检查 Ingress 配置
kubectl describe ingress -n cattle-system rancher

# 检查证书状态
kubectl get certificates -n cattle-system
```

### 12.3 节点注册失败

1. 检查节点网络是否能访问 Rancher Server
2. 检查节点时间同步（NTP）
3. 检查节点防火墙规则
4. 查看 agent 日志：
```bash
docker logs rancher-agent
```

### 12.4 cert-manager 证书签发失败

```bash
# 查看证书状态
kubectl get certificates -n cattle-system

# 查看 issuer 日志
kubectl logs -n cert-manager -l app=cert-manager
```

---

**注意**: 本指南基于 Rancher v2.8.x 版本编写。部署前请查看官方文档获取最新信息。

官方文档: https://ranchermanager.docs.rancher.com/

---

## 附录：常用命令

```bash
# 查看 Rancher 版本
helm list -n cattle-system

# 查看集群状态
kubectl get clusters -A

# 查看节点状态
kubectl get nodes

# 查看所有命名空间
kubectl get namespaces

# 查看 Pod 状态
kubectl get pods -A
```
# Rancher 部署指南

## 一、环境准备

### 1.1 服务器要求
- **操作系统**: Ubuntu 20.04/22.04 LTS 或 CentOS 7/8/9
- **CPU**: 至少 2 核
- **内存**: 至少 4GB
- **磁盘**: 至少 20GB 可用空间
- **网络**: 确保服务器可以访问互联网

### 1.2 系统配置

#### 关闭防火墙（可选）
```bash
# Ubuntu/Debian
sudo ufw disable

# CentOS/RHEL
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

#### 禁用 SELinux（CentOS/RHEL）
```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

#### 配置主机名
```bash
sudo hostnamectl set-hostname rancher-server
```

## 二、安装 Docker

### 2.1 Ubuntu/Debian 安装 Docker

```bash
# 更新软件包索引
sudo apt-get update

# 安装必要依赖
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 添加 Docker GPG 密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 设置 Docker 仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

### 2.2 CentOS/RHEL 安装 Docker

```bash
# 安装依赖
sudo yum install -y yum-utils

# 添加 Docker 仓库
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# 安装 Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io
```

### 2.3 启动 Docker 服务

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### 2.4 验证 Docker 安装

```bash
docker --version
docker run hello-world
```

## 三、安装 Rancher

### 3.1 安装 Rancher Server

```bash
sudo docker run -d \
  --restart=unless-stopped \
  -p 80:80 \
  -p 443:443 \
  --privileged \
  rancher/rancher:latest
```

**参数说明：**
- `-d`: 后台运行
- `--restart=unless-stopped`: 容器退出时自动重启
- `-p 80:80`: 映射 HTTP 端口
- `-p 443:443`: 映射 HTTPS 端口
- `--privileged`: 赋予容器特权模式

### 3.2 查看容器状态

```bash
docker ps
```

### 3.3 获取初始密码

```bash
# 等待容器启动完成（约1-2分钟）
sleep 60

# 获取初始管理员密码
sudo docker logs <容器ID> 2>&1 | grep "Bootstrap Password"
```

或者使用容器名称：
```bash
sudo docker logs rancher 2>&1 | grep "Bootstrap Password"
```

## 四、配置 Rancher

### 4.1 访问 Rancher UI

打开浏览器，访问：`https://<服务器IP或域名>`

### 4.2 登录并设置密码

1. 使用初始密码登录
2. 设置新的管理员密码
3. 设置 Rancher Server URL（建议使用域名）

### 4.3 添加集群

#### 添加自定义集群
1. 点击 **Add Cluster**
2. 选择 **Custom**
3. 输入集群名称
4. 选择集群配置（Kubernetes 版本、网络插件等）
5. 复制生成的注册命令

#### 在 Worker 节点上执行注册命令

```bash
# 在每个 Worker 节点上执行类似如下命令
sudo docker run -d --privileged --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/rancher:/var/lib/rancher \
  rancher/rancher-agent:v2.7.0 \
  --server https://<rancher-server-ip> \
  --token <registration-token> \
  --ca-checksum <ca-checksum>
```

### 4.4 等待集群就绪

在 Rancher UI 中查看集群状态，等待所有节点变为 **Active** 状态。

## 五、配置 Ingress（可选）

### 5.1 安装 NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/nginx/deploy.yaml
```

### 5.2 验证 Ingress Controller

```bash
kubectl get pods -n ingress-nginx
```

## 六、部署应用示例

### 6.1 创建 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 6.2 创建 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

### 6.3 创建 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

## 七、常见问题

### 7.1 容器启动失败

```bash
# 查看日志
docker logs <容器ID>

# 检查端口占用
netstat -tlnp | grep 443
```

### 7.2 Rancher UI 无法访问

1. 检查防火墙设置
2. 检查 SELinux 状态
3. 检查 Docker 容器运行状态

### 7.3 节点无法注册

1. 检查节点网络连通性
2. 检查节点时间同步
3. 检查防火墙规则

## 八、升级 Rancher

```bash
# 停止旧容器
docker stop rancher

# 备份数据
docker cp rancher:/var/lib/rancher /backup/rancher-data

# 拉取新版本
docker pull rancher/rancher:latest

# 启动新容器
docker run -d \
  --restart=unless-stopped \
  -p 80:80 \
  -p 443:443 \
  --privileged \
  -v /var/lib/rancher:/var/lib/rancher \
  rancher/rancher:latest
```

## 九、卸载 Rancher

```bash
# 停止并删除容器
docker stop rancher
docker rm rancher

# 删除数据（谨慎操作）
rm -rf /var/lib/rancher
```

---

**注意**: 本指南基于 Rancher v2.7.x 版本编写。部署前请查看官方文档获取最新信息。

官方文档: https://ranchermanager.docs.rancher.com/