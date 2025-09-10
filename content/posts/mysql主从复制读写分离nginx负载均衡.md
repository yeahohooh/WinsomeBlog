---
title: "Mysql 主从复制 读写分离 nginx 负载均衡"
date: 2024-10-25T18:36:33+08:00
draft: false
tags: 
  - Golang
author : winsome
scrolltotop : true
toc : true
mathjax : false
---

记录一篇流水账，关于这些配置。

场景概述：
- 服务器 A：将作为主服务器，Nginx 运行在 8080 端口。
- 服务器 B：将作为从服务器，Nginx 运行在 80 端口。
- 负载均衡目标：Nginx 实现两台服务器的流量均衡，将请求分发到两台服务器上。但写请求永远发送到主节点，读请求可以发往主或从节点。

### MySQL 主从复制配置
1.1 主服务器 A（Master）的 MySQL 配置：
1. 修改 MySQL 配置文件 /etc/mysql/mysql.conf.d/mysqld.cnf：
- cd 进入该文件夹，然后 `sudo nano mysqld.cnf`
- 下面的配置如果原文件没有就添加进去
- ctl + o 保存，ctl + x 退出
```
[mysqld]
server-id = 1                   # 唯一的服务器ID，主从服务器ID不能相同
log_bin = /var/log/mysql/mysql-bin.log  # 启用二进制日志
binlog_format = row             # 使用行格式日志
bind-address    = 0.0.0.0       # 允许所有网络接口
port            = 3306          # 端口 默认3306，可以改为别的
```
2. 重启 MySQL 服务：
```
sudo systemctl restart mysql
```
3. 创建复制用户：
`mysql -u root -p` 进入mysql
```
CREATE USER 'replicator'@'%' IDENTIFIED BY 'password';  -- 创建复制用户
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';     -- 赋予复制权限
FLUSH PRIVILEGES;                                    -- 刷新权限
```
4. 获取主服务器的二进制日志信息：
```
SHOW MASTER STATUS;
```
**记录下 File 和 Position，在从服务器配置时需要使用。**

1.2 从服务器 B（Slave）的 MySQL 配置：
1. 修改 MySQL 配置文件 /etc/mysql/mysql.conf.d/mysqld.cnf：
```
[mysqld]
server-id = 2                   # 从服务器的ID，与主服务器不同
relay-log = /var/log/mysql/mysql-relay-bin  # 启用中继日志
bind-address = 0.0.0.0       # 允许所有网络接口
port = 3306                  # 端口 默认3306，可以改为别的
```
2. 重启 MySQL 服务：
```
sudo systemctl restart mysql
```
3. 配置从服务器复制： 登录 MySQL 后执行：
```
CHANGE MASTER TO
    MASTER_HOST='A的IP地址',      -- 主服务器的IP地址
    MASTER_USER='replicator',        -- 复制用户
    MASTER_PASSWORD='password',   -- 复制用户的密码
    MASTER_LOG_FILE='记录的File',  -- 主服务器的二进制日志文件
    MASTER_LOG_POS=记录的Position; -- 主服务器的日志位置
START SLAVE;                      -- 启动复制
```
4. 检查复制状态：
```
SHOW SLAVE STATUS\G;
```
确保 Slave_IO_Running 和 Slave_SQL_Running 都是 Yes。

### 应用层的读写分离
Gorm 的 DBResolver 是一个非常有用的插件，能够帮助我们实现数据库的读写分离
1. 安装 Gorm DBResolver 插件
在使用 DBResolver 前，需要确保安装了 Gorm 的 DBResolver 插件：
```
go get gorm.io/plugin/dbresolver
```
2. 基本配置
- 配置主库和从库
- 自动区分读操作与写操作
- 支持设置主从数据库的负载均衡策略

