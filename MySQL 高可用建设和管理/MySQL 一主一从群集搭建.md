# MySQL 一主一从群集搭建

## 准备工作

确保两台 MySQL 服务器以二进制方式搭建并保证正常工作

## 主服务器配置

### 启用 log-bin

```bash
# cat /etc/m y.cnf

[mysqld]
# mysql 基础地址
basedir=/usr/local/mysql
# mysql 数据地址
datadir=/data/mysql/data
port=3306
socket=/usr/local/mysql/mysql.sock
character-set-server=utf8
log-error=/var/log/mysqld.log
pid-file=/tmp/mysqld.pid
# 主服务器唯一 ID
server-id=1
#启用二进制日志
log-bin=mysql-bin
#设置不要复制的数据库名，可设置多个
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
#设置需要复制的数据库名
binlog-do-db=mnc_bak
# 设置 logbin 格式
binlog_format=STATEMENT
[mysql]
socket=/usr/local/mysql/mysql.sock
[client]
socket=/usr/local/mysql/mysql.sock
```

### 重启 MySQL

```bash
/etc/init.d/mysqld restart
```

### 创建同步账号 slave 并授权

```sql
mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%' IDENTIFIED BY '123123';

# 查看主服务器状态，并记录 File 和 Position 的值
mysql> show master status;
```

完成后不要再操作主服务器 MySQL，以防止主服务器状态发生变化。

## 从服务器配置

### 启用 relay-log

```bash
# cat /etc/m y.cnf

[mysqld]
basedir=/usr/local/mysql
datadir=/data/mysql
port=3306
socket=/usr/local/mysql/mysql.sock
character-set-server=utf8
log-error=/var/log/mysqld.log
pid-file=/tmp/mysqld.pid
# 从服务器唯一 ID
server-id=2
# 启用中继日志
relay-log=mysql-relay
[mysql]
socket=/usr/local/mysql/mysql.sock
[client]
socket=/usr/local/mysql/mysql.sock
```

### 重启 MySQL

```bash
/etc/init.d/mysqld restart
```

### 配置需要复制的主服务器信息

```sql
mysql> CHANGE MASTER TO MASTER_HOST='192.168.102.242', # 主服务器 ip 地址
MASTER_USER='slave', # 主服务器同步账号用户名
MASTER_PASSWORD='123123', # 主服务器同步账号密码
MASTER_LOG_FILE='mysql-bin.000001', # 主服务器记录的 File 值
MASTER_LOG_POS=430; # 主服务器记录的 Position 值

mysql> start slave; # 启动从服务器复制功能

mysql> show slave status\G; # 查看从服务器状态
```

## 解除主从状态

### 停止从服务复制功能

```sql
mysql> stop slave;
```

### 重新配置主从

```sql
mysql> stop slave;

mysql> reset master;
```