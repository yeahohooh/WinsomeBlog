---
title: "Docker 搭建 Redis Cluster 集群"
date: 2024-11-01T18:40:15+08:00
draft: false
tags: 
  - Golang
author : winsome
scrolltotop : true
toc : true
mathjax : false
---

本文仅介绍 Redis Cluster 模式，也是官方推荐的模式。

为了保证高可用性，Redis 要求 Cluster 模式至少需要3个主节点，而每个主节点至少要配备一个从属节点，因此我们至少需要6个节点才能搭建起来一个Redis 的 Cluster 集群。

例子中都写到了同一个服务器，实际中可以拆开放到多个服务器，只需要把相应的内容分开就好。

#### 前期准备

**拉取 redis 镜像**
```
docker pull redis
```
**创建 Docker 容器网络**
主要是用于redis-cluster能于外界进行网络通信，一般常用桥接模式。
```
docker network create redis-cluster-net
```
**编写配置文件**
创建 Redis 宿主机配置文件
```
# 创建文件夹，7000 是将要设置的端口号，所以先用这个命名
sudo mkdir -p /data/redis7000/conf
# 创建配置文件
cd /data/redis7000/conf
sudo nano redis-cluster.conf
```
配置内容如下：
```
# 端口号
port 7000
# 后台运行，使用 docker 的话，这个参数要为 no，否则 docker run 后，会被停掉
daemonize no
# 开启集群模式
cluster-enabled yes
# 配置当集群中有一台机器宕机时集群保持可用
cluster-require-full-coverage no
# 开启aof持久化
appendonly yes
# 设置密码
requirepass 12345678
# 其它集群服务节点密码
masterauth 12345678
# 集群的配置 配置文件首次启动自动生成
cluster-config-file nodes.conf
# 请求超时
cluster-node-timeout 5000
# 服务器外网地址
cluster-announce-ip  <你的服务器外网ip>
# 外网端口
cluster-announce-port 7000
# 集群节点总线端口,节点之间互相通信，常规端口+10000，用port + 10000
cluster-announce-bus-port 17000
```
然后 ctl + o 保存，ctl + x 退出.
这些文件和文件夹要同时**创建 6 个**，之间的区别是端口号 7000...7005，还有要注意文件夹的名字。

#### 启动容器

**docker-compose 方式**

```
version: '3'

networks:
  redis-cluster-net:
    external:
      name: redis-cluster-net

services:
  redis-cluster-1:
    image: redis
    restart: always
    container_name: redis-cluster-1
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - /data/redis7000/conf/redis-cluster.conf:/usr/local/etc/redis/redis.conf
      - /data/redis7000/data:/data
    networks:
      - redis-cluster-net
    ports:
      - "7000:7000"
      - "17000:17000"

  redis-cluster-2:
    image: redis
    restart: always
    container_name: redis-cluster-2
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - /data/redis7001/conf/redis-cluster.conf:/usr/local/etc/redis/redis.conf
      - /data/redis7001/data:/data
    networks:
      - redis-cluster-net
    ports:
      - "7001:7001"
      - "17001:17001"

  redis-cluster-3:
    image: redis
    restart: always
    container_name: redis-cluster-3
    volumes:
      - /data/redis7002/conf/redis-cluster.conf:/usr/local/etc/redis/redis.conf
      - /data/redis7002/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net
    ports:
      - "7002:7002"
      - "17002:17002"

  redis-cluster-4:
    image: redis
    restart: always
    container_name: redis-cluster-4
    volumes:
      - /data/redis7003/conf/redis-cluster.conf:/usr/local/etc/redis/redis.conf
      - /data/redis7003/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net
    ports:
      - "7003:7003"
      - "17003:17003"

  redis-cluster-5:
    image: redis
    restart: always
    container_name: redis-cluster-5
    volumes:
      - /data/redis7004/conf/redis-cluster.conf:/usr/local/etc/redis/redis.conf
      - /data/redis7004/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net
    ports:
      - "7004:7004"
      - "17004:17004"

  redis-cluster-6:
    image: redis
    restart: always
    container_name: redis-cluster-6
    volumes:
      - /data/redis7005/conf/redis-cluster.conf:/usr/local/etc/redis/redis.conf
      - /data/redis7005/data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-cluster-net
    ports:
      - "7005:7005"
      - "17005:17005"
```
docker-compose.yml 文件编写完后直接 `docker-compose up -d` 启动容器。

