# MariaDB Galera 集群配置

## 1. 概述

Galera 是 MariaDB 的一个同步复制插件，提供以下特性：
- 同步复制，确保数据一致性
- 多主架构，所有节点均可读写
- 自动节点故障检测和恢复
- 无需手动配置主从关系
- 支持透明应用故障转移

本文档详细介绍 MariaDB Galera 集群的配置方法。

## 2. 环境准备

### 2.1 服务器规划

| 主机名 | IP地址 | 角色 |
|-------|-------|------|
| galera1 | 192.168.1.20 | 节点1 |
| galera2 | 192.168.1.21 | 节点2 |
| galera3 | 192.168.1.22 | 节点3 |

### 2.2 系统要求
- 操作系统：Rocky Linux 9 或 Ubuntu 22.04 LTS+  
- MariaDB 版本：10.6+  
- 内存：至少 8GB  
- 存储空间：至少 100GB  
- 网络：所有节点间网络互通，建议使用千兆以上网络

### 2.3 系统初始化

#### 2.3.1 配置主机名和 hosts 文件

在所有节点上执行以下操作：

```bash
# 配置主机名
hostnamectl set-hostname galera1  # 在 galera1 上执行
hostnamectl set-hostname galera2  # 在 galera2 上执行
hostnamectl set-hostname galera3  # 在 galera3 上执行

# 配置 hosts 文件
cat >> /etc/hosts << EOF
192.168.1.20 galera1
192.168.1.21 galera2
192.168.1.22 galera3
EOF
```

#### 2.3.2 配置防火墙（推荐）

在所有节点上执行以下操作，开放必要的端口：

```bash
# 开放 MariaDB 端口
sudo firewall-cmd --permanent --add-port=3306/tcp

# 开放 Galera 集群通信端口
sudo firewall-cmd --permanent --add-port=4567/tcp  # Galera 复制端口
sudo firewall-cmd --permanent --add-port=4568/tcp  # IST 端口
sudo firewall-cmd --permanent --add-port=4444/tcp  # SST 端口

# 重新加载防火墙规则
sudo firewall-cmd --reload
```

#### 2.3.3 配置 SELinux（推荐）

在所有节点上执行以下操作，配置 SELinux 规则：

```bash
# 安装 policycoreutils-python-utils（如果未安装）
sudo dnf install -y policycoreutils-python-utils

# 允许 MariaDB 网络连接
sudo setsebool -P mysql_connect_any 1

# 允许 Galera 集群通信
sudo semanage port -a -t mysqld_port_t -p tcp 4567
sudo semanage port -a -t mysqld_port_t -p tcp 4568
sudo semanage port -a -t mysqld_port_t -p tcp 4444
```

**注意**：在生产环境中，推荐使用上述配置方法，而不是完全关闭防火墙和 SELinux。关闭这些安全措施仅适用于测试环境或隔离网络中。

## 3. 安装 Galera 集群

### 3.1 安装 MariaDB Galera 包

在所有节点上执行：

```bash
# Rocky Linux 9
dnf install -y epel-release
dnf install -y https://mirrors.ustc.edu.cn/mariadb/yum/10.6/rhel9-amd64/rpms/MariaDB-server-10.6.16-1.el9.x86_64.rpm https://mirrors.ustc.edu.cn/mariadb/yum/10.6/rhel9-amd64/rpms/MariaDB-client-10.6.16-1.el9.x86_64.rpm https://mirrors.ustc.edu.cn/mariadb/yum/10.6/rhel9-amd64/rpms/galera-4-26.4.14-1.el9.x86_64.rpm

# Ubuntu 22.04 LTS+
apt update
apt install -y software-properties-common
apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
add-apt-repository 'deb [arch=amd64,arm64,ppc64el] http://mirror.nodesdirect.com/mariadb/repo/10.6/ubuntu jammy main'
apt update
apt install -y mariadb-server mariadb-client galera-4
```

## 4. 配置 Galera 集群

### 4.1 配置 server.cnf 文件

是的，即使使用 Galera 插件，仍然需要配置 `server.cnf` 文件来启用和配置 Galera 集群。在所有节点上修改 `/etc/my.cnf.d/server.cnf` 文件，添加以下配置：

```ini
[mysqld]
# 基本配置
server-id = 1  # 在 galera2 上为 2，galera3 上为 3
binlog_format = ROW
default-storage-engine = InnoDB
innodb_autoinc_lock_mode = 2
innodb_flush_log_at_trx_commit = 0
innodb_buffer_pool_size = 2G

# Galera 配置
wsrep_on = ON
wsrep_provider = /usr/lib64/galera-4/libgalera_smm.so
wsrep_cluster_name = "galera_cluster"
wsrep_cluster_address = "gcomm://192.168.1.20,192.168.1.21,192.168.1.22"
wsrep_node_name = "galera1"  # 在 galera2 上为 "galera2"，galera3 上为 "galera3"
wsrep_node_address = "192.168.1.20"  # 在 galera2 上为 "192.168.1.21"，galera3 上为 "192.168.1.22"
wsrep_sst_method = rsync
wsrep_sst_auth = "sstuser:sstpassword"

# 其他配置
skip_name_resolve = 1
```

### 4.2 启动 Galera 集群

#### 4.2.1 初始化第一个节点

在 galera1 上执行：

