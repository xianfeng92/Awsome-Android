
# Proxy

  在Java中，通过 java.net.Proxy 类描述一个代理服务器,只需要__代理的类型及代理服务器地址__即可描述代理服务器的全部。
  对于HTTP代理，代理服务器地址可以通过域名和IP地址等方式来描述。


## ProxySelector

  在Java中通过 ProxySelector 为一个特定的 URI 选择代理,这个组件会读区系统中配置的所有代理，并根据调用者传入的URI，返回特定的代理服务器集合。
  由于不同系统中，配置代理服务器的方法，及相关配置的保存机制不同，该接口在不同的系统中有着不同的实现。

# okhttp Route

OkHttp3 中抽象出 Route 来描述网络数据包的传输路径，__最主要是描述了直接与其建立TCP连接的目标端点__。

```
public final class Route {
  final Address address;
  final Proxy proxy;
  final InetSocketAddress inetSocketAddress;

  public Route(Address address, Proxy proxy, InetSocketAddress inetSocketAddress) {
    if (address == null) {
      throw new NullPointerException("address == null");
    }
    if (proxy == null) {
      throw new NullPointerException("proxy == null");
    }
    if (inetSocketAddress == null) {
      throw new NullPointerException("inetSocketAddress == null");
    }
    this.address = address;
    this.proxy = proxy;
    this.inetSocketAddress = inetSocketAddress;
  }

  public Address address() {
    return address;
  }

  /**
   * Returns the {@link Proxy} of this route.
   */
  public Proxy proxy() {
    return proxy;
  }

  public InetSocketAddress socketAddress() {
    return inetSocketAddress;
  }

  public boolean requiresTunnel() {
    return address.sslSocketFactory != null && proxy.type() == Proxy.Type.HTTP;
  }

  @Override public boolean equals(@Nullable Object other) {
    return other instanceof Route
        && ((Route) other).address.equals(address)
        && ((Route) other).proxy.equals(proxy)
        && ((Route) other).inetSocketAddress.equals(inetSocketAddress);
  }

  @Override public int hashCode() {
    int result = 17;
    result = 31 * result + address.hashCode();
    result = 31 * result + proxy.hashCode();
    result = 31 * result + inetSocketAddress.hashCode();
    return result;
  }

  @Override public String toString() {
    return "Route{" + inetSocketAddress + "}";
  }
}
```

主要通过__代理服务器的信息 proxy 以及连接的目标地址（inetSocketAddress ）__描述路由。连接的目标地址根据代理类型的不同而有着不同的含义，
这主要是由不同代理协议的差异而造成的。对于无需代理的情况，连接的目标地址中包含HTTP服务器经过了DNS域名解析的IP地址及协议端口号。
对于HTTP代理，其中则包含代理服务器经过域名解析的IP地址及端口号。


## RouteSelector

  在HTTP请求的TCP连接过程，主要是找到一个 Route，然后依据代理协议的规则与特定目标建立TCP连接。对于无代理的情况，是与HTTP服务器建立TCP连接。
  在OkHttp中，对 Route 连接失败有一定的错误处理机制。OkHttp会逐个利用找到的 Route 去尝试建立TCP连接，直到找到可用的那一个。这就要求对 Route 信息有良好的管理。


  OkHttp3 借助于 RouteSelector 类管理所有的 Route 信息，并帮助选择路由。 RouteSelector 主要完成3件事：

### 1. 收集所有可用的路由

```
final class RouteSelector {
  private final Address address;
  private final RouteDatabase routeDatabase;
  private final Call call;
  private final EventListener eventListener;

  /* State for negotiating the next proxy to use. */
  private List<Proxy> proxies = Collections.emptyList();
  private int nextProxyIndex;

  /* State for negotiating the next socket address to use. */
  private List<InetSocketAddress> inetSocketAddresses = Collections.emptyList();

  /* State for negotiating failed routes */
  private final List<Route> postponedRoutes = new ArrayList<>();

  RouteSelector(Address address, RouteDatabase routeDatabase, Call call,
      EventListener eventListener) {
    this.address = address;
    this.routeDatabase = routeDatabase;
    this.call = call;
    this.eventListener = eventListener;

    resetNextProxy(address.url(), address.proxy());
  }
  ...

  /** Prepares the proxy servers to try. */
  private void resetNextProxy(HttpUrl url, Proxy proxy) {
    if (proxy != null) {
      // If the user specifies a proxy, try that and only that.
      proxies = Collections.singletonList(proxy);
    } else {
      // Try each of the ProxySelector choices until one connection succeeds.
      List<Proxy> proxiesOrNull = address.proxySelector().select(url.uri());
      proxies = proxiesOrNull != null && !proxiesOrNull.isEmpty()
          ? Util.immutableList(proxiesOrNull)
          : Util.immutableList(Proxy.NO_PROXY);
    }
    nextProxyIndex = 0;
  }
  ...
}

```
收集路由分为两个步骤：

1. 第一步收集所有的代理

   通过上面的代码可知，代理的来源有两种方式，一是外部通过 address 传入了代理，此时代理集合将包含这唯一的代理。
   该代理是我们在构造 OkHttpClient 时所设置的。设置后，该client执行的所有请求都会经过该代理。另一种方式是，借助于 ProxySelector 获取多个代理。
   用户可以通过 OkHttpClient 来设置 ProxySelector。但通常情况下，都是使用系统默认的 ProxySelector，来获取系统中配置的代理。


2. 第二步则是收集在特定代理服务器情况下的所有连接的目标地址（inetSocketAddress） 。

   收集一个特定代理服务器选择下的 连接的目标地址 因代理类型的不同而不同。对于没有配置代理的情况，会对HTTP服务器的域名进行DNS域名解析，
   并为每个解析到的IP地址创建 连接的目标地址

### 2. 选择可用的路由



### 3. 维护接失败的路由的信息

      维护接失败的路由的信息，以避免浪费时间去连接一些不可用的路由。 RouteSelector 借助于RouteDatabase 维护失败的路由的信息。


前面我们看到 RouteSelector 通过 Address 提供的 Proxy 和 ProxySelector 来收集 Proxy 信息以及连接的目标地址信息。
OkHttp3中用 Address 描述建立连接所需的配置信息，包括HTTP服务器的地址，DNS，SocketFactory，Proxy，ProxySelector及TLS所需的一些设施等等。


