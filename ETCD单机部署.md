# ETCD 单机部署指南

## 1. 环境准备

### 系统要求

- Linux 系统 (推荐 CentOS/RHEL 7+ 或 Ubuntu 16.04+)
- 2GB+ 内存
- 2CPU+ 核心

### 安装依赖

```bash
# CentOS/RHEL
sudo yum install -y wget tar

# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y wget tar
```

## 2. 下载并安装ETCD

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

## 3. 配置ETCD服务

### 创建数据目录

```bash
sudo mkdir -p /var/lib/etcd
sudo chmod 700 /var/lib/etcd
```

### 创建systemd服务文件

```bash
sudo tee /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd service
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name=etcd-single \
  --data-dir=/var/lib/etcd \
  --listen-client-urls=http://0.0.0.0:2379 \
  --advertise-client-urls=http://localhost:2379
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF
```

## 4. 启动并验证服务

### 启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
```

### 检查服务状态

```bash
sudo systemctl status etcd
```

### 验证ETCD功能

```bash
# 写入测试数据
etcdctl put testkey "testvalue"

# 读取测试数据
etcdctl get testkey
```

## 5. 基本使用示例

### 键值操作

```bash
# 写入数据
etcdctl put /message "Hello World"

# 读取数据
etcdctl get /message

# 删除数据
etcdctl del /message
```

### 监听键变化

```bash
etcdctl watch /message &
```

## 注意事项

1. 生产环境建议使用TLS加密通信
2. 单机部署仅适用于开发和测试环境
3. 重要数据建议定期备份
