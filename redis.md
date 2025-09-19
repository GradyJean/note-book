# Redis 5 手动安装文档（CentOS 7/8）

## 1. 安装依赖环境

```bash
yum install -y gcc gcc-c++ make
```

## 2. 下载 Redis 5 源码包

```bash
cd /home/steve/application
wget http://download.redis.io/releases/redis-5.0.14.tar.gz
tar -zxvf redis-5.0.14.tar.gz
cd redis-5.0.14
```

## 3. 编译安装

```bash
make
make PREFIX=/home/steve/application/redis install
```

## 4. 配置文件准备

```bash
cd /home/steve/application/redis
mkdir conf  data logs run
cp ../redis-5.0.14/redis.conf .
```

### 修改配置文件

```bash
vim redis.conf
```

### 配置文件内容如下：

```properties
# ===== Redis 5 基础配置（远程可连 / 无需密码） =====
# 网络
bind 0.0.0.0
protected-mode no
port 6379
timeout 0
tcp-keepalive 300
# 进程与日志
daemonize no
supervised no
pidfile /home/steve/application/redis/run/redis.pid
loglevel notice
logfile /home/steve/application/redis/logs/redis.log
databases 16
always-show-logo yes
# ---------- RDB 持久化 ----------
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /home/steve/application/redis/data
# ---------- AOF（关闭；需要更强持久性再改为 yes） ----------
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
# ---------- 内存与淘汰 ----------
maxmemory 10gb
# 缓存型业务建议：allkeys-lru；严格持久化建议保留 noeviction
maxmemory-policy noeviction
maxmemory-samples 5
# ---------- 复制（如做从库再配 replicaof / masterauth） ----------
replica-read-only yes
replica-serve-stale-data yes
repl-diskless-sync no
repl-disable-tcp-nodelay no
replica-priority 100
# ---------- 诊断 ----------
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
# ---------- 紧凑结构阈值 ----------
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
# ---------- 其他 ----------
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

```

## 5. 创建 Systemd 管理服务

```bash
vim /etc/systemd/system/redis.service
```

### 配置文件内容如下：

```properties
[Unit]
Description=Redis 5 Server
After=network-online.target
Wants=network-online.target
[Service]
Type=simple
User=steve
Group=steve
WorkingDirectory=/home/steve/application/redis
ExecStart=/home/steve/application/redis/bin/redis-server /home/steve/application/redis/conf/redis.conf
ExecStop=/usr/bin/redis-cli -h 127.0.0.1 -p 6379 shutdown
LimitNOFILE=10032
Restart=always
RestartSec=2
TimeoutStopSec=90
[Install]
WantedBy=multi-user.target

```

## 6. 启动与开机启动

```bash
systemctl daemon-reexec
systemctl start redis
systemctl enable redis

```

## 7. 验证安装

```bash
redis-cli -h 127.0.0.1 -p 6379
set test "hello world"
get test
quit
```

## 8. 开放防火墙端口（如需要远程访问）

```bash
firewall-cmd --zone=public --add-port=6379/tcp --permanent
firewall-cmd --reload
```