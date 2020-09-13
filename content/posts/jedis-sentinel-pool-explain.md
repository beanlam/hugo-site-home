---
title: "Jedis的JedisSentinelPool源代码分析"
date: 2015-04-20T18:50:36+08:00
categories : ["redis"]
---


# 概述

Jedis是Redis官方推荐的Java客户端，更多Redis的客户端可以参考Redis官网[客户端列表](http://redis.io/clients)。Redis-Sentinel作为官方推荐的HA解决方案，Jedis也在客户端角度实现了对Sentinel的支持，主要实现在`JedisSentinelPool.java`这个类中，下文会分析这个类的实现。

# 属性

JedisSentinelPool类里有以下的属性：

```java
//基于apache的commom-pool2的对象池配置
protected GenericObjectPoolConfig poolConfig;

//超时时间，默认是2000
protected int timeout = Protocol.DEFAULT_TIMEOUT;

//sentinel的密码
protected String password;

//redis数据库的数目
protected int database = Protocol.DEFAULT_DATABASE;

//master监听器，当master的地址发生改变时，会触发这些监听者
protected Set<MasterListener> masterListeners = new HashSet<MasterListener>();

protected Logger log = Logger.getLogger(getClass().getName());

//Jedis实例创建工厂
private volatile JedisFactory factory;

//当前的master，HostAndPort是一个简单的包装了ip和port的模型类
private volatile HostAndPort currentHostMaster;

```

# 构造器

构造器的代码如下：
```java
public JedisSentinelPool(String masterName, Set<String> sentinels, final GenericObjectPoolConfig poolConfig, int timeout, final String password, final int database) {

    this.poolConfig = poolConfig;
    this.timeout = timeout;
    this.password = password;
    this.database = database;

    HostAndPort master = initSentinels(sentinels, masterName);
    initPool(master);
}
```
构造器一开始对实例变量进行赋值，参数sentinels是客户端所需要打交道的Redis-Sentinel，允许有多个，用一个集合来盛装。

然后通过initSentinels方法与sentinel沟通后，确定当前sentinel所监视的master是哪一个。然后通过master来创建好对象池，以便后续从对象池中取出一个Jedis实例，来对master进行操作。

# initSentinels方法

initSentinels方法的代码如下所示，我加了一些注释：

```java
private HostAndPort initSentinels(Set<String> sentinels, final String masterName) {

    HostAndPort master = null;
    boolean sentinelAvailable = false;

    log.info("Trying to find master from available Sentinels...");

    // 有多个sentinels,遍历这些个sentinels
    for (String sentinel : sentinels) {
        // host:port表示的sentinel地址转化为一个HostAndPort对象。
        final HostAndPort hap = toHostAndPort(Arrays.asList(sentinel.split(":")));

        log.fine("Connecting to Sentinel " + hap);

        Jedis jedis = null;
        try {
            // 连接到sentinel
            jedis = new Jedis(hap.getHost(), hap.getPort());

            // 根据masterName得到master的地址，返回一个list，host= list[0], port =
            // list[1]
            List<String> masterAddr = jedis.sentinelGetMasterAddrByName(masterName);

            // connected to sentinel...
            sentinelAvailable = true;

            if (masterAddr == null || masterAddr.size() != 2) {
                log.warning("Can not get master addr, master name: " + masterName
                        + ". Sentinel: " + hap + ".");
                continue;
            }

            master = toHostAndPort(masterAddr);
            log.fine("Found Redis master at " + master);
            // 如果在任何一个sentinel中找到了master，不再遍历sentinels
            break;
        } catch (JedisConnectionException e) {
            log.warning("Cannot connect to sentinel running @ " + hap
                    + ". Trying next one.");
        } finally {
            // 关闭与sentinel的连接
            if (jedis != null) {
                jedis.close();
            }
        }
    }

    // 到这里，如果master为null，则说明有两种情况，一种是所有的sentinels节点都down掉了，一种是master节点没有被存活的sentinels监控到
    if (master == null) {
        if (sentinelAvailable) {
            // can connect to sentinel, but master name seems to not
            // monitored
            throw new JedisException("Can connect to sentinel, but " + masterName
                    + " seems to be not monitored...");
        } else {
            throw new JedisConnectionException(
                    "All sentinels down, cannot determine where is " + masterName
                            + " master is running...");
        }
    }

    //如果走到这里，说明找到了master的地址
    log.info("Redis master running at " + master + ", starting Sentinel listeners...");

    //启动对每个sentinels的监听
    for (String sentinel : sentinels) {
        final HostAndPort hap = toHostAndPort(Arrays.asList(sentinel.split(":")));
        MasterListener masterListener = new MasterListener(masterName, hap.getHost(),
                hap.getPort());
        masterListeners.add(masterListener);
        masterListener.start();
    }

    return master;
}

```

可以看到`initSentinels`方法的参数有一个masterName，就是我们所需要查找的master的名字。
一开始，遍历多个sentinels，一个一个连接到sentinel，去询问关于masterName的消息，可以看到是通过`jedis.sentinelGetMasterAddrByName()`方法去连接sentinel，并询问当前的master的地址。点进这个方法去看看，源代码是这样写的：

```java
/**
   * <pre>
   * redis 127.0.0.1:26381> sentinel get-master-addr-by-name mymaster
   * 1) "127.0.0.1"
   * 2) "6379"
   * </pre>
   * @param masterName
   * @return two elements list of strings : host and port.
   */
  public List<String> sentinelGetMasterAddrByName(String masterName) {
    client.sentinel(Protocol.SENTINEL_GET_MASTER_ADDR_BY_NAME, masterName);
    final List<Object> reply = client.getObjectMultiBulkReply();
    return BuilderFactory.STRING_LIST.build(reply);
  }
```
调用的是与Jedis绑定的client去发送一个**"get-master-addr-by-name"**命令。

回到`initSentinels`方法中，如果没有询问到master的地址，那就询问下一个sentinel。如果询问到了master的地址，那么将不再遍历sentinel集合，直接break退出循环遍历。

如果循环结束后，master的值为null,那么有两种可能：

* 一种是所有的sentinel实例都不可用了
* 另外一种是，sentinel实例有可用的，但是没有监控名字为masterName的Redis。

如果master为null，程序会抛出异常，不再往下走了。如果master不为null呢，继续往下走。

可以从代码中看到，为每个sentinel都启动了一个监听者`MasterListener`。MasterListener本身是一个线程，它会去订阅sentinel上关于master节点地址改变的消息。

接下来先分析构造方法中的另外一个方法：`initPool`。之后再看MasterListener的实现。

# initPool方法

initPool的实现源代码如下所示：

```java
private void initPool(HostAndPort master) {
        if (!master.equals(currentHostMaster)) {
            currentHostMaster = master;
            if (factory == null) {
                factory = new JedisFactory(master.getHost(), master.getPort(), timeout,
                        password, database);
                initPool(poolConfig, factory);
            } else {
                factory.setHostAndPort(currentHostMaster);
                // although we clear the pool, we still have to check the
                // returned object
                // in getResource, this call only clears idle instances, not
                // borrowed instances
                internalPool.clear();
            }

            log.info("Created JedisPool to master at " + master);
        }
    }
    
```

可以看到，作为参数传进来的master会与实例变量currentHostMaster作比较，看看是否是相同的，为什么要作这个比较呢，因为前文中提到的`MasterListener`会在发现master地址改变以后，去调用`initPool`方法。
如果是第一次调用`initPool`方法(构造函数中调用)，那么会初始化Jedis实例创建工厂，如果不是第一次调用(`MasterListener`中调用)，那么只对已经初始化的工厂进行重新设置。
从以上也可以看出为什么`currentHostMaster`和`factory`这两个变量为什么要声明为`volatile`，它们会在多线程环境下被访问和修改，因此必须保证**可见性**。
第一次调用时，会调用`initPool(poolConfig, factory)`方法。
看看这个方法的源代码：
```java
public void initPool(final GenericObjectPoolConfig poolConfig,
        PooledObjectFactory<T> factory) {

    if (this.internalPool != null) {
        try {
            closeInternalPool();
        } catch (Exception e) {
        }
    }

    this.internalPool = new GenericObjectPool<T>(factory, poolConfig);
}
```

基本上只干了一件事：初始化内部对象池。

# MasterListener监听者线程

直接看它的run方法实现吧：

```java


public void run() {

    running.set(true);

    while (running.get()) {

        j = new Jedis(host, port);

        try {
            //订阅sentinel上关于master地址改变的消息
            j.subscribe(new JedisPubSub() {
                @Override
                public void onMessage(String channel, String message) {
                    log.fine("Sentinel " + host + ":" + port + " published: "
                            + message + ".");

                    String[] switchMasterMsg = message.split(" ");

                    if (switchMasterMsg.length > 3) {

                        if (masterName.equals(switchMasterMsg[0])) {
                            initPool(toHostAndPort(Arrays.asList(
                                    switchMasterMsg[3], switchMasterMsg[4])));
                        } else {
                            log.fine("Ignoring message on +switch-master for master name "
                                    + switchMasterMsg[0]
                                    + ", our master name is " + masterName);
                        }

                    } else {
                        log.severe("Invalid message received on Sentinel " + host
                                + ":" + port + " on channel +switch-master: "
                                + message);
                    }
                }
            }, "+switch-master");

        } catch (JedisConnectionException e) {

            if (running.get()) {
                log.severe("Lost connection to Sentinel at " + host + ":" + port
                        + ". Sleeping 5000ms and retrying.");
                try {
                    Thread.sleep(subscribeRetryWaitTimeMillis);
                } catch (InterruptedException e1) {
                    e1.printStackTrace();
                }
            } else {
                log.fine("Unsubscribing from Sentinel at " + host + ":" + port);
            }
        }
    }
}

```

可以看到它依然委托了Jedis去与sentinel打交道，订阅了关于master地址变换的消息，当master地址变换时，就会再调用一次`initPool`方法，重新设置对象池相关的设置。

# 尾声
Jedis的JedisSentinelPool的实现仅仅适用于单个master-slave。
现在有了更多的需求，既需要sentinel提供的自动主备切换机制，又需要客户端能够做数据分片(Sharding)，类似于memcached用一致性哈希进行数据分片。

