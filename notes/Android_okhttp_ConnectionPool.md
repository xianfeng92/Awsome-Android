
客户端和服务器建立 socket 连接需要经历 TCP 的三次握手和四次挥手，是一种比较消耗资源的动作。Http中有一种 keepAlive connections 的机制，
在和客户端通信结束以后可以保持连接指定的时间。OkHttp3 支持5个并发 socket 连接，默认的 keepAlive 时间为5分钟。

下面我们来看看OkHttp3是怎么实现连接池复用的。

## OkHttp3的连接池--ConnectionPool

```
public final class ConnectionPool {
  // 线程池
  final RealConnectionPool delegate;

  public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.delegate = new RealConnectionPool(maxIdleConnections, keepAliveDuration, timeUnit);
  }

  // 返回池中的空闲连接数
  public int idleConnectionCount() {
    return delegate.idleConnectionCount();
  }

  // 返回池中的连接总数
  public int connectionCount() {
    return delegate.connectionCount();
  }

  // 关闭并删除池中的所有空闲连接
  public void evictAll() {
    delegate.evictAll();
  }
}
```

上面的代码可以看出，在构造 ConnectionPool 时，其实也就是将参数接收过来构造一个 RealConnectionPool 。


```
public final class RealConnectionPool {
  // executor线程池，类似于CachedThreadPool，用于执行清理空闲连接的任务。
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<>(), Util.threadFactory("OkHttp ConnectionPool", true));

  // 最大空闲连接数
  private final int maxIdleConnections;
  // connection 的 keepAlive 时间
  private final long keepAliveDurationNs;
  ...
  // Deque 双向队列，用于存放所有的 connection 
  private final Deque<RealConnection> connections = new ArrayDeque<>();
  // RouteDatabase，用来记录连接失败的路线名单
  final RouteDatabase routeDatabase = new RouteDatabase();
  boolean cleanupRunning;

  public RealConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
  }
...
}
```

## connection 的创建

这里的 connection 就是 socket（The application layer socket），有了 socket ，客户端就可以和服务端进行数据传输了。

connection 的创建在 ConnectInterceptor 中调用 transmitter.newExchange 来完成的，看看其主要做了啥：

```
  /** Returns a new exchange to carry a new request and response. */
  Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    synchronized (connectionPool) {
      if (noMoreExchanges) throw new IllegalStateException("released");
      if (exchange != null) throw new IllegalStateException("exchange != null");
    }

    ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);
    Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);

    synchronized (connectionPool) {
      this.exchange = result;
      this.exchangeRequestDone = false;
      this.exchangeResponseDone = false;
      return result;
    }
  }
```

该方法的重点在：

```
ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);

```

其中ExchangeCodec主要负责对HTTP请求进行编码以及对HTTP响应进行解码。下面看看ExchangeCodec#find

```
  public ExchangeCodec find(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      return resultConnection.newCodec(client, chain);
    } catch (RouteException e) {
      trackFailure();
      throw e;
    } catch (IOException e) {
      trackFailure();
      throw new RouteException(e);
    }
  }
```
该方法很简单，通过 findHealthyConnection 找到一个 connection。

```
  /**
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        candidate.noNewExchanges();
        continue;
      }

      return candidate;
    }
  }
```

通过上述代码可知，findHealthyConnection 里面有一个循环，该循环退出的唯一条件就是找到一个可用的 connection。



## 缓存操作：添加、获取、回收连接

### 从缓存中获取连接

当 ConnectInterceptor 要和服务器建立连接时，会调用此方法来 check 当前 connectionPool 中是否存在
符合条件的 connection，如果存在则复用它。

```
  boolean transmitterAcquirePooledConnection(Address address, Transmitter transmitter,
      @Nullable List<Route> routes, boolean requireMultiplexed) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (requireMultiplexed && !connection.isMultiplexed()) continue;
      if (!connection.isEligible(address, routes)) continue;
      transmitter.acquireConnectionNoEvents(connection);
      return true;
    }
    return false;
  }
```

获取连接的逻辑比较简单，就遍历连接池里所有的 connection，然后用 RealConnection 的 isEligible 方法找到符合条件的连接，即判断此 connection 能否将流分配带到 
指定的address。如果有符合条件的 connection 则复用。同时还调用了 transmitter 的 acquireConnectionNoEvents 方法, acquireConnectionNoEvents 方法的作用是对
满足条件的 connection 引用的 transmitter 进行计数，如果一个 connection 的 transmitter 的引用计数为0，则 okhttp 会自动回收该 connection。

