---
title: "Jedis的Sharded源代码分析"
date: 2015-04-20T18:53:03+08:00
categories : ["redis"]
---


# 概述

Jedis是Redis官方推荐的Java客户端，更多Redis的客户端可以参考Redis官网[客户端列表](http://redis.io/clients)。当业务的数据量非常庞大时，需要考虑将数据存储到多个缓存节点上，如何定位数据应该存储的节点，一般用的是一致性哈希算法。Jedis在客户端角度实现了一致性哈希算法，对数据进行分片，存储到对应的不同的redis实例中。
Jedis对Sharded的实现主要是在`ShardedJedis.java`和`ShardedJedisPool.java`中。本文主要介绍ShardedJedis的实现，ShardedJedisPool是基于apache的common-pool2的对象池实现。


# 继承关系

ShardedJedis--->BinaryShardedJedis--->Sharded &lt;Jedis, JedisShardInfo>

# 构造函数

查看其构造函数
```java
public ShardedJedis(List<JedisShardInfo> shards, Hashing algo, Pattern keyTagPattern) {
        super(shards, algo, keyTagPattern);
    }
```

构造器参数解释：

* shards是一个JedisShardInfo的列表，一个JedisShardedInfo类代表一个数据分片的主体。

* algo是用来进行数据分片的算法

* keyTagPattern，自定义分片算法所依据的key的形式。例如，可以不针对整个key的字符串做哈希计算，而是类似对**thisisa{key}**中包含在大括号内的字符串进行哈希计算。

**`JedisShardInfo`**是什么样的？
```java
public class JedisShardInfo extends ShardInfo<Jedis> {

  public String toString() {
    return host + ":" + port + "*" + getWeight();
  }

  private int connectionTimeout;
  private int soTimeout;
  private String host;
  private int port;
  private String password = null;
  private String name = null;
  // Default Redis DB
  private int db = 0;

  public String getHost() {
    return host;
  }

  public int getPort() {
    return port;
  }

  public JedisShardInfo(String host) {
    super(Sharded.DEFAULT_WEIGHT);
    URI uri = URI.create(host);
    if (JedisURIHelper.isValid(uri)) {
      this.host = uri.getHost();
      this.port = uri.getPort();
      this.password = JedisURIHelper.getPassword(uri);
      this.db = JedisURIHelper.getDBIndex(uri);
    } else {
      this.host = host;
      this.port = Protocol.DEFAULT_PORT;
    }
  }

  public JedisShardInfo(String host, String name) {
    this(host, Protocol.DEFAULT_PORT, name);
  }

  public JedisShardInfo(String host, int port) {
    this(host, port, 2000);
  }

  public JedisShardInfo(String host, int port, String name) {
    this(host, port, 2000, name);
  }

  public JedisShardInfo(String host, int port, int timeout) {
    this(host, port, timeout, timeout, Sharded.DEFAULT_WEIGHT);
  }

  public JedisShardInfo(String host, int port, int timeout, String name) {
    this(host, port, timeout, timeout, Sharded.DEFAULT_WEIGHT);
    this.name = name;
  }

  public JedisShardInfo(String host, int port, int connectionTimeout, int soTimeout, int weight) {
    super(weight);
    this.host = host;
    this.port = port;
    this.connectionTimeout = connectionTimeout;
    this.soTimeout = soTimeout;
  }

  public JedisShardInfo(String host, String name, int port, int timeout, int weight) {
    super(weight);
    this.host = host;
    this.name = name;
    this.port = port;
    this.connectionTimeout = timeout;
    this.soTimeout = timeout;
  }

  public JedisShardInfo(URI uri) {
    super(Sharded.DEFAULT_WEIGHT);
    if (!JedisURIHelper.isValid(uri)) {
      throw new InvalidURIException(String.format(
        "Cannot open Redis connection due invalid URI. %s", uri.toString()));
    }

    this.host = uri.getHost();
    this.port = uri.getPort();
    this.password = JedisURIHelper.getPassword(uri);
    this.db = JedisURIHelper.getDBIndex(uri);
  }

@Override
  public Jedis createResource() {
    return new Jedis(this);
  }
    /**
    *    省略setters和getters
    **/
}
```

可见JedisShardInfo包含了一个redis节点ip地址，端口号，name，密码等等相关信息。要构造一个ShardedJedis，提供一个或多个JedisShardInfo。

最终构造函数的实现在其父类`Sharded`里面

```java
public Sharded(List<S> shards, Hashing algo, Pattern tagPattern) {
        this.algo = algo;
        this.tagPattern = tagPattern;
        initialize(shards);
    }
```

# 哈希环的初始化
Sharded类里面维护了一个TreeMap，基于红黑树实现，用来盛放经过一致性哈希计算后的redis节点，另外维护了一个LinkedHashMap，用来保存ShardInfo与Jedis实例的对应关系。
**定位的流程如下**
先在TreeMap中找到对应key所对应的ShardInfo，然后通过ShardInfo在LinkedHashMap中找到对应的Jedis实例。

Sharded类对这些实例变量的定义如下所示：

```java
public static final int DEFAULT_WEIGHT = 1;
    private TreeMap<Long, S> nodes;
    private final Hashing algo;
    private final Map<ShardInfo<R>, R> resources = new LinkedHashMap<ShardInfo<R>, R>();

    /**
     * The default pattern used for extracting a key tag. The pattern must have
     * a group (between parenthesis), which delimits the tag to be hashed. A
     * null pattern avoids applying the regular expression for each lookup,
     * improving performance a little bit is key tags aren't being used.
     */
    private Pattern tagPattern = null;
    // the tag is anything between {}
    public static final Pattern DEFAULT_KEY_TAG_PATTERN = Pattern.compile("\\{(.+?)\\}");
```

接下来看其构造函数中的initialize方法

```java
private void initialize(List<S> shards) {
        nodes = new TreeMap<Long, S>();

        for (int i = 0; i != shards.size(); ++i) {
            final S shardInfo = shards.get(i);
            if (shardInfo.getName() == null)
                for (int n = 0; n < 160 * shardInfo.getWeight(); n++) {
                    nodes.put(this.algo.hash("SHARD-" + i + "-NODE-" + n), shardInfo);
                }
            else
                for (int n = 0; n < 160 * shardInfo.getWeight(); n++) {
                    nodes.put(
                            this.algo.hash(shardInfo.getName() + "*"
                                    + shardInfo.getWeight() + n), shardInfo);
                }
            resources.put(shardInfo, shardInfo.createResource());
        }
    }
```

可以看到，它对每一个ShardInfo通过一定规则计算其哈希值，然后存到TreeMap中，这里它实现了一致性哈希算法中虚拟节点的概念，因为我们可以看到同一个ShardInfo不止一次被放到TreeMap中，数量是，权重*160。
增加了虚拟节点的一致性哈希有很多好处，能避免数据在redis节点间分布不均匀。

然后，在LinkedHashMap中放入ShardInfo以及其对应的Jedis实例，通过调用其自身的createSource()来得到jedis实例。

# 数据定位

从ShardedJedis的代码中可以看到，无论进行什么操作，都要先根据key来找到对应的Redis，然后返回一个可供操作的Jedis实例。

例如其set方法：

```java
public String set(String key, String value) {
        Jedis j = getShard(key);
        return j.set(key, value);
    }
```

而getShard方法则在Sharded.java中实现，其源代码如下所示：

```java
public R getShard(byte[] key) {
        return resources.get(getShardInfo(key));
    }

    public R getShard(String key) {
        return resources.get(getShardInfo(key));
    }

    public S getShardInfo(byte[] key) {
        SortedMap<Long, S> tail = nodes.tailMap(algo.hash(key));
        if (tail.isEmpty()) {
            return nodes.get(nodes.firstKey());
        }
        return tail.get(tail.firstKey());
    }

    public S getShardInfo(String key) {
        return getShardInfo(SafeEncoder.encode(getKeyTag(key)));
    }

```
可以看到，先通过getShardInfo方法从TreeMap中获得对应的ShardInfo，然后根据这个ShardInfo就能够再LinkedHashMap中获得对应的Jedis实例了。