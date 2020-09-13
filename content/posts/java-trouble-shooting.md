---
title: "Java Trouble Shooting"
date: 2017-06-06T18:33:55+08:00
categories : ["java"]
---


# 什么是线程栈(thread dump)
线程栈是某个时间点，JVM所有线程的活动状态的一个汇总；通过线程栈，可以查看某个时间点，各个线程正在做什么，通常使用线程栈来定位软件运行时的各种问题，例如 CPU 使用率特别高，或者是响应很慢，性能大幅度下滑。


线程栈包含了多个线程的活动信息，一个线程的活动信息通常看起来如下所示：
```bash
"main" prio=10 tid=0x00007faac0008800 nid=0x9f0 waiting on condition [0x00007faac6068000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at ThreadDump.main(ThreadDump.java:4)
```

这条线程的线程栈信息包含了以下这些信息：
- 线程的名字：其中 **main** 就是线程的名字，需要注意的是，当使用 `Thread` 类来创建一条线程，并且没有指定线程的名字时，这条线程的命名规则为 **Thread-i**，i 代表数字。如果使用 `ThreadFactory` 来创建线程，则线程的命名规则为 **pool-i-thread-j**，i 和 j 分别代表数字。
- 线程的优先级：**prio=10** 代表线程的优先级为 10
- 线程 id：**tid=0x00007faac0008800** 代表线程 id 为 0x00007faac0008800，而** nid=0x9f0** 代表该线程对应的操作系统级别的线程 id。所谓的 nid，换种说法就是 native id。在操作系统中，分为内核级线程和用户级线程，JVM 的线程是用户态线程，内核不知情，但每一条 JVM 的线程都会映射到操作系统一条具体的线程
- 线程的状态：**java.lang.Thread.State: TIMED_WAITING (sleeping)** 以及 **waiting on condition** 代表线程当前的状态
- 线程占用的内存地址：**[0x00007faac6068000]** 代表当前线程占用的内存地址
- 线程的调用栈：**at java.lang.Thread.sleep(Native Method)** 以及它之后的相类似的信息，代表线程的调用栈

# 回顾线程状态

![线程状态](/assets/java-trouble-shooting/thread-state.png)

- NEW：线程初创建，未运行
- RUNNABLE：线程正在运行，但**不一定消耗 CPU**
- BLOCKED：线程正在等待另外一个线程释放锁
- WAITING：线程执行了 `wait, join, park` 方法
- TIMED_WAITING：线程调用了`sleep, wait, join, park` 方法，与 WAITING 状态不同的是，这些方法带有表示时间的参数。

例如以下代码：
```java
public static void main(String[] args) throws InterruptedException {
        int sum = 0;
        while (true) {
            int i = 0;
            int j = 1;
            sum = i + j;
        }
}
```
main 线程对应的线程栈就是
```bash
"main" prio=10 tid=0x00007fe1b4008800 nid=0x1292 runnable [0x00007fe1bd88f000]
   java.lang.Thread.State: RUNNABLE
        at ThreadDump.main(ThreadDump.java:7)
```
其状态为 RUNNABLE

如果是以下代码，两个线程会竞争同一个锁，其中只有一个线程能获得锁，然后进行 sleep(time)，从而进入 TIMED_WAITING 状态，另外一个线程由于等待锁，会进入 BLOCKED 状态。
```java
    public static void main(String[] args) throws InterruptedException {

        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                try {
                    fun1();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        t1.setDaemon(false);
        t1.setName("MyThread1");
        
        Thread t2 = new Thread(new Runnable() {
            
            @Override
            public void run() {
                try {
                    fun2();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        t2.setDaemon(false);
        t2.setName("MyThread2");
        t1.start();
        t2.start();
        */
        
    }

    private static synchronized void fun1() throws InterruptedException {
        System.out.println("t1 acquire");
        Thread.sleep(Integer.MAX_VALUE);
    }

    private static synchronized void fun2() throws InterruptedException {
        System.out.println("t2 acquire");
        Thread.sleep(Integer.MAX_VALUE);
    }
```
对应的线程栈为：
```bash
"MyThread2" prio=10 tid=0x00007ff1e40b1000 nid=0x12eb waiting for monitor entry [0x00007ff1e07f6000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at ThreadDump.fun2(ThreadDump.java:45)
        - waiting to lock <0x00000000eb8602f8> (a java.lang.Class for ThreadDump)
        at ThreadDump.access$100(ThreadDump.java:1)
        at ThreadDump$2.run(ThreadDump.java:25)
        at java.lang.Thread.run(Thread.java:745)

"MyThread1" prio=10 tid=0x00007ff1e40af000 nid=0x12ea waiting on condition [0x00007ff1e08f7000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at ThreadDump.fun1(ThreadDump.java:41)
        - locked <0x00000000eb8602f8> (a java.lang.Class for ThreadDump)
        at ThreadDump.access$000(ThreadDump.java:1)
        at ThreadDump$1.run(ThreadDump.java:10)
        at java.lang.Thread.run(Thread.java:745)
```
可以看到，t1 线程的调用栈里有这么一句 ** - locked <0x00000000eb8602f8> (a java.lang.Class for ThreadDump)**，说明它获得了锁，并且进行 sleep(sometime) 操作，因此状态为 TIMED_WAITING。而 t2 线程由于获取不到锁，所以在它的调用栈里能看到 **- waiting to lock <0x00000000eb8602f8> (a java.lang.Class for ThreadDump)**，说明它正在等待锁，因此进入 BLOCKED 状态。

