# RealConnection


```
public final class RealConnection extends Http2Connection.Listener implements Connection {
  private static final String NPE_THROW_WITH_NULL = "throw with null exception";
  private static final int MAX_TUNNEL_ATTEMPTS = 21;

  public final RealConnectionPool connectionPool;
  private final Route route;

  // The fields below are initialized by connect() and never reassigned.

  /** The low-level TCP socket. */
  private Socket rawSocket;// 底层socket

  /**
   * The application layer socket. Either an {@link SSLSocket} layered over {@link #rawSocket}, or
   * {@link #rawSocket} itself if this connection does not use SSL.
   */
  private Socket socket; // 应用层socket
  private Handshake handshake; // 握手
  private Protocol protocol;
  private Http2Connection http2Connection;
  // source和sink，为与服务器交互的输入输出流
  private BufferedSource source;
  private BufferedSink sink;

  // The fields below track connection state and are guarded by connectionPool.
  // 下面这个字段是 属于表示connection状态的字段，并且有connectPool统一管理

  /**
   * If true, no new exchanges can be created on this connection. Once true this is always true.
   * Guarded by {@link #connectionPool}.
   */
  // 如果 noNewExchanges 被设为true，则 noNewExchanges 一直为true，不会被改变，并且表示这个 connection 不会再创建新的 exchange
  boolean noNewExchanges;

  /**
   * The number of times there was a problem establishing a stream that could be due to route
   * chosen. Guarded by {@link #connectionPool}.
   */
  int routeFailureCount;
  //成功的次数
  int successCount;
  private int refusedStreamCount;

  /**
   * The maximum number of concurrent streams that can be carried by this connection. If {@code
   * allocations.size() < allocationLimit} then new streams can be created on this connection.
   */
   // 该 connection 可以承载最大并发流的限制，如果不超过限制，可以随意增加
  private int allocationLimit = 1;

  /** Current calls carried by this connection. */
  final List<Reference<Transmitter>> transmitters = new ArrayList<>();

  /** Nanotime timestamp when {@code allocations.size()} reached zero. */
  long idleAtNanos = Long.MAX_VALUE;

  public RealConnection(RealConnectionPool connectionPool, Route route) {
    this.connectionPool = connectionPool;
    this.route = route;
  }

  /** Prevent further exchanges from being created on this connection. */
  public void noNewExchanges() {
    assert (!Thread.holdsLock(connectionPool));
    synchronized (connectionPool) {
      noNewExchanges = true;
    }
  }

  static RealConnection testConnection(
      RealConnectionPool connectionPool, Route route, Socket socket, long idleAtNanos) {
    RealConnection result = new RealConnection(connectionPool, route);
    result.socket = socket;
    result.idleAtNanos = idleAtNanos;
    return result;
  }

  public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled, Call call,
      EventListener eventListener) {
    if (protocol != null) throw new IllegalStateException("already connected");
    // 线路的选择
    RouteException routeException = null;
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);

    if (route.address().sslSocketFactory() == null) {
      if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication not enabled for client"));
      }
      String host = route.address().url().host();
      if (!Platform.get().isCleartextTrafficPermitted(host)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication to " + host + " not permitted by network security policy"));
      }
    } else {
      if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
        throw new RouteException(new UnknownServiceException(
            "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"));
      }
    }
   // 连接开始
    while (true) {
      try {
        // 根据请求判断是否需要建立隧道连接，如果建立隧道连接则调用connectTunnel
        if (route.requiresTunnel()) {
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break;
          }
        } else {
           // 一般都走这条逻辑了，实际上很简单就是socket的连接
          connectSocket(connectTimeout, readTimeout, call, eventListener);
        }
         // https的建立
        establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
        eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
        break;
      } catch (IOException e) {
        closeQuietly(socket);
        closeQuietly(rawSocket);
        socket = null;
        rawSocket = null;
        source = null;
        sink = null;
        handshake = null;
        protocol = null;
        http2Connection = null;

        eventListener.connectFailed(call, route.socketAddress(), route.proxy(), null, e);

        if (routeException == null) {
          routeException = new RouteException(e);
        } else {
          routeException.addConnectException(e);
        }

        if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
          throw routeException;
        }
      }
    }

    if (route.requiresTunnel() && rawSocket == null) {
      ProtocolException exception = new ProtocolException("Too many tunnel connections attempted: "
          + MAX_TUNNEL_ATTEMPTS);
      throw new RouteException(exception);
    }

    if (http2Connection != null) {
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();
      }
    }
  }

```


