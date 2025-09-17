# CentOS 手动安装 Nginx

## 1. 安装前准备

安装编译 Nginx 需要的依赖包：

```bash
sudo yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

## 2. 下载 Nginx 源码包

前往 [Nginx 官网](https://nginx.org/en/download.html) 获取最新版本链接，使用 `wget` 下载：

```bash
wget https://nginx.org/download/nginx-1.24.0.tar.gz
```

>

## 3. 解压与编译安装

```bash
tar -zxvf nginx-1.24.0.tar.gz

cd nginx-1.24.0

./configure \
  --prefix=/home/steve/application/nginx \
  --with-http_ssl_module \
  --with-http_stub_status_module

make

sudo make install
```

## 4. 参数说明：

- `--prefix`：指定安装路径
- `--with-http_ssl_module`：启用 SSL 支持
- `--with-http_stub_status_module`：启用状态监控模块

## 5. 启动、停止、重载命令

默认安装路径为 `/home/steve/application/nginx`，相关命令如下：

```bash
# 启动 Nginx
sudo /home/steve/application/nginx/sbin/nginx
# 停止 Nginx
sudo /home/steve/application/nginx/sbin/nginx -s stop
# 重载配置
sudo /home/steve/application/nginx/sbin/nginx -s reload
# 检查配置文件语法
sudo /home/steve/application/nginx/sbin/nginx -t
```

## 6. 设置开机自启（Systemd 配置）

创建 systemd 服务文件 `/etc/systemd/system/nginx.service`：

```bash
sudo tee /etc/systemd/system/nginx.service > /dev/null << 'EOF'
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
ExecStart=/home/steve/application/nginx/sbin/nginx
ExecReload=/home/steve/application/nginx/sbin/nginx -s reload
ExecStop=/home/steve/application/nginx/sbin/nginx -s quit
PIDFile=/home/steve/application/nginx/logs/nginx.pid
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

重新加载 systemd 并设置开机自启：

```bash
sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl start nginx
```

---
如需卸载 Nginx，只需删除安装目录（如 `/home/steve/application/nginx`）并移除 systemd 服务文件即可。

## 7. 开放端口 80 443

```bash
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

## 8. 目录授权

```bash
chmod 755 /home/steve
chmod 755 /home/steve/application
chmod -R 755 /home/steve/application/nginx/html
chmod 644 /home/steve/application/nginx/html/index.html
```

## 9. Tips

- 在 Linux 上，1024 以下的端口需要 root 权限。
- Nginx 默认要监听 80/443 → 所以 master 进程通常必须以 root 启动。

## 10. 全局配置优化

### worker_processes
将 `worker_processes` 设置为 `auto`，让Nginx根据CPU核心数自动调整工作进程数。这样可以充分利用多核CPU的处理能力。

### worker_rlimit_nofile
设置 `worker_rlimit_nofile` 为 65535，增加文件描述符限制。这允许每个工作进程处理更多的并发连接，避免因文件描述符不足导致的性能瓶颈。

## 11. Events块优化

### worker_connections
增加 `worker_connections` 到 4096 或更高，提高单个工作进程能处理的并发连接数。

### use epoll
添加 `use epoll;` 以在Linux系统上使用epoll事件模型，相比传统的select模型能更高效地处理大量并发连接。

### multi_accept
启用 `multi_accept on;` 允许一个worker进程尽可能多地接受新连接，提高连接处理效率。

## 12. HTTP块优化

### TCP优化
- `tcp_nopush on;` - 启用TCP_NOPUSH选项，允许TCP协议栈优化数据包发送
- `tcp_nodelay on;` - 启用TCP_NODELAY选项，禁用Nagle算法，减少小包传输延迟

### Keepalive优化
- `keepalive_timeout 30;` - 设置keepalive连接超时时间为30秒，保持连接以减少重复连接开销
- `keepalive_requests 1000;` - 设置单个keepalive连接最多处理1000个请求

### Gzip压缩优化
- `gzip on;` - 启用gzip压缩
- `gzip_vary on;` - 添加Vary头以确保代理正确处理压缩内容
- `gzip_min_length 1024;` - 只压缩大于1KB的响应，避免压缩小文件的开销
- `gzip_comp_level 6;` - 设置压缩级别为6，平衡压缩率和CPU消耗
- `gzip_types` - 指定需要压缩的MIME类型，包括文本、CSS、JavaScript等

### 缓冲区优化
- `client_body_buffer_size 128k;` - 设置客户端请求体缓冲区大小
- `client_max_body_size 10m;` - 设置客户端最大请求体大小
- `client_header_buffer_size 1k;` - 设置客户端请求头缓冲区大小
- `large_client_header_buffers 4 4k;` - 设置大客户端请求头缓冲区数量和大小
- `output_buffers 1 32k;` - 设置输出缓冲区
- `postpone_output 1460;` - 设置延迟输出阈值

### 超时设置
- `client_header_timeout 3m;` - 设置读取客户端请求头的超时时间
- `client_body_timeout 3m;` - 设置读取客户端请求体的超时时间
- `send_timeout 3m;` - 设置响应传输的超时时间

### 文件缓存
- `open_file_cache max=1000 inactive=20s;` - 设置文件缓存的最大文件数和非活动时间
- `open_file_cache_valid 30s;` - 设置检查文件缓存有效性的频率
- `open_file_cache_min_uses 2;` - 设置在inactive时间内文件至少被使用2次才缓存
- `open_file_cache_errors on;` - 缓存文件查找错误信息

## 13. 静态资源缓存优化

### expires
设置 `expires 1y;` 为静态资源添加1年缓存期，减少重复请求。

### Cache-Control
添加 `Cache-Control: public, immutable` 头，告诉浏览器和其他代理可以缓存这些资源，并且它们不会改变。

## 14. 代理优化

### HTTP/1.1支持
设置 `proxy_http_version 1.1;` 启用HTTP/1.1支持，允许持久连接。

### 连接超时
- `proxy_connect_timeout 60s;` - 设置与后端服务器建立连接的超时时间
- `proxy_send_timeout 60s;` - 设置向后端服务器发送请求的超时时间
- `proxy_read_timeout 60s;` - 设置从后端服务器读取响应的超时时间

## 15. 应用优化配置示例

```nginx
# 全局配置优化
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    tcp_nopush on;
    tcp_nodelay on;

    keepalive_timeout 30;
    keepalive_requests 1000;

    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/xml+rss
        application/json;

    client_body_buffer_size 128k;
    client_max_body_size 10m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 4k;
    output_buffers 1 32k;
    postpone_output 1460;

    client_header_timeout 3m;
    client_body_timeout 3m;
    send_timeout 3m;

    open_file_cache max=1000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    server {
        # 静态资源缓存
        location /static/ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # 代理优化
        location /api/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_cache_bypass $http_upgrade;
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }
    }
}
```

完成这些优化配置后，使用 `nginx -s reload` 命令重新加载配置以使更改生效。