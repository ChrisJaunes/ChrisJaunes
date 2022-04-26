---
title: ZooKeeper-源码浅析-数据模型相关
date: 2022-04-26 15:36:28
categories: ["分布式", "ZooKeeper"]
tags:
- ZooKeeper
- 分布式
excerpt: 本文介绍了ZooKeeper的数据模型
---
## DataNode

---

考虑 ZooKeeper 的节点，ZK节点同时具有 目录和文件两种性质，而这要求我们维护: 子节点集合children 和 节点数据data

进一步需要维护节点的基本状态信息 stat

从安全性的角度出发，需要维护节点的访问权限 acl

为了进一步优化性能，考虑使用摘要 (digest 和 digestCached)

节点需要能够被序列化和被序列化，所以实现了Record接口

---

从线程安全的角度来讲，对于节点信息的相关操作需要加锁，包括对节点数据data、子节点集合children、状态信息stat等的修改、取值操作

子节点集合children默认访问可见性为private，类中相关函数addChild、removeChild、setChildren、getChildren均使用了synchronized进行修饰，而getChildren返回的是不可变类型，无暴露children的风险，因此children是线程安全的

节点数据data默认访问可见性为default，在package里直接操作 DataNode对象中 data的代码 都对 相应DataNode对象上了同步锁。但获取到data对象的线程都可以修改data，尝试修改源码的同学务必小心。不过ZK本身要求对于节点数据操作具有原子性，读操作将获取与节点相关的所有数据，写操作也将替换掉节点的所有数据，去修改data已经破坏了这个性质。

统计信息stat默认访问可见性为public，但获取到stat对象的线程都可以修改stat，尝试修改源码的同学务必小心

digest 相关代码并没有加锁, 使用了volatile来阻止CPU缓存

