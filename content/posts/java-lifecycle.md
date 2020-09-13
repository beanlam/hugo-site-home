---
title: "Java 实现生命周期管理机制"
date: 2016-01-13T18:22:03+08:00
categories : ["java"]
---


# 前言

最近一直在研究某个国产开源的MySQL数据库中间件，拉下其最新版的代码到eclipse后，启动起来，然后做各种测试和代码追踪；用完想要关闭它时，拉出它的STOP类想要运行时，发现这个类里赫然只写以下几行代码，于是我感觉瞬间受到了很多伤害。


```java
    public static void main(String[] args) {
        System.out.println(new Date() + ",server shutdown!");
    }
```
这个中间件启动和运行的时候，开启了监听，启动着许多线程在跑着，并且有许多socket连接。但是并没有找到一个优雅的方式将其关闭。于是无奈之下，我只能去点eclipse的心碎小红点，强行停掉VM。

如果是一个架构良好，模块化清晰的软件，特别是Server类的软件，拥有一套生命周期管理机制是非常重要的。不仅可以管理各个模块的生命周期，也可以在启停整个软件的时候更优雅，不会漏掉任何资源。



# 生命周期机制简易实现

## 生命周期状态
一个模块的生命周期状态一般有以下几个：

> 新生 -> 初始化中 -> 初始化完成 -> 启动中 -> 启动完成 -> 正在暂停 -> 已经暂停 -> 正在恢复 -> 已经恢复 -> 正在销毁 -> 已经销毁

其中，任何一个状态之间的转化如果失败，那么就会进入另外一种状态：失败。

为此，可以用一个枚举类来枚举出这几个状态，如下所示：
```java
public enum LifecycleState {

    NEW, //新生

    INITIALIZING, INITIALIZED, //初始化

    STARTING, STARTED, //启动

    SUSPENDING, SUSPENDED, //暂停

    RESUMING, RESUMED,//恢复

    DESTROYING, DESTROYED,//销毁

    FAILED;//失败

}
```

## 接口

生命周期中的各种行为规范，也需要一个接口来定义，如下所示:
```java
public interface ILifecycle {

    /**
     * 初始化
     * 
     * @throws LifecycleException
     */
    public void init() throws LifecycleException;

    /**
     * 启动
     * 
     * @throws LifecycleException
     */
    public void start() throws LifecycleException;

    /**
     * 暂停
     * 
     * @throws LifecycleException
     */
    public void suspend() throws LifecycleException;

    /**
     * 恢复
     * 
     * @throws LifecycleException
     */
    public void resume() throws LifecycleException;

    /**
     * 销毁
     * 
     * @throws LifecycleException
     */
    public void destroy() throws LifecycleException;

    /**
     * 添加生命周期监听器
     * 
     * @param listener
     */
    public void addLifecycleListener(ILifecycleListener listener);

    /**
     * 删除生命周期监听器
     * 
     * @param listener
     */
    public void removeLifecycleListener(ILifecycleListener listener);

}
```

发生生命周期状态转化时，可能需要触发对某类事件感兴趣的监听者，因此`ILifeCycle`也定义了两个方法可以添加和移除监听者。分别是：`public void addLifecycleListener(ILifecycleListener listener);`和`public void removeLifecycleListener(ILifecycleListener listener);`

监听者也由一个接口来定义其行为规范，如下所示:
```java
public interface ILifecycleListener {

    /**
     * 对生命周期事件进行处理
     * 
     * @param event 生命周期事件
     */
    public void lifecycleEvent(LifecycleEvent event);
}
```

生命周期事件由`LifecycleEvent`来表示，如下所示:
```java
public final class LifecycleEvent {

    private LifecycleState state;

    public LifecycleEvent(LifecycleState state) {
        this.state = state;
    }

    /**
     * @return the state
     */
    public LifecycleState getState() {
        return state;
    }

}
```

## 骨架实现
有了ILifeCycle接口以后，任何实现了这个接口的类将会被作为一个生命周期管理对象，这个类可以是一个socket监听服务，也可以代表一个特定的模块，等等。那我们是不是只要实现ILifeCycle就可以了? 可以这么说，但考虑到各个生命周期管理对象在生命周期的各个阶段会有一些共同的行为，比如说：
- 设置自身的生命周期状态
- 检查状态的转换是否符合逻辑
- 通知监听者生命周期状态发生了变化

