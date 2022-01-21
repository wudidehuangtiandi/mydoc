# 关于DRUID连接池的一些探究

*在业务中，发现，当前置任务处理时间过长时，由于不使用链接池时则链接可以存货超过300秒没有问题，怀疑是连接池的问题，这里对其原因进行一些分析*

> druid版本：1.2.8

我们进入源码关键类`DruidDataSource`



![avatar](https://picture.zhanghong110.top/docsify/16425727839468.png)

查看其`init`方法,可以看到其中初始化了很多东西，我们直接看其中销毁机制的初始化`createAndStartDestroyThread`

可以看到其中根据`timeBetweenEvictionRunsMillis`参数定义了一个定时器，`timeBetweenEvictionRunsMillis`参数为我们配置的，如果不配置默认为一秒

![avatar](https://picture.zhanghong110.top/docsify/16425730321575.png)

我们进入`DestroyTask`对象看下发现调用了`public void shrink(boolean checkTime, boolean keepAlive)`

代码如下

```
public void shrink(boolean checkTime, boolean keepAlive) {
        try {
            lock.lockInterruptibly();
        } catch (InterruptedException e) {
            return;
        }

        boolean needFill = false;
        int evictCount = 0;
        int keepAliveCount = 0;
        int fatalErrorIncrement = fatalErrorCount - fatalErrorCountLastShrink;
        fatalErrorCountLastShrink = fatalErrorCount;
        
        try {
            if (!inited) {
                return;
            }

            final int checkCount = poolingCount - minIdle;
            final long currentTimeMillis = System.currentTimeMillis();
              
            for (int i = 0; i < poolingCount; ++i) {
                
                DruidConnectionHolder connection = connections[i];
              
                if ((onFatalError || fatalErrorIncrement > 0) && (lastFatalErrorTimeMillis > connection.connectTimeMillis))  {
                    keepAliveConnections[keepAliveCount++] = connection;
                    continue;
                }
          
                if (checkTime) {
                    if (phyTimeoutMillis > 0) {
                        long phyConnectTimeMillis = currentTimeMillis - connection.connectTimeMillis;
                        if (phyConnectTimeMillis > phyTimeoutMillis) {
                            evictConnections[evictCount++] = connection;
                            continue;
                        }
                    }
                
                    long idleMillis = currentTimeMillis - connection.lastActiveTimeMillis;

               
                    if (idleMillis < minEvictableIdleTimeMillis
                            && idleMillis < keepAliveBetweenTimeMillis
                    ) {
                        break;
                    }

                  
                    if (idleMillis >= minEvictableIdleTimeMillis) {
               
                        if (checkTime && i < checkCount) {
                            evictConnections[evictCount++] = connection;
                            continue;
                        } else if (idleMillis > maxEvictableIdleTimeMillis) {
                            evictConnections[evictCount++] = connection;
                            continue;
                        }
                    }

                    if (keepAlive && idleMillis >= keepAliveBetweenTimeMillis) {
                        keepAliveConnections[keepAliveCount++] = connection;
                    }
                } else {
                    if (i < checkCount) {
                        evictConnections[evictCount++] = connection;
                    } else {
                        break;
                    }
                }
            }

            int removeCount = evictCount + keepAliveCount;
            if (removeCount > 0) {
                System.arraycopy(connections, removeCount, connections, 0, poolingCount - removeCount);
                Arrays.fill(connections, poolingCount - removeCount, poolingCount, null);
                poolingCount -= removeCount;
            }
            keepAliveCheckCount += keepAliveCount;

            if (keepAlive && poolingCount + activeCount < minIdle) {
                needFill = true;
            }
        } finally {
            lock.unlock();
        }

        if (evictCount > 0) {
            for (int i = 0; i < evictCount; ++i) {
                DruidConnectionHolder item = evictConnections[i];
                Connection connection = item.getConnection();
                JdbcUtils.close(connection);
                destroyCountUpdater.incrementAndGet(this);
            }
            Arrays.fill(evictConnections, null);
        }

        if (keepAliveCount > 0) {
            // keep order
            for (int i = keepAliveCount - 1; i >= 0; --i) {
                DruidConnectionHolder holer = keepAliveConnections[i];
                Connection connection = holer.getConnection();
                holer.incrementKeepAliveCheckCount();

                boolean validate = false;
                try {
                    this.validateConnection(connection);
                    validate = true;
                } catch (Throwable error) {
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("keepAliveErr", error);
                    }
                    // skip
                }

                boolean discard = !validate;
                if (validate) {
                    holer.lastKeepTimeMillis = System.currentTimeMillis();
                    boolean putOk = put(holer, 0L, true);
                    if (!putOk) {
                        discard = true;
                    }
                }

                if (discard) {
                    try {
                        connection.close();
                    } catch (Exception e) {
                        // skip
                    }

                    lock.lock();
                    try {
                        discardCount++;

                        if (activeCount + poolingCount <= minIdle) {
                            emptySignal();
                        }
                    } finally {
                        lock.unlock();
                    }
                }
            }
            this.getDataSourceStat().addKeepAliveCheckCount(keepAliveCount);
            Arrays.fill(keepAliveConnections, null);
        }

        if (needFill) {
            lock.lock();
            try {
                int fillCount = minIdle - (activeCount + poolingCount + createTaskCount);
                for (int i = 0; i < fillCount; ++i) {
                    emptySignal();
                }
            } finally {
                lock.unlock();
            }
        } else if (onFatalError || fatalErrorIncrement > 0) {
            lock.lock();
            try {
                emptySignal();
            } finally {
                lock.unlock();
            }
        }
    }

```

我们看下移除链接的代码`evictConnections[evictCount++] = connection;`可以看到一共有四出，我们分别分析可以得到如下结论

1.需要`phyTimeoutMillis > 0`时才可能触发,默认是-1。可见不是这里导致。

2.`idleMillis >= minEvictableIdleTimeMillis`为先决条件，`idleMillis `是系统时间减去上次活跃时间，`minEvictableIdleTimeMillis`默认为30分钟

 第一：`checkTime && i < checkCount`此时`checkTime` 默认为`true` 。` checkCount`为 `poolingCount` - `minIdle`。`poolingCount` 为当前链接数后者为设置的值。

第二：`idleMillis > maxEvictableIdleTimeMillis`，`maxEvictableIdleTimeMillis`不设置默认大小为7小时

3.最后一处为`checkTime`为`false`时才能触发，这里不会进入。

我们设置参数每分钟轮询一次

```
initial-size: 5
min-idle: 5
timeBetweenEvictionRunsMillis: 60000 （一分钟）
```

我们新建个服务看看，打个断点看下到底是哪边导致的链接被移除。

事实证明德鲁伊还没动手（断点还未放开，而且参数配置页不支持300秒就杀死链接），这个链接就嗝屁了，这里不得不再度怀疑`mysql`

我们使用`show status`发现Aborted_clients	到达300秒后就会增加德鲁伊连接池指定的连接数,可见是`mysql`的问题

奇怪的点在于通过`show PROCESSLIST`命令确实可以发现许多超过300秒依旧存活的链接，并且呢`show variables like '%timeout%'`可以查到

`interactive_timeout`,`mysqlx_interactive_timeout`,`mysqlx_wait_timeout`, `wait_timeout`都是28800秒

我们看下Aborted_clients状态变量值增加的几种情况

1.客户端退出之前，没有调用`mysql_close()`

2.客户端“沉睡”，超过`wait_timeout `或者` interactive_timeout`设置时间(单位：秒)没有向服务器发起任何请求。

3.客户端程序在数据传输的期间突然终止。

4.Linux使用的Ethernet协议，包含半双工和全双工。有些Linux Ethernet驱动存在这个bug。

!>初步排查下来都不满足以上的情况

我们继续将druid参数修改成如下

```
timeBetweenEvictionRunsMillis: 15000
minEvictableIdleTimeMillis: 30000
maxEvictableIdleTimeMillis: 60000
```

可见15秒轮询一次，最大存活时间60秒；

查询数据库可知60秒后，5个新建的链接都被杀死，但是`mysql`新建的`Aborted_clients`并没有增加。可见应当是数据库杀死了链接。

> 但是可以发现，当超时请求进来后由于`checkCount`已经是-1了，应为之前的5个链接已经被杀死，所以30秒后未处理，需要60秒后处理,
>
> 这里可以得出一个结论，如果说数据库或者别的什么没有杀死链接，当只有5个链接时将会一直维持到数据库杀死它为止，或者到达maxEvictableIdleTimeMillis的时间为止，一般为了防止被数据库杀死后使用出现异常所以默认为7小时，数据库为八小时。

再次执行那个超长时间的链接。我们发现了一个现象，此时数据库有两条链接，但是这个回收轮询只能轮循到一条，我们处理的线程却未在`poolingCount`中,这个也解释了为什么我数值设置的如此小缺并没有成功回收掉链接(后尝试5条链接没被销毁时去占用，发现占用也会一直持续，直到300秒被杀死，不会在60秒终止，可见druid应该认为该链接不在闲置范围内。)，直到300秒后这个链接被不明原因杀死（暂且猜测还是`mysql`）问题。

这边通过debug发现无论时druid没有链接了（空闲时间过长全都销毁了）去新建，还是说还有线程可用，只要被执行中的任务占用则就不会被回收（任务一定会保持到300秒后被杀死）。可见和这个锅和德鲁伊确实没关系。所以无论这个回收时间设为多少都无法避免这个问题，就可以理解了。

> 我们兜兜转转最后所有的证据都指向mysql数据库，但是通过上面的1，2，3，4发现这些条件都不满足，联想到我的服务器部署中间还经过了跳板机及FRP服务，我们尝试直连数据库服务器试试

居然成了，最后不是数据库的问题也不是德鲁伊的问题而是跳板机或者FRP服务的问题

我们再来探究下到底是跳板机还是FRP的问题。

经过初步排查FRP应该没有该项配置。问题最终甩给了腾讯云的跳板机，猜测是否云网关导致的。明天找腾讯云问下。