对于 WAITING 状态的线程栈，可以使用以下代码来模拟制造：
```java
    private static final Object lock = new Object();
    public static void main(String[] args) throws InterruptedException {
        synchronized (lock) {
            lock.wait();
        }
    }
```
得到的线程栈为：
```java
"main" prio=10 tid=0x00007f1fdc008800 nid=0x13fe in Object.wait() [0x00007f1fe1fec000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000eb860640> (a java.lang.Object)
        at java.lang.Object.wait(Object.java:503)
        at ThreadDump.main(ThreadDump.java:7)
        - locked <0x00000000eb860640> (a java.lang.Object)
```

# 如何输出线程栈
由于线程栈反映的是 JVM 在某个时间点的线程状态，因此分析线程栈时，为避免偶然性，有必要多输出几份进行分析。以下以 HOT SPOT JVM 为例，首先可以通过以下两种方式得到 JVM 的进程 ID。
1. jps 命令
```bash
[root@localhost ~]# jps
5163 ThreadDump
5173 Jps
```
1. ps -ef | grep java
```bash
[root@localhost ~]# ps -ef | grep java
root       5163   2479  0 01:18 pts/0    00:00:00 java ThreadDump
root       5185   2553  0 01:18 pts/1    00:00:00 grep --color=auto java
```

接下来通过 JDK 自带的 jstack 命令
```bash
[root@localhost ~]# jstack 5163
2017-04-21 01:19:41
Full thread dump Java HotSpot(TM) 64-Bit Server VM (24.79-b02 mixed mode):

"Attach Listener" daemon prio=10 tid=0x00007f72b8001000 nid=0x144c waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Service Thread" daemon prio=10 tid=0x00007f72d4095000 nid=0x1433 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" daemon prio=10 tid=0x00007f72d4092800 nid=0x1432 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" daemon prio=10 tid=0x00007f72d4090000 nid=0x1431 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" daemon prio=10 tid=0x00007f72d408e000 nid=0x1430 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" daemon prio=10 tid=0x00007f72d4065000 nid=0x142f in Object.wait() [0x00007f72d9b83000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000eb804858> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:135)
        - locked <0x00000000eb804858> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:151)
        at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

"Reference Handler" daemon prio=10 tid=0x00007f72d4063000 nid=0x142e in Object.wait() [0x00007f72d9c84000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000eb804470> (a java.lang.ref.Reference$Lock)
        at java.lang.Object.wait(Object.java:503)
        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:133)
        - locked <0x00000000eb804470> (a java.lang.ref.Reference$Lock)

"main" prio=10 tid=0x00007f72d4008800 nid=0x142c in Object.wait() [0x00007f72dc971000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000eb860620> (a java.lang.Object)
        at java.lang.Object.wait(Object.java:503)
        at ThreadDump.main(ThreadDump.java:7)
        - locked <0x00000000eb860620> (a java.lang.Object)

"VM Thread" prio=10 tid=0x00007f72d405e800 nid=0x142d runnable 

"VM Periodic Task Thread" prio=10 tid=0x00007f72d40a0000 nid=0x1434 waiting on condition 

JNI global references: 107
```
即可将线程栈输出到控制台。若输出信息过多，在控制台上不方便分析，则可以将输出信息重定向到文件中，如下所示：
```bash
jstack 5163 > thread.stack
```
若系统中没有 jstack 命令，因为 jstack 命令是 JDK 带的，而有的环境只安装了 JRE 环境。则可以用 kill -3 命令来代替，`kill -3 pid`。Java虚拟机提供了线程转储(Thread dump)的后门， 通过这个后门， 可以将线程堆栈打印出来。 这个后门就是通过向Java进程发送一个QUIT信号， Java虚拟机收到该信号之后， 将系
统当前的JAVA线程调用堆栈打印出来。