因此，提供一个抽象类`AbstractLifeCycle`，作为`ILifeCycle`的**骨架实现**是有重要意义的，这样避免了很多的重复代码，使得架构更加清晰。这个抽象类会实现`ILifeCycle`中定义的所有接口方法，并添加对应的抽象方法，供子类实现。`AbstractLifeCycle`可以这么实现：
```java
public abstract class AbstractLifecycle implements ILifecycle {

    private List<ILifecycleListener> listeners = new CopyOnWriteArrayList<ILifecycleListener>();

    /**
     * state 代表当前生命周期状态
     */
    private LifecycleState state = LifecycleState.NEW;

    /*
     * @see ILifecycle#init()
     */
    @Override
    public final synchronized void init() throws LifecycleException {
        if (state != LifecycleState.NEW) {
            return;
        }

        setStateAndFireEvent(LifecycleState.INITIALIZING);
        try {
            init0();
        } catch (Throwable t) {
            setStateAndFireEvent(LifecycleState.FAILED);
            if (t instanceof LifecycleException) {
                throw (LifecycleException) t;
            } else {
                throw new LifecycleException(formatString(
                        "Failed to initialize {0}, Error Msg: {1}", toString(), t.getMessage()), t);
            }
        }
        setStateAndFireEvent(LifecycleState.INITIALIZED);
    }

    protected abstract void init0() throws LifecycleException;

    /*
     * @see ILifecycle#start()
     */
    @Override
    public final synchronized void start() throws LifecycleException {
        if (state == LifecycleState.NEW) {
            init();
        }

        if (state != LifecycleState.INITIALIZED) {
            return;
        }

        setStateAndFireEvent(LifecycleState.STARTING);
        try {
            start0();
        } catch (Throwable t) {
            setStateAndFireEvent(LifecycleState.FAILED);
            if (t instanceof LifecycleException) {
                throw (LifecycleException) t;
            } else {
                throw new LifecycleException(formatString("Failed to start {0}, Error Msg: {1}",
                        toString(), t.getMessage()), t);
            }
        }
        setStateAndFireEvent(LifecycleState.STARTED);
    }

    protected abstract void start0() throws LifecycleException;

    /*
     * @see ILifecycle#suspend()
     */
    @Override
    public final synchronized void suspend() throws LifecycleException {
        if (state == LifecycleState.SUSPENDING || state == LifecycleState.SUSPENDED) {
            return;
        }

        if (state != LifecycleState.STARTED) {
            return;
        }

        setStateAndFireEvent(LifecycleState.SUSPENDING);
        try {
            suspend0();
        } catch (Throwable t) {
            setStateAndFireEvent(LifecycleState.FAILED);
            if (t instanceof LifecycleException) {
                throw (LifecycleException) t;
            } else {
                throw new LifecycleException(formatString("Failed to suspend {0}, Error Msg: {1}",
                        toString(), t.getMessage()), t);
            }
        }
        setStateAndFireEvent(LifecycleState.SUSPENDED);
    }

    protected abstract void suspend0() throws LifecycleException;

    /*
     * @see ILifecycle#resume()
     */
    @Override
    public final synchronized void resume() throws LifecycleException {
        if (state != LifecycleState.SUSPENDED) {
            return;
        }

        setStateAndFireEvent(LifecycleState.RESUMING);
        try {
            resume0();
        } catch (Throwable t) {
            setStateAndFireEvent(LifecycleState.FAILED);
            if (t instanceof LifecycleException) {
                throw (LifecycleException) t;
            } else {
                throw new LifecycleException(formatString("Failed to resume {0}, Error Msg: {1}",
                        toString(), t.getMessage()), t);
            }
        }
        setStateAndFireEvent(LifecycleState.RESUMED);
    }

    protected abstract void resume0() throws LifecycleException;

    /*
     * @see ILifecycle#destroy()
     */
    @Override
    public final synchronized void destroy() throws LifecycleException {
        if (state == LifecycleState.DESTROYING || state == LifecycleState.DESTROYED) {
            return;
        }

        setStateAndFireEvent(LifecycleState.DESTROYING);
        try {
            destroy0();
        } catch (Throwable t) {
            setStateAndFireEvent(LifecycleState.FAILED);
            if (t instanceof LifecycleException) {
                throw (LifecycleException) t;
            } else {
                throw new LifecycleException(formatString("Failed to destroy {0}, Error Msg: {1}",
                        toString(), t.getMessage()), t);
            }
        }
        setStateAndFireEvent(LifecycleState.DESTROYED);
    }

    protected abstract void destroy0() throws LifecycleException;

    /*
     * @see
     * ILifecycle#addLifecycleListener(ILifecycleListener)
     */
    @Override
    public void addLifecycleListener(ILifecycleListener listener) {
        listeners.add(listener);
    }

    /*
     * @see
     * ILifecycle#removeLifecycleListener(ILifecycleListener)
     */
    @Override
    public void removeLifecycleListener(ILifecycleListener listener) {
        listeners.remove(listener);
    }

    private void fireLifecycleEvent(LifecycleEvent event) {
        for (Iterator<ILifecycleListener> it = listeners.iterator(); it.hasNext();) {
            ILifecycleListener listener = it.next();
            listener.lifecycleEvent(event);
        }
    }

    protected synchronized LifecycleState getState() {
        return state;
    }

    private synchronized void setStateAndFireEvent(LifecycleState newState) throws LifecycleException {
        state = newState;
        fireLifecycleEvent(new LifecycleEvent(state));
    }

    private String formatString(String pattern, Object... arguments) {
        return MessageFormat.format(pattern, arguments);
    }

    /*
     * @see java.lang.Object#toString()
     */
    @Override
    public String toString() {
        return getClass().getSimpleName();
    }
}

```
可以看到，抽象类的骨架实现中做了几件生命周期管理中通用的事情，检查状态之间的转换是否合法(比如说start之前必须要init)，设置内部状态，以及触发相应的监听者。