**docker命令 方式**

也可以用 docker 命令，一个一个启动，**注意修改对应的端口号和文件路径**
```
docker run -it -d -p 7000:7000 -p 17000:17000 --name redis-cluster-1 --net redis-cluster-net -v /data/redis7000/conf/redis-cluster.conf:/usr/local/etc/redis/redis.conf -v /data/redis7000/data:/data redis redis-server /usr/local/etc/redis/redis.conf
```

#### 创建 Cluster 集群
使用 docker 指定任意一个容器，启动集群
```
# 前三个是主节点，后三个是从节点
# --cluster-replicas 1 表示： 1主1从
docker exec -it redis-cluster-1 redis-cli  -a 12345678 --cluster create <你的服务器ip>:7000 <你的服务器ip>:7001 <你的服务器ip>:7002 <你的服务器ip>:7003 <你的服务器ip>:7004 <你的服务器ip>:7005   --cluster-replicas 1
```
等待连接之后，就创建成功了。

然后可以进入容器，查看集群信息
```
# 进入容器
docker exec -it redis-cluster-1 /bin/bash
# 进入 redis
redis-cli -p 7000 -c -a 12345678
# 查看信息
cluster info
# 查看节点信息
cluster nodes
# 可以测试输入输出数据
set k1 haha
get k1
```
#### 加入新的节点
还是用 docker 创建新的容器，然后进入容器。

加入节点
```
redis-cli -a 12345678 --cluster add-node <new-ip>:new-port <old-ip>:old-port
```
执行重新分片
```
redis-cli -a 12345678 --cluster reshard <new-ip>:new-port --cluster-from oldxxxxxxxxxx --cluster-to yourxxxxxxxxxx --cluster-slots 4000
# 其中 xxx 是指其他节点的信息，比如 07e1b5c2827cf963623d491faeba3b9e24fc4550，可以用 cluster nodes 查看
cluster nodes
```


#### 可能产生的问题

记得查看服务器上的安全组，或者防火墙，有没有打开对应的端口，port 和 10000 + port，两个都要打开。

如果创建集群过程中，Waiting for the cluster to join ...... 等待过久，大概率是安全组或防火墙没打开。

#### 在 Go 中操作 Cluster

在 Go 中使用 go-redis 包来创建 Redis 集群客户端非常简单。下面是一个基本的使用示例，包括连接到 Redis 集群、设置和获取键值对的操作。

1. 安装 go-redis
```
go get github.com/go-redis/redis/v9
```
2. 创建 Redis 集群客户端
```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/go-redis/redis/v9"
)

func main() {
    ctx := context.Background()

    // 创建 Redis 集群客户端
    clusterClient := redis.NewClusterClient(&redis.ClusterOptions{
        Addrs: []string{
            "192.168.1.1:7000", // 节点地址
            "192.168.1.2:7000", // 节点地址
            "192.168.1.3:7000", // 节点地址
        },
        Password: "", // 如果有密码，填写这里
    })

    // 测试连接
    _, err := clusterClient.Ping(ctx).Result()
    if err != nil {
        log.Fatalf("Could not connect to Redis cluster: %v", err)
    }
    fmt.Println("Connected to Redis cluster!")

    // 设置一个键值对
    err = clusterClient.Set(ctx, "key", "value", 0).Err()
    if err != nil {
        log.Fatalf("Could not set value: %v", err)
    }
    fmt.Println("Key set successfully!")

    // 获取键的值
    value, err := clusterClient.Get(ctx, "key").Result()
    if err != nil {
        log.Fatalf("Could not get value: %v", err)
    }
    fmt.Printf("The value of 'key' is: %s\n", value)

    // 关闭客户端
    err = clusterClient.Close()
    if err != nil {
        log.Fatalf("Could not close the cluster client: %v", err)
    }
}
```

