# Zookeeper 3.5.10 单节点安装文档

## 1. 环境准备
- 操作系统：CentOS 7/8
- 用户：`steve`
- 安装目录：`/home/steve/application/zookeeper`
- JDK：OpenJDK 1.8 已安装

## 2. 下载 Zookeeper
```bash
cd /home/steve/application
wget https://repo.huaweicloud.com/apache/zookeeper/zookeeper-3.5.10/apache-zookeeper-3.5.10-bin.tar.gz
tar -zxvf apache-zookeeper-3.5.10-bin.tar.gz
mv apache-zookeeper-3.5.10-bin zookeeper
```

## 3. 配置 Zookeeper

进入配置目录：
```bash
cd /home/steve/application/zookeeper/conf
cp zoo_sample.cfg zoo.cfg
```

编辑 `zoo.cfg`：
```properties
tickTime=2000
initLimit=5
syncLimit=2

# 数据与事务日志分目录，便于磁盘管理
dataDir=/home/steve/application/zookeeper/data
dataLogDir=/home/steve/application/zookeeper/logs

# 客户端端口（默认 2181）
clientPort=2181
maxClientCnxns=200

# 自动清理旧快照/日志，避免磁盘打满
autopurge.snapRetainCount=3
autopurge.purgeInterval=6

# 管理 HTTP（3.5 默认开启在 8080），单机一般关闭
admin.enableServer=false

# 可按需放开四字命令（仅用于排障）
4lw.commands.whitelist=srvr,stat,ruok

```

### 配置参数说明
- **tickTime**：心跳时间，单位毫秒，Zookeeper 使用的基本时间单位。
- **dataDir**：存储快照文件的目录。
- **clientPort**：客户端连接的端口，默认 2181。
- **initLimit**：集群模式下 Follower 初始化时的最大心跳数（单节点可保留默认）。
- **syncLimit**：集群模式下 Leader 与 Follower 之间请求和应答的最大心跳数（单节点可保留默认）。

创建数据目录：
```bash
mkdir -p /home/steve/application/zookeeper/data
chown -R steve:steve /home/steve/application/zookeeper
```
## 4. 日志配置

编辑 `conf/log4j.properties`：
```properties
zookeeper.root.logger=INFO, CONSOLE, ROLLINGFILE
zookeeper.console.threshold=INFO
zookeeper.log.dir=/home/steve/application/zookeeper/logs
zookeeper.log.file=zookeeper.log
zookeeper.log.threshold=INFO
zookeeper.tracelog.dir=/home/steve/application/zookeeper/logs
zookeeper.tracelog.file=zookeeper_trace.log
log4j.rootLogger=${zookeeper.root.logger}
log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender
log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern=%d{ISO8601} [%t] %-5p %c - %m%n
log4j.appender.ROLLINGFILE=org.apache.log4j.RollingFileAppender
log4j.appender.ROLLINGFILE.Threshold=INFO
log4j.appender.ROLLINGFILE.File=${zookeeper.log.dir}/${zookeeper.log.file}
log4j.appender.ROLLINGFILE.MaxFileSize=10MB
log4j.appender.ROLLINGFILE.MaxBackupIndex=10
log4j.appender.ROLLINGFILE.layout=org.apache.log4j.PatternLayout
log4j.appender.ROLLINGFILE.layout.ConversionPattern=%d{ISO8601} [%t] %-5p %c - %m%n
```

## 5. systemd 管理

新建 `etc/systemd/system/zookeeper.service`：
```ini
[Unit]
Description=Apache Zookeeper Server
After=network.target

[Service]
Type=forking
User=steve
Group=steve
ExecStart=/home/steve/application/zookeeper/bin/zkServer.sh start
ExecStop=/home/steve/application/zookeeper/bin/zkServer.sh stop
ExecReload=/home/steve/application/zookeeper/bin/zkServer.sh restart
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

复制到 systemd 目录并启用：
```bash
sudo systemctl daemon-reload
sudo systemctl enable zookeeper
sudo systemctl start zookeeper
sudo systemctl status zookeeper
```

## 6. 防火墙配置

放开 2181 端口：
```bash
sudo firewall-cmd --permanent --add-port=2181/tcp
sudo firewall-cmd --reload
```

## 7. 测试连接

使用自带客户端：
```bash
/home/steve/application/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181
```

测试命令：
```bash
ls /
create /test "hello"
get /test
delete /test
quit
```

---
✅ 至此，Zookeeper 3.5.10 单节点已完成安装与验证。