若是有运行图形界面的环境，也可以使用一些图形化的工具，例如 JVisualVM 来生成线程栈文件。

# 使用线程栈定位问题
## 发现死锁
当两个或多个线程正在等待被对方占有的锁， 死锁就会发生。 死锁会导致两个线程无法继续运行， 被永远挂起。 
以下代码会产生死锁
```java
/**
 *
 *
 * @author beanlam
 * @version 1.0
 *
 */
public class ThreadDump {
    
    public static void main(String[] args) throws InterruptedException {
        Object lock1 = new Object();
        Object lock2 = new Object();
        
        new Thread1(lock1, lock2).start();
        new Thread2(lock1, lock2).start();
    }

    private static class Thread1 extends Thread {
        Object lock1 = null;
        Object lock2 = null;
        
        public Thread1(Object lock1, Object lock2) {
            this.lock1 = lock1;
            this.lock2 = lock2;
            this.setName(getClass().getSimpleName());
        }
        
        public void run() {
            synchronized (lock1) {
                try {
                    Thread.sleep(2);
                } catch(Exception e) {
                    e.printStackTrace();
                }
                
                synchronized (lock2) {
                    
                }
            }
        }
    }
    
    private static class Thread2 extends Thread {
        Object lock1 = null;
        Object lock2 = null;
        
        public Thread2(Object lock1, Object lock2) {
            this.lock1 = lock1;
            this.lock2 = lock2;
            this.setName(getClass().getSimpleName());
        }
        
        public void run() {
            synchronized (lock2) {
                try {
                    Thread.sleep(2);
                } catch(Exception e) {
                    e.printStackTrace();
                }
                
                synchronized (lock1) {
                    
                }
            }
        }
    }
}
```
对应的线程栈是
```bash
"Thread2" prio=10 tid=0x00007f9bf40a1000 nid=0x1472 waiting for monitor entry [0x00007f9bf8944000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at ThreadDump$Thread2.run(ThreadDump.java:63)
        - waiting to lock <0x00000000eb860498> (a java.lang.Object)
        - locked <0x00000000eb8604a8> (a java.lang.Object)

"Thread1" prio=10 tid=0x00007f9bf409f000 nid=0x1471 waiting for monitor entry [0x00007f9bf8a45000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at ThreadDump$Thread1.run(ThreadDump.java:38)
        - waiting to lock <0x00000000eb8604a8> (a java.lang.Object)
        - locked <0x00000000eb860498> (a java.lang.Object)

Found one Java-level deadlock:
=============================
"Thread2":
  waiting to lock monitor 0x00007f9be4004f88 (object 0x00000000eb860498, a java.lang.Object),
  which is held by "Thread1"
"Thread1":
  waiting to lock monitor 0x00007f9be40062c8 (object 0x00000000eb8604a8, a java.lang.Object),
  which is held by "Thread2"

Java stack information for the threads listed above:
===================================================
"Thread2":
        at ThreadDump$Thread2.run(ThreadDump.java:63)
        - waiting to lock <0x00000000eb860498> (a java.lang.Object)
        - locked <0x00000000eb8604a8> (a java.lang.Object)
"Thread1":
        at ThreadDump$Thread1.run(ThreadDump.java:38)
        - waiting to lock <0x00000000eb8604a8> (a java.lang.Object)
        - locked <0x00000000eb860498> (a java.lang.Object)

Found 1 deadlock.
```
可以看到，当发生了死锁的时候，堆栈中直接打印出了死锁的信息** Found one Java-level deadlock: **，并给出了分析信息。

要避免死锁的问题， 唯一的办法是修改代码。死锁可能会导致整个系统的瘫痪， 具体的严重程度取决于这些线程执行的是什么性质的功能代码， 要想恢复系统， 临时也是唯一的规避办法是将系统重启。

## 定位 CPU 过高的原因
首先需要借助操作系统提供的一些工具，来定位消耗 CPU 过高的 native 线程。不同的操作系统，提供的不同的 CPU 统计命令如下所示：

| 操作系统 | solaris| linux | aix|
|---|---|---|---|
|命令名称|prstat -L <pid>|top -p <pid>|ps -emo THREAD|

以 Linux 为例，首先通过 top -p <pid> 输出该进程的信息，然后输入 H，查看所有的线程的统计情况。
	