**代码示例**
```go
package main

import (
    "fmt"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "gorm.io/plugin/dbresolver"
)

// 初始化数据库连接
func initDB() (*gorm.DB, error) {
    // 主库的 DSN
    masterDSN := "root:password@tcp(master-db:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"

    // 从库的 DSN
    slave1DSN := "root:password@tcp(slave1-db:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
    slave2DSN := "root:password@tcp(slave2-db:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"

    // 连接主库
    db, err := gorm.Open(mysql.Open(masterDSN), &gorm.Config{})
    if err != nil {
        return nil, err
    }

    // 配置读写分离
    db.Use(dbresolver.Register(dbresolver.Config{
        // 主库，用于写操作
        Sources: []gorm.Dialector{mysql.Open(masterDSN)},
        // 从库，用于读操作
        Replicas: []gorm.Dialector{
            mysql.Open(slave1DSN),
            mysql.Open(slave2DSN),
        },
        // 负载均衡策略，默认是随机选择一个从库
        Policy: dbresolver.RandomPolicy{},
    }).SetMaxOpenConns(10).SetMaxIdleConns(5))

    return db, nil
}

// 模型
type User struct {
    ID   uint
    Name string
}

func main() {
    // 初始化数据库
    db, err := initDB()
    if err != nil {
        panic(fmt.Sprintf("failed to connect to DB: %v", err))
    }

    // 写操作（插入数据）会使用主库
    user := User{Name: "Alice"}
    db.Create(&user)

    // 读操作（查询数据）会使用从库
    var users []User
    db.Find(&users)

    fmt.Println("用户数据：", users)
}
```
### Nginx 负载均衡配置
**服务器 A 的 Nginx 配置：**
1. 新建 /etc/nginx/sites-available/load_balancer.conf：
```
upstream go_backend {
    server 127.0.0.1:8082;    # 本地 Go 服务
    server B的公网IP:80;   # 负载到服务器B上的 Go 服务
}

server {
    listen 8080;              # 对外暴露的端口
    server_name A的公网IP;   # 服务器A的公网IP

    location / {
        proxy_pass http://go_backend;  # 反向代理到 go_backend 上游
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

2. 重启 Nginx：
```
// 创建符号链接将这个站点启用
sudo ln -s /etc/nginx/sites-available/load_balancer.conf /etc/nginx/sites-enabled/
// 测试 Nginx 配置是否有错误
sudo nginx -t
// 重启
sudo systemctl restart nginx
```
**服务器 B 的 Nginx 配置：**
1. 新建 /etc/nginx/sites-available/load_balancer.conf：
```
server {
    listen 80;                # 对外暴露的端口
    server_name B的公网IP;   # 服务器B的公网IP

    location / {
        proxy_pass http://127.0.0.1:8082;  # 反向代理到本地 Go 服务
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

2. 重启 Nginx：
```
// 创建符号链接将这个站点启用
sudo ln -s /etc/nginx/sites-available/load_balancer.conf /etc/nginx/sites-enabled/
// 测试 Nginx 配置是否有错误
sudo nginx -t
// 重启
sudo systemctl restart nginx
```

### 可能产生的错误
- 在 mysql 创建用户的时候，注意权限问题，有的权限复制过来的结构不全，
```
// 赋予用户'replicator'所有权限
GRANT ALL PRIVILEGES ON *.* TO 'replicator'@'%';
// 这里的 ALL PRIVILEGES 可以根据需求调整，如果你只需要备份和复制操作，可以改成更有限的权限，例如
GRANT SELECT, RELOAD, LOCK TABLES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'replicator'@'%';
// ON 后面可以写成指定的 database
GRANT ALL PRIVILEGES ON databaseName.* TO 'replicator'@'%';
// '%' 指允许所有 ip，可以替换成允许指定的 ip，生产环境建议这么做
GRANT ALL PRIVILEGES ON *.* TO 'replicator'@'IP地址';
```
- 注意防火墙配置，还有 mysql 配置的 bind-address 是否有限制
- 从库一定不能填错主库的 File 和 Position
- 从库在 `SHOW SLAVE STATUS\G;` 检查的时候，如果有 Last_SQL_Error 错误，也可以查看排查问题 `mysql> select * from performance_schema.replication_applier_status_by_worker\G`
- 在配置基本都对的情况，有的时候 Slave_SQL_Running 为 NO，很有可能是用户权限不够导致的，或者从库和主库的表结构不一致，可以将主库的内容整体复制过来，也可以只复制表结构
  ```
  // 手动复制主库
  mysqldump -h 主服务器IP -u用户名 -p密码 --all-databases > 
  backup.sql
  // nodata
  mysqldump -h 主服务器IP -u用户名 -p密码 --no-data --all- 
  databases > backup.sql
  // 指定某个database
  mysqldump -h 主服务器IP -u用户名 -p密码 -databaseName > 
  backup.sql
  // 导入从库
  mysql -h 从服务器IP -u用户名 -p密码 < backup.sql
  ```
- nginx 负载均衡的时候注意不要端口冲突
- 注意 nginx 的 default 配置，有的时候可能会占用 80 端口，可以将 default 文件删除，用我们自己的 conf 就好 `sudo rm /etc/nginx/sites-enabled/default`