抽象类实现了`ILifeCycle`定义的方法后，又留出了相应的抽象方法供其子类实现，如上面的代码所示，其留出来的抽象方法有以下这些:
```java
protected abstract void init0() throws LifecycleException;
protected abstract void start0() throws LifecycleException;
protected abstract void suspend0() throws LifecycleException;
protected abstract void resume0() throws LifecycleException;
protected abstract void destroy0() throws LifecycleException;
```

## 优雅的实现
到目前为止，我们已经定义了接口`ILifeCycle`，以及其骨架实现`AbstractLifeCycle`，并且增加了监听者机制。貌似我们可以开始写一个类来继承`AbstractLifecycle`，并重写其定义的抽象方法了，so far so good。

但在开始之前，我们还需要考虑另外几个问题，
- 我们的实现类是否对所有的抽象方法都感兴趣？
- 是否每个实现累都需要实现`init0`, `start0`, `suspend0`, `resume0`, `destroy0`?
- 是否有时候，我们的那些有生命的类或者模块并不支持暂停(suspend),恢复(resume)?

直接继承`AbstractLifeCycle`，就意味着必须实现其全部的抽象方法。
因此，我们还需要一个默认实现，`DefaultLifeCycle`，让它继承`AbstractLifeCycle`，并实现所有抽象方法，但它并不做任何实际的事情, do nothing。只是让我们真正的实现类来继承这个默认的实现类，并重写感兴趣的方法。

于是，我们的`DefaultLifeCycle`就这么诞生了:
```java 
public class DefaultLifecycle extends AbstractLifecycle {

    /*
     * @see AbstractLifecycle#init0()
     */
    @Override
    protected void init0() throws LifecycleException {
        // do nothing
    }

    /*
     * @see AbstractLifecycle#start0()
     */
    @Override
    protected void start0() throws LifecycleException {
        // do nothing
    }

    /*
     * @see AbstractLifecycle#suspend0()
     */
    @Override
    protected void suspend0() throws LifecycleException {
        // do nothing
    }

    /*
     * @see AbstractLifecycle#resume0()
     */
    @Override
    protected void resume0() throws LifecycleException {
        // do nothing
    }

    /*
     * @see AbstractLifecycle#destroy0()
     */
    @Override
    protected void destroy0() throws LifecycleException {
        // do nothing
    }

}
```

对于`DefaultLifeCycle`来说，do nothing就是其职责。
因此接下来我们可以写一个自己的实现类，继承`DefaultLifeCycle`，并重写那些感兴趣的生命周期方法。

例如，我有一个类只需要在初始化，启动，和销毁时做一些任务，那么可以这么写:

```java 

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

public class SocketServer extends DefaultLifecycle {

    private ServerSocket acceptor = null;
    private int port = 9527;
    /* 
     * @see DefaultLifecycle#init0()
     */
    @Override
    protected void init0() throws LifecycleException {
        try {
            acceptor = new ServerSocket(port);
        } catch (IOException e) {
            throw new LifecycleException(e);
        }
    }

    /* 
     * @see DefaultLifecycle#start0()
     */
    @Override
    protected void start0() throws LifecycleException {
        Socket socket = null;
        try {
            socket = acceptor.accept();
            //do something with socket
            
            
        } catch (IOException e) {
            throw new LifecycleException(e);
        } finally {
            if (socket != null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        }
    }

    /* 
     * @see DefaultLifecycle#destroy0()
     */
    @Override
    protected void destroy0() throws LifecycleException {
        if (acceptor != null) {
            try {
                acceptor.close();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }
}

```

这里的ServerSocket中，init0初始化socket监听，start0开始获取socket连接, destroy0销毁socket监听。
在这套生命周期管理机制下，我们将会很容易地对资源进行管理，不会发生资源未关闭的情况，架构和模块化更加清晰。