```bash
top - 02:04:54 up  2:43,  3 users,  load average: 0.10, 0.05, 0.05
Threads:  13 total,   0 running,  13 sleeping,   0 stopped,   0 zombie
%Cpu(s):  97.74 us,  0.2 sy,  0.0 ni, 2.22 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:   1003456 total,   722012 used,   281444 free,        0 buffers
KiB Swap:  2097148 total,    62872 used,  2034276 free.    68880 cached Mem

PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
3368 zmw2 25 0 256m 9620 6460 R 93.3 0.7 5:42.06 java
3369 zmw2 15 0 256m 9620 6460 S 0.0 0.7 0:00.00 java
3370 zmw2 15 0 256m 9620 6460 S 0.0 0.7 0:00.00 java
3371 zmw2 15 0 256m 9620 6460 S 0.0 0.7 0:00.00 java
3372 zmw2 15 0 256m 9620 6460 S 0.0 0.7 0:00.00 java
3373 zmw2 15 0 256m 9620 6460 S 0.0 0.7 0:00.00 java
3374 zmw2 15 0 256m 9620 6460 S 0.0 0.7 0:00.00 java
3375 zmw2 15 0 256m 9620 6460 S 0.0 0.7 0:00.00 java
```
这个命令输出的 PID 代表的是 native 线程的 id，如上所示，id 为 3368 的 native 线程消耗 CPU 最高。在Java Thread Dump文件中， 每个线程都有tid=...nid=...的属性， 其中nid就是native thread id， 只不过nid中用16进制来表示。 例如上面的例子中3368的十六进制表示为0xd28.在Java线程中查找nid=0xd28即是本地线程对应Java线程。
```bash
"main" prio=1 tid=0x0805c988 nid=0xd28 runnable [0xfff65000..0xfff659c8]
at java.lang.String.indexOf(String.java:1352)
at java.io.PrintStream.write(PrintStream.java:460)
- locked <0xc8bf87d8> (a java.io.PrintStream)
at java.io.PrintStream.print(PrintStream.java:602)
at MyTest.fun2(MyTest.java:16)
- locked <0xc8c1a098> (a java.lang.Object)
at MyTest.fun1(MyTest.java:8)
- locked <0xc8c1a090> (a java.lang.Object)
at MyTest.main(MyTest.java:26)
```
导致 CPU 过高的原因有以下几种原因：
1. Java 代码死循环
2. Java 代码使用了复杂的算法，或者频繁调用
3. JVM 自身的代码导致 CPU 很高

如果在Java线程堆栈中找到了对应的线程ID,并且该Java线程正在执行Native code,说明导致CPU过高的问题代码在JNI调用中，此时需要打印出 Native 线程的线程栈，在 linux 下，使用 pstack <pid> 命令。
如果在 native 线程堆栈中可以找到对应的消耗 CPU 过高的线程 id，可以直接定位为 native 代码的问题。
但是有可能在 native 线程堆栈中找不到对应的消耗 CPU 过高的线程 id，这可能是因为 JNI 调用中重新创建的线程来执行， 那么在 Java 线程堆栈中就不存在该线程的信息，也有可能是虚拟机自身代码导致的 CPU 过高， 如堆内存使用过高导致的频繁 FULL GC ，或者 JVM 的 Bug。

## 定位性能下降原因
性能下降一般是由于资源不足所导致。如果资源不足， 那么有大量的线程在等待资源， 打印的线程堆栈如果发现大量的线程停在同样的调用上下文上， 那么就说明该系统资源是瓶颈。 
导致资源不足的原因可能有：
- 资源数量配置太少（ 如连接池连接配置过少等）， 而系统当前的压力比较大， 资源不足导致了某些线程不能及时获得资源而等待在那里(即挂起)
- 获得资源的线程把持资源时间太久， 导致资源不足，例如以下代码：
```java
void fun1() {
   Connection conn = ConnectionPool.getConnection();//获取一个数据库连接
   //使用该数据库连接访问数据库
   //数据库返回结果，访问完成
   //做其它耗时操作,但这些耗时操作数据库访问无关，
   conn.close(); //释放连接回池
}
```
- 设计不合理导致资源占用时间过久， 如SQL语句设计不恰当， 或者没有索引导致的数据库访问太慢等。
- 资源用完后， 在某种异常情况下， 没有关闭或者回池， 导致可用资源泄漏或者减少， 从而导致资源竞争。