```bash
# 停止 MariaDB 服务
systemctl stop mariadb

# 初始化集群
galera_new_cluster

# 检查集群状态
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep%';"
```

#### 4.2.2 启动其他节点

在 galera2 和 galera3 上执行：

```bash
# 停止 MariaDB 服务
systemctl stop mariadb

# 启动服务（会自动加入集群）
systemctl start mariadb

# 检查集群状态
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep%';"
```

### 4.3 创建 SST 用户

在任意节点上执行：

```sql
mysql -u root -p

CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'sstpassword';
GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
FLUSH PRIVILEGES;
```

### 4.4 验证 Galera 集群状态

在任意节点上执行：

```sql
SHOW STATUS LIKE 'wsrep%';
```

关键状态指标：
- `wsrep_cluster_size`：集群节点数量，应该为 3
- `wsrep_cluster_status`：集群状态，应该为 "Primary"
- `wsrep_connected`：节点连接状态，应该为 "ON"
- `wsrep_ready`：节点就绪状态，应该为 "ON"

### 4.5 测试 Galera 集群功能

#### 4.5.1 创建测试数据库和表

在 galera1 上执行：

```sql
CREATE DATABASE test_galera;
USE test_galera;
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50),
  email VARCHAR(100)
);
INSERT INTO users (name, email) VALUES ('User1', 'user1@example.com');
```

#### 4.5.2 验证数据同步

在 galera2 和 galera3 上执行：

```sql
USE test_galera;
SELECT * FROM users;
```

应该能看到在 galera1 上创建的数据。

#### 4.5.3 测试写操作在所有节点

在 galera2 上执行：

```sql
USE test_galera;
INSERT INTO users (name, email) VALUES ('User2', 'user2@example.com');
```

在 galera3 上执行：

```sql
USE test_galera;
INSERT INTO users (name, email) VALUES ('User3', 'user3@example.com');
```

在所有节点上验证数据一致性：

```sql
USE test_galera;
SELECT * FROM users;
```

## 5. Galera 集群维护

### 5.1 节点管理

- **添加节点**：安装 Galera 包，配置 server.cnf 文件，启动服务即可自动加入集群
- **移除节点**：在节点上执行 `systemctl stop mariadb` 即可安全移除
- **节点故障恢复**：修复服务器后，启动服务即可自动重新加入集群

### 5.2 备份策略

- **全量备份**：使用 `mysqldump` 或 `mariabackup` 备份任意节点
- **增量备份**：使用 `mariabackup` 进行增量备份
- **一致性备份**：在备份前执行 `FLUSH TABLES WITH READ LOCK`

### 5.3 监控 Galera 集群

创建监控脚本 `check_galera.sh`：

```bash
#!/bin/bash

for host in galera1 galera2 galera3; do
  echo "Checking Galera status on $host..."
  mysql -h $host -u root -p'your_password' -e "SHOW STATUS LIKE 'wsrep%'" | grep -E "wsrep_cluster_size|wsrep_cluster_status|wsrep_connected|wsrep_ready"
done
```

## 6. Galera 集群注意事项

1. **网络要求**：Galera 集群对网络要求较高，建议使用低延迟、高带宽的网络
2. **数据一致性**：Galera 提供同步复制，确保所有节点数据一致
3. **写操作性能**：由于同步复制，写操作性能可能比传统复制稍慢
4. **节点数量**：建议使用奇数个节点（3、5等），以避免脑裂
5. **磁盘空间**：所有节点需要足够的磁盘空间，因为数据会在所有节点上存储
6. **备份**：定期备份集群数据，建议从非主节点备份

## 7. Galera 集群故障排查

### 7.1 常见问题及解决方法

- **集群分裂**：检查网络连接，确保所有节点可以相互通信
- **节点无法加入**：检查 `wsrep_cluster_address` 配置，确保 IP 地址正确
- **复制延迟**：检查网络带宽，调整 `wsrep_slave_threads` 参数
- **内存不足**：增加节点内存，调整 `innodb_buffer_pool_size`

### 7.2 查看 Galera 日志

Galera 日志通常位于 `/var/log/mariadb/mariadb.log`，可以通过以下命令查看：

```bash
tail -f /var/log/mariadb/mariadb.log | grep -i galera
```

## 8. Galera 集群性能优化

### 8.1 配置优化

- **innodb_buffer_pool_size**：建议设置为服务器内存的 50-70%
- **wsrep_slave_threads**：根据 CPU 核心数调整，通常设置为 CPU 核心数的一半
- **wsrep_provider_options**：调整 `gcache.size` 等参数
- **innodb_flush_log_at_trx_commit**：设置为 0 或 2 以提高性能

### 8.2 监控优化

- 使用 Prometheus 和 Grafana 监控 Galera 集群
- 设置慢查询日志，优化查询性能
- 定期分析表，优化索引

## 9. 总结

MariaDB Galera 集群是一个基于同步复制的高可用性解决方案，提供了强数据一致性和自动故障恢复能力。通过正确配置 `server.cnf` 文件和遵循最佳实践，可以构建一个稳定、可靠的 Galera 集群。

对于关键业务系统，Galera 集群是一个理想的选择，因为它确保了数据的一致性和服务的高可用性。

---

**注意**：本配置适用于中小型环境，大型生产环境可能需要更复杂的配置和监控策略。