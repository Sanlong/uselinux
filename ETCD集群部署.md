# ETCD 三节点集群部署指南 (带TLS)

## 0. 变量定义

```bash
# 节点信息变量
export NODE1_NAME="etcd1"
export NODE1_IP="<节点1IP>"
export NODE2_NAME="etcd2"
export NODE2_IP="<节点2IP>"
export NODE3_NAME="etcd3"
export NODE3_IP="<节点3IP>"
```

## 1. 环境准备

### 系统要求

- 三台Linux服务器 (推荐CentOS/RHEL 7+或Ubuntu 16.04+)
- 每台服务器: 2GB+内存, 2CPU+核心
- 服务器之间网络互通
- 时间同步(建议安装NTP)

### 安装依赖

```bash
# CentOS/RHEL
sudo yum install -y wget tar

# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y wget tar
```

## 2. TLS证书准备

### 创建CA证书

```bash
# 创建CA私钥
openssl genrsa -out ca.key 2048

# 创建CA证书
openssl req -new -x509 -days 365 -key ca.key -out ca.crt \
  -subj "/CN=etcd-ca"
```

### 为每个节点生成证书

```bash
# 生成私钥
openssl genrsa -out etcd1.key 2048

# 创建证书签名请求(CSR)
openssl req -new -key etcd1.key -out etcd1.csr \
  -subj "/CN=etcd1" \
  -addext "subjectAltName=DNS:${NODE1_NAME},DNS:localhost,IP:127.0.0.1,IP:${NODE1_IP}"

# 使用CA签名证书
openssl x509 -req -days 365 -in etcd1.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out etcd1.crt
```

## 3. 下载并安装ETCD

### 下载最新稳定版

```bash
ETCD_VER=v3.5.0
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
```

### 解压并安装

```bash
tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
cd etcd-${ETCD_VER}-linux-amd64
sudo mv etcd etcdctl /usr/local/bin/
```

## 4. 配置ETCD集群

### 创建数据目录

```bash
sudo mkdir -p /var/lib/etcd
sudo chmod 700 /var/lib/etcd
```

### 创建systemd服务文件(节点1示例)

```bash
sudo tee /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd service
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name=${NODE1_NAME} \
  --data-dir=/var/lib/etcd \
  --initial-advertise-peer-urls=https://${NODE1_IP}:2380 \
  --listen-peer-urls=https://0.0.0.0:2380 \
  --listen-client-urls=https://0.0.0.0:2379 \
  --advertise-client-urls=https://${NODE1_IP}:2379 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-cluster=${NODE1_NAME}=https://${NODE1_IP}:2380,${NODE2_NAME}=https://${NODE2_IP}:2380,${NODE3_NAME}=https://${NODE3_IP}:2380 \
  --initial-cluster-state=new \
  --client-cert-auth \
  --trusted-ca-file=/etc/etcd/ssl/ca.crt \
  --cert-file=/etc/etcd/ssl/etcd1.crt \
  --key-file=/etc/etcd/ssl/etcd1.key \
  --peer-client-cert-auth \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.crt \
  --peer-cert-file=/etc/etcd/ssl/etcd1.crt \
  --peer-key-file=/etc/etcd/ssl/etcd1.key
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF
```

## 5. 启动集群并验证

### 启动所有节点服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
```

### 检查集群状态

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://${NODE1_IP}:2379,https://${NODE2_IP}:2379,https://${NODE3_IP}:2379 \
  --cacert=/etc/etcd/ssl/ca.crt \
  --cert=/etc/etcd/ssl/etcd1.crt \
  --key=/etc/etcd/ssl/etcd1.key \
  endpoint status --write-out=table
```

## 6. 安全注意事项

1. 妥善保管CA私钥
2. 定期轮换证书
3. 限制访问ETCD端口的IP
4. 生产环境建议使用更复杂的证书配置
5. 定期备份ETCD数据