```
private void connectSocket(int connectTimeout, int readTimeout, Call call,
      EventListener eventListener) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();
    // 根据代理类型来选择socket类型，是代理还是直连
    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);

    eventListener.connectStart(call, route.socketAddress(), proxy);
    rawSocket.setSoTimeout(readTimeout);
    try {
       // 连接socket，之所以这样写是因为支持不同的平台
      //里面实际上是  socket.connect(address, connectTimeout);
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
      ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
      ce.initCause(e);
      throw ce;
    }
    try {
      // 得到输入／输出流
      source = Okio.buffer(Okio.source(rawSocket));
      sink = Okio.buffer(Okio.sink(rawSocket));
    } catch (NullPointerException npe) {
      if (NPE_THROW_WITH_NULL.equals(npe.getMessage())) {
        throw new IOException(npe);
      }
    }
  }
```

1. 创建Socket，非SOCKS代理的情况下，通过SocketFactory创建；在SOCKS代理则传入proxy手动new一个出来。
2. 为Socket设置超时
3. 完成特定于平台的连接建立
4. 创建用于I/O的source和sink





```
  @Override public void connectSocket(Socket socket, InetSocketAddress address,
      int connectTimeout) throws IOException {
    try {
      socket.connect(address, connectTimeout);
    } catch (AssertionError e) {
      if (Util.isAndroidGetsocknameError(e)) throw new IOException(e);
      throw e;
    } catch (ClassCastException e) {
      // On android 8.0, socket.connect throws a ClassCastException due to a bug
      // see https://issuetracker.google.com/issues/63649622
      if (Build.VERSION.SDK_INT == 26) {
        throw new IOException("Exception in connect", e);
      } else {
        throw e;
      }
    }
  }
```


```
 public void connect(SocketAddress endpoint, int timeout) throws IOException {
        if (endpoint == null)
            throw new IllegalArgumentException("connect: The address can't be null");

        if (timeout < 0)
          throw new IllegalArgumentException("connect: timeout can't be negative");

        if (isClosed())
            throw new SocketException("Socket is closed");

        if (!oldImpl && isConnected())
            throw new SocketException("already connected");

        if (!(endpoint instanceof InetSocketAddress))
            throw new IllegalArgumentException("Unsupported address type");

        InetSocketAddress epoint = (InetSocketAddress) endpoint;
        InetAddress addr = epoint.getAddress ();
        int port = epoint.getPort();
        checkAddress(addr, "connect");

        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            if (epoint.isUnresolved())
                security.checkConnect(epoint.getHostName(), port);
            else
                security.checkConnect(addr.getHostAddress(), port);
        }
        if (!created)
            createImpl(true);
        if (!oldImpl)
            impl.connect(epoint, timeout);
        else if (timeout == 0) {
            if (epoint.isUnresolved())
                impl.connect(addr.getHostName(), port);
            else
                impl.connect(addr, port);
        } else
            throw new UnsupportedOperationException("SocketImpl.connect(addr, timeout)");
        connected = true;
        /*
         * If the socket was not bound before the connect, it is now because
         * the kernel will have picked an ephemeral port & a local address
         */
        bound = true;
    }
```




[OKHttp源码解析(九):OKHTTP连接中三个"核心"RealConnection、ConnectionPool、StreamAllocation](https://www.jianshu.com/p/6166d28983a2)