```
    void acquireConnectionNoEvents(RealConnection connection) {
    assert (Thread.holdsLock(connectionPool));

    if (this.connection != null) throw new IllegalStateException();
    this.connection = connection;
    connection.transmitters.add(new TransmitterReference(this, callStackTrace));
  }

  /** Current calls carried by this connection. */
  final List<Reference<Transmitter>> transmitters = new ArrayList<>();

```

每一个 connection 中都有一个 transmitters 变量，用于记录对 transmitter 的引用。transmitter 中包装有 HttpCodec，而HttpCodec里面封装有Request和
Response读写 Socket 的抽象。每一个请求 Request 通过 Http 来请求数据时都需要通过 transmitter 来获取 HttpCodec，从而读取响应结果，而每一个 transmitter
都是和一个 connection 绑定的，因为只有通过 connection 才能建立socket连接。所以 transmitter 可以说是 connection、HttpCodec和请求之间的桥梁。

当然同样的 transmitter 还有一个 releaseConnectionNoEvents 方法，用于移除计数，也就是将当前的 transmitter 的引用从对应的 connection 的引用列表中移除。

### 向缓存中添加连接

```
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```

添加连接之前会先调用线程池执行清理空闲连接的任务，也就是回收空闲的连接。


### 空闲连接的回收

```
  private final Runnable cleanupRunnable = () -> {
    while (true) {
      long waitNanos = cleanup(System.nanoTime());
      if (waitNanos == -1) return;
      if (waitNanos > 0) {
        long waitMillis = waitNanos / 1000000L;
        waitNanos -= (waitMillis * 1000000L);
        synchronized (RealConnectionPool.this) {
          try {
            RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
          } catch (InterruptedException ignored) {
          }
        }
      }
    }
  };
```
cleanupRunnable 中执行清理任务是通过 cleanup 方法来完成，cleanup方法会返回下次需要清理的间隔时间，然后会调用 wait 方法释放锁和时间片。
等时间到了就再次进行清理。下面看看具体的清理逻辑：

```
 long cleanup(long now) {
    //记录活跃的连接数
    int inUseConnectionCount = 0;
    //记录空闲的连接数
    int idleConnectionCount = 0;
    //空闲时间最长的连接
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        //判断连接是否在使用，也就是通过 Transmitter 的引用计数来判断
        //返回值大于0说明正在被使用
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          //活跃的连接数+1
          inUseConnectionCount++;
          continue;
        }
        //说明是空闲连接，所以空闲连接数+1
        idleConnectionCount++;

        // 找出了空闲时间最长的连接，准备移除
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
         //如果空闲时间最长的连接的空闲时间超过了5分钟
        //或是空闲的连接数超过了限制，就移除
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
         //如果存在空闲连接但是还没有超过5分钟
        //就返回剩下的时间，便于下次进行清理
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        //如果没有空闲的连接，那就等5分钟后再尝试清理
        return keepAliveDurationNs;
      } else {
        //当前没有任何连接，就返回-1，跳出循环
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }
```

下面我们看看判断连接是否是活跃连接的 pruneAndGetAllocationCount 方法：

```
  private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<Transmitter>> references = connection.transmitters;
    for (int i = 0; i < references.size(); ) {
      Reference<Transmitter> reference = references.get(i);
      // 如果存在引用，就说明是活跃连接，就继续看下一个 Transmitter
      if (reference.get() != null) {
        i++;
        continue;
      }

      // 发现泄漏的引用，会打印日志
      TransmitterReference transmitterRef = (TransmitterReference) reference;
      String message = "A connection to " + connection.route().address().url()
          + " was leaked. Did you forget to close a response body?";
      Platform.get().logCloseableLeak(message, transmitterRef.callStackTrace);
      // 如果没有引用，就移除
      references.remove(i);
      connection.noNewExchanges = true;

       // 如果列表为空，就说明此连接上没有StreamAllocation引用了，就返回0，表示是空闲的连接
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }
    // 遍历结束后，返回引用的数量，说明当前连接是活跃连接
    return references.size();
  }
```

## 总结

1. OkHttp3中支持5个并发socket连接，默认的keepAlive时间为5分钟，当然我们可以在构建OkHttpClient时设置不同的值。

2. OkHttp3通过Deque来存储和管理连接

3. OkHttp3通过每个连接的引用计数对象 Transmitter 的计数来回收空闲的连接，向连接池添加新的连接时会触发执行清理空闲连接的任务。
   清理空闲连接的任务通过线程池来执行。




[OKHttp源码](https://www.jianshu.com/p/412155af841f)
[OkHttp网络请求分析](https://www.jianshu.com/p/8f34c928cf13)
[android okhttp3 网络请求框架解析](https://www.jianshu.com/p/816223c8960c)
[OkHttp3 架构分析](https://blog.dreamtobe.cn/okhttp3-framework/)