{% spoiler "DataNode 构造相关源码" %}
源码链接 [GitHub](https://github.com/apache/zookeeper/blob/master/zookeeper-server/src/main/java/org/apache/zookeeper/server/DataNode.java)
```java
public class DataNode implements Record {
    // 该节点的摘要值，根据路径、数据和统计信息计算得出该节点的摘要值
    private volatile long digest;
    // 指示此节点的摘要是否是最新的，用于优化性能
    volatile boolean digestCached;
    // datanode的数据
    byte[] data;
    // 该节点的acl被映射成一个long值, 在datatree中维护了一个映射关系表
    Long acl;
    // 该节点的统计信息, 可以被持久化到磁盘
    public StatPersisted stat;
    // 子节点集合 注意这个列表保持的子节点不包含父路径
    private Set<String> children = null;
    // 空集合
    private static final Set<String> EMPTY_SET = Collections.emptySet();
    DataNode() {}
    public DataNode(byte[] data, Long acl, StatPersisted stat) {
        this.data = data;
        this.acl = acl;
        this.stat = stat;
    }
    public synchronized boolean addChild(String child) {
        if (children == null) {
            children = new HashSet<String>(8);
        }
        return children.add(child);
    }
    public synchronized boolean removeChild(String child) {
        if (children == null) {
            return false;
        }
        return children.remove(child);
    }
    public synchronized void setChildren(HashSet<String> children) { this.children = children; }
    public synchronized Set<String> getChildren() {
        if (children == null) {
            return EMPTY_SET;
        }
        return Collections.unmodifiableSet(children);
    }
    public synchronized void copyStat(Stat to) {
        to.setAversion(stat.getAversion());
        to.setCtime(stat.getCtime());
        to.setCzxid(stat.getCzxid());
        to.setMtime(stat.getMtime());
        to.setMzxid(stat.getMzxid());
        to.setPzxid(stat.getPzxid());
        to.setVersion(stat.getVersion());
        to.setEphemeralOwner(getClientEphemeralOwner(stat));
        to.setDataLength(data == null ? 0 : data.length);
        int numChildren = 0;
        if (this.children != null) {
            numChildren = children.size();
        }
        to.setCversion(stat.getCversion() * 2 - numChildren);
        to.setNumChildren(numChildren);
    }
    private static long getClientEphemeralOwner(StatPersisted stat) {
        EphemeralType ephemeralType = EphemeralType.get(stat.getEphemeralOwner());
        if (ephemeralType != EphemeralType.NORMAL) {
            return 0;
        }
        return stat.getEphemeralOwner();
    }
    public synchronized void deserialize(InputArchive archive, String tag) throws IOException { ... }

    public synchronized void serialize(OutputArchive archive, String tag) throws IOException { ... }

    public boolean isDigestCached() { return digestCached; }

    public void setDigestCached(boolean digestCached) { this.digestCached = digestCached; }

    public long getDigest() { return digest; }

    public void setDigest(long digest) { this.digest = digest; }

    public synchronized byte[] getData() { return data; }
}
```
{% endspoiler %}

## Stat 和 StatPersisted

ZK节点需要一些统计信息，包括创建时的事务ID、最后一次修改的事务ID、创建时间等等信息，为了更好的维护这些信息，ZK把这部分抽象成了 Stat 和 StatParsisted 部分

Stat 和 StatPersisted 是由 ZooKeeper-jute 中 ZooKeeper-jute/src/main/resources/zookeeper.jute 生成的

{% spoiler "Stat 和 StatPersisted 相关定义" %}
源码链接 [GitHub](https://github.com/apache/zookeeper/blob/master/zookeeper-jute/src/main/resources/zookeeper.jute)
```jute
module org.apache.zookeeper.data {
    // information shared with the client
    class Stat {
        long czxid;      // created zxid
        long mzxid;      // last modified zxid
        long ctime;      // created
        long mtime;      // last modified
        int version;     // version
        int cversion;    // child version
        int aversion;    // acl version
        long ephemeralOwner; // owner id if ephemeral, 0 otw
        int dataLength;  //length of the data in the node
        int numChildren; //number of children of this node
        long pzxid;      // last modified children
    }
    // information explicitly stored by the server persistently
    class StatPersisted {
        long czxid;      // created zxid
        long mzxid;      // last modified zxid
        long ctime;      // created
        long mtime;      // last modified
        int version;     // version
        int cversion;    // child version
        int aversion;    // acl version
        long ephemeralOwner; // owner id if ephemeral, 0 otw
        long pzxid;      // last modified children
    }
}
```
{% endspoiler %}

jute 生成的代码：

{% spoiler "Stat 和 StatPersisted 相关源码" %}
Stat 和 StatPersisted 基本一样
```java
public class StatPersisted implements Record {
  private long czxid;
  private long mzxid;
  private long ctime;
  private long mtime;
  private int version;
  private int cversion;
  private int aversion;
  private long ephemeralOwner;
  private long pzxid;
  public StatPersisted() {}
  public StatPersisted(long czxid,long mzxid,long ctime,long mtime,int version,int cversion,int aversion,long ephemeralOwner,long pzxid) { ... }
  public long getXXX() { ... }
  public void setXXX(long m_) { ... }
  public void serialize(OutputArchive a_, String tag) throws java.io.IOException { ... }
  public void deserialize(InputArchive a_, String tag) throws java.io.IOException { ... }
  public String toString() { ... }
  public void write(java.io.DataOutput out) throws java.io.IOException { ... }
  public void readFields(java.io.DataInput in) throws java.io.IOException { ... }
  public int compareTo (Object peer_) throws ClassCastException { ... }
  public boolean equals(Object peer_) { ... }
  public int hashCode() { ... }
  public static String signature() { ... }
}
```
{% endspoiler %}

## NodeHashMap

为了提升性能，ZK采用了K-V结构，把节点路径作为Key、把其值作为Value，利用了哈希表去加快数据的存储、修改等操作，因此使用ZK必须使用绝对路径而不能采用相对路径

在 NodeHashMap 中规定了需要实现的接口
{% spoiler "NodeHashMap 相关源码" %}
源码链接 [GitHub](https://github.com/apache/zookeeper/blob/master/zookeeper-server/src/main/java/org/apache/zookeeper/server/NodeHashMap.java)
```java
public interface NodeHashMap {
    // Add the node into the map and update the digest with the new node.
    DataNode put(String path, DataNode node);
    // Add the node into the map without update the digest.
    DataNode putWithoutDigest(String path, DataNode node);
    // Return the data node associated with the path.
    DataNode get(String path);
    // Remove the path from the internal nodes map.
    DataNode remove(String path);
    // Return all the entries inside this map.
    Set<Map.Entry<String, DataNode>> entrySet();
    // Clear all the items stored inside this map.
    void clear();
    // Return the size of the nodes stored in this map.
    int size();
    // Called before we made the change on the node, which will clear the digest associated with it.
    void preChange(String path, DataNode node);
    // Called after making the changes on the node, which will update the digest.
    void postChange(String path, DataNode node);
    // Return the digest value.
    long getDigest();

}
```
{% endspoiler %}

NodeHashMapImpl 是 NodeHashMap 的一个实现

其核心是维护了一个 nodes， 这是一个线程安全的HashMap

其次维护了一个 AdHash 用来维护 digest， 具体作用待探究

{% spoiler "NodeHashMapImpl 相关源码" %}
源码链接 [GitHub](https://github.com/apache/zookeeper/blob/master/zookeeper-server/src/main/java/org/apache/zookeeper/server/NodeHashMapImpl.java)
```java
public class NodeHashMapImpl implements NodeHashMap {
    private final ConcurrentHashMap<String, DataNode> nodes;
    private final boolean digestEnabled;
    private final DigestCalculator digestCalculator;
    private final AdHash hash;

    public NodeHashMapImpl(DigestCalculator digestCalculator) {
        this.digestCalculator = digestCalculator;
        nodes = new ConcurrentHashMap<>();
        hash = new AdHash();
        digestEnabled = ZooKeeperServer.isDigestEnabled();
    }

    @Override
    public DataNode put(String path, DataNode node) {
        DataNode oldNode = nodes.put(path, node);
        addDigest(path, node);
        if (oldNode != null) {
            removeDigest(path, oldNode);
        }
        return oldNode;
    }

    @Override
    public DataNode putWithoutDigest(String path, DataNode node) { return nodes.put(path, node); }

    @Override
    public DataNode get(String path) { return nodes.get(path); }

    @Override
    public DataNode remove(String path) {
        DataNode oldNode = nodes.remove(path);
        if (oldNode != null) {
            removeDigest(path, oldNode);
        }
        return oldNode;
    }

    @Override
    public Set<Map.Entry<String, DataNode>> entrySet() { return nodes.entrySet(); }

    @Override
    public void clear() { 
        nodes.clear(); 
        hash.clear(); 
    }

    @Override
    public int size() { return nodes.size(); }

    @Override
    public void preChange(String path, DataNode node) { removeDigest(path, node); }

    @Override
    public void postChange(String path, DataNode node) { 
        node.digestCached = false; 
        addDigest(path, node);
    }

    private void addDigest(String path, DataNode node) {
        if (path.startsWith(ZooDefs.ZOOKEEPER_NODE_SUBTREE)) {
            return;
        }
        if (digestEnabled) {
            hash.addDigest(digestCalculator.calculateDigest(path, node));
        }
    }

    private void removeDigest(String path, DataNode node) {
        if (path.startsWith(ZooDefs.ZOOKEEPER_NODE_SUBTREE)) {
            return;
        }
        if (digestEnabled) {
            hash.removeDigest(digestCalculator.calculateDigest(path, node));
        }
    }

    @Override
    public long getDigest() { return hash.getHash(); }
}
```
{% endspoiler %}