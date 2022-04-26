---
title: ZooKeeper-环境搭建
date: 2022-04-25 19:21:39
categories: ["分布式", "ZooKeeper"]
tags: 
- ZooKeeper
- 分布式
excerpt: 本文介绍了如何搭建ZK集群
---

## 伪集群搭建

### 安装环境

下述环境为 

1. Ubuntu 20 LTS
    Linux version 5.15.0-25-generic (buildd@ubuntu) (gcc (Ubuntu 11.2.0-19ubuntu1) 11.2.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #25-Ubuntu SMP Wed Mar 30 15:54:22 UTC 2022
2. Java
    openjdk version "11.0.14.1" 2022-02-08

### ZooKeeper 下载

ZooKeeper下载链接：https://zookeeper.apache.org/releases.html 

在下载页面分为最新的Release版本和最近的稳定Release版本

本文采用的Zookeeper版本为 (Apache ZooKeeper 3.8.0) 

下载地址：https://dlcdn.apache.org/zookeeper/zookeeper-3.8.0/apache-zookeeper-3.8.0-bin.tar.gz

### 伪集群目录结构

建立 zk-demo 目录，把 apache-zookeeper-3.8.0-bin.tar.gz 放到 zk-demo 目录下面

在 zk-demo 打开 Terminal

```shell
$ tar -xzvf apache-zookeeper-3.8.0-bin.tar.gz
$ mkdir zk1 zk2 zk3
$ cp -R apache-zookeeper-3.8.0-bin zk1/
$ cp -R apache-zookeeper-3.8.0-bin zk2/
$ cp -R apache-zookeeper-3.8.0-bin zk3/
$ cd zk1
$ mkdir data logs
$ echo "1" >> data/myid
$ cd ../zk2
$ mkdir data logs
$ echo "2" >> data/myid
$ cd ../zk3
$ mkdir data logs
$ echo "3" >> data/myid
```

在 zk-demo 下的目录结构

```shell
.
├── apache-zookeeper-3.8.0-bin.tar.gz
├── apache-zookeeper-3.8.0-bin
├── zk1
│   ├── apache-zookeeper-3.8.0-bin
│   ├── data
│   │   ├── myid
│   └── logs
├── zk2
│   ├── apache-zookeeper-3.8.0-bin
│   ├── data
│   │   ├── myid
│   └── logs
└── zk3
    ├── apache-zookeeper-3.8.0-bin
    ├── data
    │   ├── myid
    └── logs
```

### ZooKeeper 配置

在 apache-zookeeper-3.8.0-bin 中 conf 下存在 zoo_sample.cfg 样例文件

在 apache-zookeeper-3.8.0-bin 中 conf 下创建 zoo.cfg

注意 dataDir、dataLogDir、clientProt 对于每个 ZK实例 是不同的

ZK1 对应 的 zoo.cfg
```text
tickTime=2000  # 单位时间，其他时间都是以这个倍数来表示
initLimit=10   # 节点初始化时间，10倍单位时间(即十倍tickTime)
syncLimit=5    # 心跳最大延迟周期
dataDir=/home/zk-demo/zk1/data     # 该实例对应的数据目录（上文步骤创建）
dataLogDir=/home/zk-demo/zk1/logs  # 该实例对应的日志目录（上文步骤创建）
clientPort=2181                              # 端口（每个实例不同）
server.1=127.0.0.1:8881:7771                 # server.id=host:port:port
server.2=127.0.0.1:8882:7772                 # server.id=host:port:port
server.3=127.0.0.1:8883:7773                 # server.id=host:port:port
```
ZK2 对应 的 zoo.cfg
```text
tickTime=2000  # 单位时间，其他时间都是以这个倍数来表示
initLimit=10   # 节点初始化时间，10倍单位时间(即十倍tickTime)
syncLimit=5    # 心跳最大延迟周期
dataDir=/home/zk-demo/zk2/data     # 该实例对应的数据目录（上文步骤创建）
dataLogDir=/home/zk-demo/zk2/logs  # 该实例对应的日志目录（上文步骤创建）
clientPort=2182                              # 端口（每个实例不同）
server.1=127.0.0.1:8881:7771                 # server.id=host:port:port
server.2=127.0.0.1:8882:7772                 # server.id=host:port:port
server.3=127.0.0.1:8883:7773                 # server.id=host:port:port
```
ZK3 对应 的 zoo.cfg
```text
tickTime=2000  # 单位时间，其他时间都是以这个倍数来表示
initLimit=10   # 节点初始化时间，10倍单位时间(即十倍tickTime)
syncLimit=5    # 心跳最大延迟周期
dataDir=/home/zk-demo/zk3/data     # 该实例对应的数据目录（上文步骤创建）
dataLogDir=/home/zk-demo/zk3/logs  # 该实例对应的日志目录（上文步骤创建）
clientPort=2183                              # 端口（每个实例不同）
server.1=127.0.0.1:8881:7771                 # server.id=host:port:port
server.2=127.0.0.1:8882:7772                 # server.id=host:port:port
server.3=127.0.0.1:8883:7773                 # server.id=host:port:port
```

### ZooKeeper 启动

在 zk-demo 中打开 Terminal
```shell
$ ./zk1/apache-zookeeper-3.8.0-bin/bin/zkServer.sh start
$ ./zk2/apache-zookeeper-3.8.0-bin/bin/zkServer.sh start
$ ./zk3/apache-zookeeper-3.8.0-bin/bin/zkServer.sh start
```

出现了 "Starting zookeeper ... FAILED TO START" 请检查配置文件，可以对应实例的 apache-zookeeper-3.8.0-bin\logs 文件夹的相关文件

### ZooKeeper 连接

使用 zkCli 连接集群
``` shell
/home/zk-demo/zk1/apache-zookeeper-3.8.0-bin/bin/zkCli.sh -server 127.0.0.1:2182
```