## 定位系统假死原因
导致系统挂死的原因有很多， 其中有一个最常见的原因是线程挂死。每次打印线程堆栈， 该线程必然都在同一个调用上下文上， 因此定位该类型的问题原理是，通过打印多次堆栈， 找出对应业务逻辑使用的线程， 通过对比前后打印的堆栈确认该线程执行的代码段是否一直没有执行完成。 通过打印多次堆栈， 找到挂起的线程（ 即不退出）。
导致线程无法退出的原因可能有：
- 线程正在执行死循环的代码
- 资源不足或者资源泄漏， 造成当前线程阻塞在锁对象上（ 即wait在锁对象上）， 长期得不到唤醒(notify)。
- 如果当前程序和外部通信， 当外部程序挂起无返回时， 也会导致当前线程挂起。


# 性能优化的理念
粗略地划分，代码可分为 cpu consuming 和 io consuming 两种类型，即耗 CPU 的和耗 IO 的代码。如果当前CPU已经能够接近100%的利用率， 并且代码业务逻辑无法再简化， 那么说明该系统的已经达到了性能最大化， 如果再想提高性能， 只能增加处理器(增加更多的机器或者安装更多的CPU)。
而耗 IO 的代码，一般体现为**请求某种资源**，这可以是访问数据库，或者访问网络对端。


评价程序写得好不好，要看随着访问压力的上升，CPU 使用率的变化，好的代码，随着访问压力的上升，CPU 的使用率最终能趋近100%，而坏的代码，使用率始终无法趋近 100%，有可能在 70% 就已经上不去了。好的代码应该在代码本身效率足够高的情况下，通过使用并发等手段，让 CPU 的尽量地忙起来。随着访问压力的上升，CPU 使用率也上升，并且 CPU 所跑的代码都是已经无法再进行逻辑优化或者效率提升的代码，这是最理想的状态。

# 常见性能瓶颈
## 多余的同步
不相关的两个函数， 共用了一个锁,或者不同的共享变量共用了同一个锁， 无谓地制造出了资源争用，如下代码所示：
```java
class MyClass {
  Object sharedObj;
  synchronized void fun1() {...} //访问共享变量sharedObj
  synchronized void fun2() {...} //访问共享变量sharedObj
  synchronized void fun3() {...} //不访问共享变量sharedObj
  synchronized void fun4() {...} //不访问共享变量sharedObj
  synchronized void fun5() {...} //不访问共享变量sharedObj
}
```
上面的代码将sychronized加在类的每一个方法上面， 违背了保护什么锁什么的原则。对于无共享资源的两个方法， 使用了同一个锁， 人为造成了不必要的锁等待。 上述的代码可作如下修改：
```java
class MyClass {
  Object sharedObj;
  synchronized void fun1() {...} //访问共享变量sharedObj
  synchronized void fun2() {...} //访问共享变量sharedObj
  void fun3() {...} //不访问共享变量sharedObj
  void fun4() {...} //不访问共享变量sharedObj
  void fun5() {...} //不访问共享变量sharedObj
}
```

## 锁粒度过大
对共享资源访问完成后， 没有将后续的代码放在synchronized同步代码块之外。 这样会导致当前线程长时间无谓的占有该锁， 其它争用该锁的线程只能等待， 最终导致性能受到极大影响。如下代码所示：
```java
void fun1() {
    synchronized(lock){
    ... ... //正在访问共享资源
    ... ... //做其它耗时操作,但这些耗时操作与共享资源无关
  }
}
```
上面的代码， 会导致一个线程过长地占有锁， 而在这么长的时间里其它线程只能等待。应将上述代码作如下修改，在多 CPU 的环境中，可获得性能的提升：
```java
void fun1() {
  synchronized(lock) {
    ... ... //正在访问共享资源
  }
  ... ... //其它耗时操作代码拿到synchronized代码外面
}
```  
## 字符串连接的滥用
```java
String c = new String("abc") + new String("efg") + new String("12345");
```
每一次+操作都会产生一个临时对象， 并伴随着数据拷贝， 这个对性能是一个极大的消耗。 这个写法常常成为系统的瓶颈， 如果这个地方恰好是一个性能瓶颈， 修改成StringBuffer之后， 性能会有大幅的提升。

## 不恰当的线程模型
在多线程场合下， 如果线程模型不恰当， 也会使性能低下，在网络IO的场合， 使用消息发送队列和消息接收队列来进行异步IO，性能会有显著的提升。

![消息接收](/assets/java-trouble-shooting/1.jpg)
![消息发送](/assets/java-trouble-shooting/2.jpg)

## 不恰当的 GC 参数
不恰当的GC参数设置会导致严重的性能问题，比如堆内存设置过小导致大量的 CPU 时间片被用来做垃圾回收