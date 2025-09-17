# CentOS 安装 MySQL 8.0 

## 1. 创建安装目录并下载安装包

```bash
mkdir mysql
cd mysql
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.36-1.el7.x86_64.rpm-bundle.tar
```

## 2. 解压安装包

使用以下命令解压下载的 tar 包：

```bash
tar -xvf mysql-8.0.36-1.el7.x86_64.rpm-bundle.tar
```

## 3. 安装 MySQL

执行以下命令进行本地安装：

```bash
yum localinstall *.rpm
```

## 4. 启动 MySQL 服务

启动并设置 MySQL 开机自启：

```bash
sudo systemctl start mysqld
sudo systemctl enable mysqld
```

## 5. 进行安全配置

运行安全配置脚本：

```bash
mysql_secure_installation
```

按照提示完成以下配置：
- 设置 root 用户密码
- 删除匿名用户
- 禁用远程 root 登录
- 删除 test 数据库
- 重新加载权限表

## 6. 配置远程访问

### 6.1 修改配置文件

编辑 `/etc/my.cnf` 文件，添加以下内容：

```ini
[mysqld]
bind-address = 0.0.0.0
```

### 6.2 配置防火墙

开放 MySQL 默认端口 3306：

```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

### 6.3 创建远程访问用户

登录 MySQL 数据库并执行以下 SQL 语句：

```sql
CREATE USER 'root'@'%' IDENTIFIED BY 'YourStrongPass!';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

### 6.4 重启 MySQL 服务
```bash
sudo systemctl restart mysqld
```