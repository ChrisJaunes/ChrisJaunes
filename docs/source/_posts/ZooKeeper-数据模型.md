---
title: ZooKeeper-数据模型
date: 2022-04-25 20:54:39
categories: ["分布式", "ZooKeeper"]
tags:
- ZooKeeper
- 分布式
excerpt: 本文介绍了ZooKeeper的数据模型
---
## Z-node 数据节点 [^1]

ZooKeeper的数据节点可以视为树状结构，树中的各节点被称为Z-node（即ZooKeeper node）

一个Z-node可以有多个子节点，ZooKeeper中的所有存储的数据是由 Z-node 组成，并以 key/value 形式存储数据。

整体结构类似于 linux 文件系统的模式以树形结构存储，其中根路径以 / 开头， 但也有自己的特性：
    1. Z-node 兼具文件和目录特点 既像文件一样维护着数据、信息、时间戳等数据，又像目录一样可以作为路径标识的一部分，并可以具有子 Z-node。用户对 Z-node 具有增、删、改、查等操作 
    2. Z-node 具有原子性操作 读操作将获取与节点相关的所有数据，写操作也将替换掉节点的所有数据
    3. Z-node 存储数据大小有限制 每个Znode的数据大小至多1M，但是常规使用中应该远小于此值
    4. Z-node 通过路径引用 如同Unix中的文件路径。路径必须是绝对的，因此他们必须由斜杠字符来开头。除此以外，他们必须是唯一的，也就是说每一个路径只有一个表示，因此这些路径不能改变

Z-node 路径名称要求， 任何 unicode 字符都可以在受以下约束的路径中使用：
    1. 空字符 (\u0000) 不能是路径名的一部分 (这会导致 C 绑定出现问题)
    2. 不能使用 (\u0001 - \u001F 和 \u007F \u009F) 字符 (它们显示不好或者呈现方式混乱)
    3. 不允许使用 (\ud800 - uF8FF、\uFFF0 - uFFFF) 字符
    4. 这 "." 字符可以用作另一个名称的一部分，但 "."和".." 不能单独用于指示路径上的节点 (因为 ZooKeeper 不使用相对路径, 类型(“/a/b/./c”或“/a/b/../c”)内容无效)
    5. "zookeeper"是保留字

## Z-node 节点类型

Znode有两种，分别为临时节点和永久节点，节点的类型在创建时即被确定，并且不能改变

永久节点：该节点的生命周期不依赖于会话。一旦将该节点创建为持久节点，该节点数据会一直存储在 ZooKeeper 服务器，即使创建该节点的客户端预服务端的会话关闭了，该节点依然不会被删除，只有在客户端显式执行删除操作的时候，才能被删除。Znode还有一个序列化的特性。

临时节点：该节点的生命周期依赖于创建它们的会话。如果将节点创建为临时节点，该节点数据不会一直存储在 ZooKeeper 服务器，当创建该临时节点的客户端会话因超时或发生异常而关闭时，该节点也相应在ZooKeeper服务器上被删除，当然也可以手动删除。临时节点不允许拥有子节点。 

有序特性：如果创建的时候指定的话，该Znode的名字后面会自动追加一个单调递增的序列号。序列号对于此节点的父节点来说是唯一的，这样便会记录每个子节点创建的先后顺序。

因此组合之后，Znode有四种节点类型： 
    PERSISTENT：永久节点 
    EPHEMERAL：临时节点 
    PERSISTENT_SEQUENTIAL：永久顺序节点 
    EPHEMERAL_SEQUENTIAL：临时顺序节点 

## Z-node 节点状态结构 [^2]

由此每个节点维护这些内容：一个二进制数组(byte data[]，用来存储节点的数据)、ACL访问控制信息、子节点数据（临时节点不允许子节点的，子节点字段为null)、记录自身状态信息的字段stat。

|状态属性|说明|
|---|---|
|czxid|表示该节点被创建时的事务ID|
|mzxid|表示该节点最后一次被更新的事务ID|
|pzxid|表示该节点的子节点列表最后一次被修改时的事务ID|
|ctime|表示该节点的创建时间(单位：毫秒)|
|mtime|表示该节点最后一次被更新的时间(单位：毫秒)|
|version|数据节点的版本号|
|cversion|子节点的版本号|
|aversion|节点的ACL版本号|
|ephemeralOwner|创建该临时节点的会话SessionID，如果该节点时持久节点, 那么这个属性值是0|
|dataLength|数据内容的长度|
|numChildren|当前节点的子节点个数|

### Z-node 节点版本

在 ZooKeeper 中为数据节点引入了版本的概念，每个数据节点有 3 种类型的版本信息，对数据节点的任何更新操作都会引起版本号的变化。ZooKeeper 的版本信息表示的是对节点数据内容、子节点信息或者是 ACL 信息的修改次数。

这是因为 ZooKeeper 大多是应用场景是定位数据模型上的节点，并在相关节点上进行操作。像这种查找与给定值相等的记录问题最适合用散列来解决。因此 ZooKeeper 在底层实现的时候，使用了一个 hashtable，即 hashtableConcurrentHashMap<String, DataNode> nodes ，用节点的完整路径来作为 key 存储节点数据。这样就大大提高了 ZooKeeper 的性能。

参考连接：

[^1]: The ZooKeeper Data Model [参考连接](https://zookeeper.apache.org/doc/r3.6.0/zookeeperProgrammers.html#:~:text=ZooKeeper%20Basic%20Operations.-,The%20ZooKeeper%20Data%20Model,-ZooKeeper%20has%20a)

[^2]: ZooKeeper Stat Structure [参考连接](https://zookeeper.apache.org/doc/r3.6.0/zookeeperProgrammers.html#:~:text=and%20znode%20modification.-,ZooKeeper%20Stat%20Structure,-The%20Stat%20structure)

[^3]: ZooKeeper 数据模型 节点的特性与应用 [参考连接](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/ZooKeeper%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%8E%E5%AE%9E%E6%88%98-%E5%AE%8C/01%20ZooKeeper%20%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B%EF%BC%9A%E8%8A%82%E7%82%B9%E7%9A%84%E7%89%B9%E6%80%A7%E4%B8%8E%E5%BA%94%E7%94%A8.md)


