# CacheInterceptor


首先我们看 CacheInterceptor 的构造函数：

```
public final class CacheInterceptor implements Interceptor {
  final @Nullable InternalCache cache;

  public CacheInterceptor(@Nullable InternalCache cache) {
    this.cache = cache;
  }
```

通过在 RealCall 内部添加 CacheInterceptor 拦截器的方式可知，CacheInterceptor中的InternalCache来自OkHttpClient中:


```
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {

  ...

  final @Nullable Cache cache;
  final @Nullable InternalCache internalCache;
 
  public OkHttpClient() {
    this(new Builder());
  }
  public static final class Builder {
   
    ...
    /** Sets the response cache to be used to read and write cached responses. */
    public Builder cache(@Nullable Cache cache) {
      this.cache = cache;
      this.internalCache = null;
      return this;
    }
    ...
}

```

1. OkHttpClient 中有2个跟缓存有关的变量，一个是Cache，一个是internalCache。其中我们可以通过 Builder 来设置Cache，但是不能设置internalCache。
   默认Cache和internalCache都是null，也就是OkHttpClient没有默认的缓存实现。

2. 缓存拦截器CacheInterceptor中的internalCache来自OkHttpClient的Cache，因为OkHttpClient中的internalCache一直是null，我们没法从外界设置，
   所以如果我们没有为OkHttpClient设置Cache，那么缓存拦截器中的internalCache就也为null了，也就没法提供缓存功能。

从上面的源码中我们还发现，internalCache 虽然不能从外界设置，但是它却是cache的一个内部变量。下面我们来具体看看缓存Cache的实现。

```
public final class Cache implements Closeable, Flushable {

   ...
  // internalCache的方法内部调用的都是Cache的方法
  final InternalCache internalCache = new InternalCache() {
    @Override public @Nullable Response get(Request request) throws IOException {
      return Cache.this.get(request);
    }

    @Override public @Nullable CacheRequest put(Response response) throws IOException {
      return Cache.this.put(response);
    }

    @Override public void remove(Request request) throws IOException {
      Cache.this.remove(request);
    }

    @Override public void update(Response cached, Response network) {
      Cache.this.update(cached, network);
    }

    @Override public void trackConditionalCacheHit() {
      Cache.this.trackConditionalCacheHit();
    }

    @Override public void trackResponse(CacheStrategy cacheStrategy) {
      Cache.this.trackResponse(cacheStrategy);
    }
  };
  // 缓存是用DiskLruCache实现的
  final DiskLruCache cache;
  ...

  // 缓存为key-value形式，其中key为请求的URL的md5值
  public static String key(HttpUrl url) {
    return ByteString.encodeUtf8(url.toString()).md5().hex();
  }

  @Nullable Response get(Request request) {
    String key = key(request.url());
    DiskLruCache.Snapshot snapshot;
    Entry entry;
    try {
      snapshot = cache.get(key);
      if (snapshot == null) {
        return null;
      }
    } catch (IOException e) {
      // Give up because the cache cannot be read.
      return null;
    }

    try {
      entry = new Entry(snapshot.getSource(ENTRY_METADATA));
    } catch (IOException e) {
      Util.closeQuietly(snapshot);
      return null;
    }

    Response response = entry.response(snapshot);

    if (!entry.matches(request, response)) {
      Util.closeQuietly(response.body());
      return null;
    }

    return response;
  }
}

```

这一看我们就明白了，internalCache 的确是 Cache 的一个内部变量，我们设置了Cache也就有了internalCache。而实际上，internalCache是一个接口类型的变量，它的一系列get、put方法，
都是调用的Cache的方法，这也是外观模式的一种典型应用。当然这里internalCache从名字也可以看出是给内部其他对象调用的，所以internalCache和Cache的职责很明确，
Cache供外部设置，而internalCache供内部调用。同时我们看到Cache内部是通过DiskLruCache来实现缓存的，缓存的key就是request的URL的md5值，缓存的值就是Response。我们设置自己的缓存时，可以通过Cache的构造函数传入我们想要存放缓存的文件路径以及缓存文件大小即可。比如：

```
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .connectTimeout(15, TimeUnit.SECONDS)
                .readTimeout(15,TimeUnit.SECONDS)
                .writeTimeout(15,TimeUnit.SECONDS)
                .cache(new Cache(getExternalCacheDir(),20*1024*1024))
                .build();
```

下面我们回到缓存拦截器CacheInterceptor中，看看具体的缓存逻辑。主要的逻辑都在 intercept 方法中。

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
      // 如果缓存中的Response为null
      if (cacheResponse == null) {
        return new CacheStrategy(request, null);
      }

      // 如果缓存中的response缺少必要的握手信息
      if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
      }

      // 根据request和response是否能被缓存来生成CacheStrategy
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

可以看到在CacheStrategy的内部工厂类Factory中有一个getCandidate方法，会根据具体的情况生成CacheStrategy类返回，是个典型的简单工厂模式。
生成的CacheStrategy中有2个变量，networkRequest和cacheResponse，如果networkRequest为null，则表示不进行网络请求。
而如果cacheResponse为null，则表示没有有效的缓存。当我们没有设置缓存Cache时，显然 cacheResponse 始终都会为null。

下面我们继续看intercept中的方法。

```
//如果networkRequest和cacheResponse都为null，则表示不请求网络而缓存又为null，那就返回504，请求失败
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
