# CacheInterceptor

RealCall#getResponseWithInterceptorChain中会创建一个CacheInterceptor：

```
interceptors.add(new CacheInterceptor(client.internalCache()));

```

首先我们看 CacheInterceptor 的构造函数：

```
public final class CacheInterceptor implements Interceptor {
  final @Nullable InternalCache cache;

  public CacheInterceptor(@Nullable InternalCache cache) {
    this.cache = cache;
  }
```

再看看 client.internalCache()：

```
  // 这里的cache来自于构建 OkHttpClient 时设置的 cache，默认是为 null 
  @Nullable InternalCache internalCache() {
    return cache != null ? cache.internalCache : internalCache;
  }

```

1. OkHttpClient 中有2个跟缓存有关的变量，一个是 Cache，一个是 internalCache，其默认初始值都为null

2. internalCache 是 Cache 的一个内部变量，我们设置了 Cache 也就有了 internalCache。internalCache 中的
   它的一系列get、put方法，都是调用的 Cache 的方法

3. 当我们在构建 OkHttpClient 时，没有设置 Cache，即不需要缓存功能，则 request 经过 CacheInterceptor 时会直接传给下一个拦截器


在构建 OkHttpClient 时，通过Cache的构造函数传入我们想要存放缓存的文件路径以及缓存文件大小即可。比如：

```
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .connectTimeout(15, TimeUnit.SECONDS)
                .readTimeout(15,TimeUnit.SECONDS)
                .writeTimeout(15,TimeUnit.SECONDS)
                .cache(new Cache(getExternalCacheDir(),20*1024*1024))
                .build();
```

下面我们回到缓存拦截器 CacheInterceptor 中，看看具体的缓存逻辑。主要的逻辑都在 intercept 方法中。

```
@Override public Response intercept(Chain chain) throws IOException {
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;// 如果我们没有设置缓存或是当前request没有缓存，那么cacheCandidate就为null了

    long now = System.currentTimeMillis();
    //获取具体的缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    //后面会根据networkRequest和cacheResponse是否为空来做相应的操作
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    ...

    return response;
  }
```

可以看到 intercept 方法中会调用缓存策略工厂的get方法获取缓存策略 CacheStrategy。我们进入CacheStrategy看看:

```
public final class CacheStrategy {
    ...

    public Factory(long nowMillis, Request request, Response cacheResponse) {
      this.nowMillis = nowMillis;
      this.request = request;
      this.cacheResponse = cacheResponse;
      ...

    /**
     * Returns a strategy to satisfy {@code request} using the a cached response {@code response}.
     */
    public CacheStrategy get() {
      CacheStrategy candidate = getCandidate();

      if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
        // We're forbidden from using the network and the cache is insufficient.
        return new CacheStrategy(null, null);
      }

      return candidate;
    }

    // 根据不同的情况返回CacheStrategy
    private CacheStrategy getCandidate() {
      // 如果缓存中的 Response 为null
      if (cacheResponse == null) {
        return new CacheStrategy(request, null);
      }

      // 如果缓存中的response缺少必要的握手信息
      if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
      }

      // 根据 request 和 response 是否能被缓存来生成CacheStrategy
      if (!isCacheable(cacheResponse, request)) {
        return new CacheStrategy(request, null);
      }

      CacheControl requestCaching = request.cacheControl();
      if (requestCaching.noCache() || hasConditions(request)) {
        return new CacheStrategy(request, null);
      }

      CacheControl responseCaching = cacheResponse.cacheControl();
      ...

      if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        return new CacheStrategy(null, builder.build());
      }
      ...
}
```

可以看到在 CacheStrategy 的内部工厂类 Factory 中有一个 getCandidate 方法，会根据具体的情况生成 CacheStrategy 类返回，是个典型的简单工厂模式。
生成的 CacheStrategy 中有2个变量， networkRequest 和 cacheResponse，如果 networkRequest 为 null，则表示不进行网络请求。
而如果 cacheResponse 为 null，则表示没有有效的缓存。


下面我们继续看 intercept 中的方法。

```
//如果 networkRequest 和 cacheResponse 都为 null，则表示不请求网络而缓存又为null，那就返回504，请求失败
if (networkRequest == null && cacheResponse == null) {
  return new Response.Builder()
      .request(chain.request())
      .protocol(Protocol.HTTP_1_1)
      .code(504)
      .message("Unsatisfiable Request (only-if-cached)")
      .body(Util.EMPTY_RESPONSE)
      .sentRequestAtMillis(-1L)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();
}

//如果不请求网络，但是存在缓存，那就直接返回缓存，不用请求网络
if (networkRequest == null) {
    return cacheResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .build();
}

//否则的话就请求网络，就会调用下一个拦截器链，将请求转发到下一个拦截器
Response networkResponse = null;
try {
    networkResponse = chain.proceed(networkRequest);
} finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
    }
}
```

根据上述源码我们可以总结出缓存拦截器中的缓存策略:

1. 首先会尝试从缓存Cache中获取当前请求request的缓存response，并根据传入的请求request和获取的缓存response通过缓存策略对象CacheStrategy的工厂类
   的get方法生成缓存策略类CacheStrategy。

2. CacheStrategy的工厂类的get方法里面会根据一些规则生成CacheStrategy，这里面的规则实际上控制着缓存拦截器CacheInterceptor的处理逻辑。
   而这些规则都是根据请求的Request和缓存的Response的header头部信息来生成的，比如是否有noCache标志位，是否是immutable不可变的，以及缓存是否过期等。

3. CacheInterceptor会根据CacheStrategy中的networkRequest和cacheResponse是否为空，来判断是请求网络还是直接使用缓存。






