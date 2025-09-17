# CentOS 手动安装 Elasticsearch 2.3.4

## 1. 安装前准备
确保系统里有 JDK 1.8（Elasticsearch 2.x 只能用 Java 8）：
- 检查 Java 版本：

```bash
java -version
```

- 如果没有安装 JDK 1.8，请先安装。

---

## 2. 下载 Elasticsearch 2.3.4
- 访问 [官方存档地址](https://www.elastic.co/downloads/past-releases/elasticsearch-2-3-4) 下载 tar 包，或者使用 `wget`：

```bash
cd /home/steve/application
wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.4/elasticsearch-2.3.4.tar.gz
```

---

## 3. 解压安装
- 解压下载的包并重命名目录：

```bash
tar -zxvf elasticsearch-2.3.4.tar.gz
mv elasticsearch-2.3.4 elasticsearch
```

---

## 4. 创建用户
- 出于安全考虑，Elasticsearch 不能用 root 用户运行。
- 这里直接使用现有的 `steve` 用户：
  
```bash
chown -R steve:steve /home/steve/application/elasticsearch
```

---

## 5. 修改配置
- 进入配置目录：

```bash
cd /home/steve/application/elasticsearch/config
```

- 编辑 `elasticsearch.yml`，修改以下内容：

```yaml
cluster.name: tech-cluster
node.name: node-1
path.data: /home/steve/application/elasticsearch/data
path.logs: /home/steve/application/elasticsearch/logs
network.host: 0.0.0.0   # 或服务器内网 IP
http.port: 9200
bootstrap.mlockall: true
discovery.zen.ping.unicast.hosts: ["127.0.0.1"]
gateway.recover_after_nodes: 1
discovery.zen.minimum_master_nodes: 1  # 单节点设为1，避免分裂脑
# 启用所有脚本类型
script.inline: true
script.indexed: true
script.file: true
# 明确启用 Groovy 脚本引擎
script.groovy.sandbox.enabled: true

```

---

## 5.1 修改 JVM 内存配置
- 编辑 `bin/elasticsearch.in.sh`，添加或修改以下内容，用于分配 20G 内存。

```bash
ES_MIN_MEM=20g
ES_MAX_MEM=20g
```

## 5.2 系统资源限制配置
- 为保证 Elasticsearch 正常运行，需要调整系统资源限制。编辑 `/etc/security/limits.conf`，为 `steve` 用户添加如下配置：

```conf
steve soft memlock unlimited
steve hard memlock unlimited
steve soft nofile 65536
steve hard nofile 65536
steve soft nproc  4096
steve hard nproc  4096
```

---

## 6. 测试启动 Elasticsearch
- 切换到 `steve` 用户并启动 Elasticsearch：

```bash
su - steve
cd /home/steve/application/elasticsearch/bin
./elasticsearch -d
```

- 查看日志确认启动情况：

```bash
tail -f /home/steve/application/elasticsearch/logs/elasticsearch.log
```

---

## 7. 测试访问
- 在浏览器或使用 curl 访问：

```bash
curl http://localhost:9200
```

- 正常情况下会返回 JSON，包括版本号（2.3.4）。

---
## 8 离线安装插件

### 8.1. 下载插件 zip 包：
   ```bash
    cd /home/steve/application/elasticsearch/plugins
   wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v1.9.4/elasticsearch-analysis-ik-1.9.4.zip
   wget https://github.com/medcl/elasticsearch-analysis-mmseg/releases/download/v1.9.4/elasticsearch-analysis-mmseg-1.9.4.zip
   wget https://github.com/lukas-vlcek/bigdesk/archive/master.zip -O bigdesk-master.zip
   ```

### 8.2 解压插件 zip 包：
   ```bash
   mkdir analysis-ik analysis-mmseg bigdesk
   unzip elasticsearch-analysis-ik-1.9.4.zip -d analysis-ik
   unzip bigdesk-master.zip -d bigdesk
   unzip elasticsearch-analysis-mmseg-1.9.4.zip -d analysis-mmseg
   ```
---

## 9. 设置开机自启
- 创建 systemd 服务文件 `/etc/systemd/system/elasticsearch.service`，内容如下：

```ini
[Unit]
Description=Elasticsearch
After=network.target

[Service]
Type=simple
User=steve
Group=steve
ExecStart=/home/steve/application/elasticsearch/bin/elasticsearch
Restart=on-failure
LimitMEMLOCK=infinity
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

- 启用并启动服务：

```bash
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
```
