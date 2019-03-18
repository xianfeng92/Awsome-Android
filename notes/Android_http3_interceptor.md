
## interceptor

```
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

getResponseWithInterceptorChain 中文意思就是使用拦截器链获取响应,该方法做了如下3件事:

1. 用一个 List 存储所有的 interceptor,主要有如下几种:

* client.interceptors() 为 在配置 OkHttpClient 时设置的 interceptors,可以通过 Builder 的addInterceptor方法添加的拦截器。

* retryAndFollowUpInterceptor 负责失败重试以及重定向

* BridgeInterceptor(桥接器) 负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应

* CacheInterceptor 负责读取缓存直接返回、更新缓存

* ConnectInterceptor 负责和服务器建立连接

* client.networkInterceptors() 配置 OkHttpClient 时设置的 networkInterceptors

* CallServerInterceptor 负责向服务器发送请求数据以及从服务器读取响应数据

2. new 一个RealInterceptorChain 对象, 该对象是啥?

```
/**
 * A concrete interceptor chain that carries the entire interceptor chain: all application
 * interceptors, the OkHttp core, all network interceptors, and finally the network caller.
 */
public final class RealInterceptorChain implements Interceptor.Chain {

```

RealInterceptorChain 可以看作是整个拦截器链的控制器，通过它可以调用链上的下一个拦截器，而每一个拦截器就相当于链上的一环,在其构造方法中传入了所有的
interceptor,originalRequest,eventListener以及client的读写以及连接的超时时间。这里值得注意的是,RealInterceptorChain构造方法中第五个参数传入的为 0，
表示的是 interceptors 列表中的索引。


3. 返回一个 chain.proceed(originalRequest), 其实该方法执行后会返回一个 Response.

经过以上分析,重点应该就在 chain.proceed(originalRequest) 了,看一看该方法:

```
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    // 如果我们已经有一个stream, 确定即将到来的request会使用它
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    // 如果我们已经有一个stream, 确定这是 chain.proceed() 所处理的唯一的call
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    // 这里index+1，所以就会调用interceptors中的下一个拦截器
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```

代码很多，但是主要是进行一些判断，主要的代码在这:

```
 // 调用链的下一个拦截器
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
```

 上面的代码主要做了如下3件事:

1. 再次实例化一个 RealInterceptorChain 对象,注意这次实例化时,第五个参数传入的是 index + 1。

2. interceptor 列表中获取 index 位置的 interceptor.如果在client中自己没有设置的 interceptor,那么这个 index 为0所对应 interceptor 就是 retryAndFollowUpInterceptor.

3. interceptor.intercept(next), 将 RealInterceptorChain 对象传入 retryAndFollowUpInterceptor, 并调用 retryAndFollowUpInterceptor 的 intercept 方法


通过上面的分析可知: response 就是 retryAndFollowUpInterceptor 对象调用 intercept 方法后的返回值.


重点又来到了 retryAndFollowUpInterceptor 的 intercept 方法：

```
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    streamAllocation = new StreamAllocation(client.connectionPool(), createAddress(request.url()),
        call, eventListener, callStackTrace);

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        // 如果是通过路由连接失败，请求将不会再发送
        if (!recover(e.getLastConnectException(), false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        // 与服务器尝试通信失败,请求不会再发送。
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        // 抛出未检查的异常，释放资源
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp = followUpRequest(response);

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }

```

上面代码的关键在 response = realChain.proceed(request, streamAllocation, null, null), 这里又调用了 RealInterceptorChain 的 proceed。只不过这里的

RealInterceptorChain在构造时的index值已经增为1。所以在 proceed 方法中 interceptors.get(index) 获取到的已经是 BridgeInterceptor 了.

然后重点又来到了 BridgeInterceptor 的 intercept 方法:

```
@Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();
    // 将用户的request转换为发送到server的请求
    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    // GZIP压缩
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }

```

上面代码主要做了如下几件事:

1. BridgeInterceptor对于 request 的格式进行检查，并构建了一个新的request

2. 调用下一个 interceptor 来得到 response

3. 对得到的response进行一些判断操作，最后将结果返回


到这里我们知道 response 还是要靠下一个 interceptor 来获取,下一个对应index为2,即是 CacheInterceptor


首先我们看CacheInterceptor的构造函数：

```
public final class CacheInterceptor implements Interceptor {
  final @Nullable InternalCache cache;

  public CacheInterceptor(@Nullable InternalCache cache) {
    this.cache = cache;
  }
```
通过在 RealCall 内部添加 CacheInterceptor 拦截器的方式可知，CacheInterceptor中的InternalCache来自 OkHttpClient 中：

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
Cache供外部设置，而internalCache供内部调用。同时我们看到Cache内部是通过DiskLruCache来实现缓存的，缓存的key就是request的URL的md5值，缓存的值就是Response。我们设置自己的缓存时，
可以通过Cache的构造函数传入我们想要存放缓存的文件路径以及缓存文件大小即可。比如：

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

根据上述源码我们可以总结出缓存拦截器中的缓存策略：

1. 首先会尝试从缓存Cache中获取当前请求request的缓存response，并根据传入的请求request和获取的缓存response通过缓存策略对象CacheStrategy的工厂类
   的get方法生成缓存策略类CacheStrategy。

2. CacheStrategy的工厂类的get方法里面会根据一些规则生成CacheStrategy，这里面的规则实际上控制着缓存拦截器CacheInterceptor的处理逻辑。
   而这些规则都是根据请求的Request和缓存的Response的header头部信息来生成的，比如是否有noCache标志位，是否是immutable不可变的，以及缓存是否过期等。

3. CacheInterceptor会根据CacheStrategy中的networkRequest和cacheResponse是否为空，来判断是请求网络还是直接使用缓存。


当 CacheInterceptor 中没有有效缓存作为response时，其会将 networkRequest 交给下一个拦截器来处理，下一个拦截器就是 ConnectInterceptor。

ConnectInterceptor#intercept 源码如下：

```
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    Transmitter transmitter = realChain.transmitter();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

    return realChain.proceed(request, transmitter, exchange);
  }
```
其中 Transmitter 为 OkHttp 的应用程序和网络层之间的桥梁，此类公开了高级应用程序层原语：连接，请求，响应和流。经过 ConnectInterceptor
拦截器，客户端会建立和目标服务器的连接。在建立连接后，ConnectInterceptor拦截器将请求又交给了 CallServerInterceptor，该拦截器主要负责
发送和接收数据。

```

    public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain)chain;
        Exchange exchange = realChain.exchange();
        Request request = realChain.request();
        long sentRequestMillis = System.currentTimeMillis();
        exchange.writeRequestHeaders(request);
        boolean responseHeadersStarted = false;
        Builder responseBuilder = null;
        if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
            if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
                exchange.flushRequest();
                responseHeadersStarted = true;
                exchange.responseHeadersStart();
                responseBuilder = exchange.readResponseHeaders(true);
            }

            if (responseBuilder == null) {
                BufferedSink bufferedRequestBody;
                if (request.body().isDuplex()) {
                    exchange.flushRequest();
                    bufferedRequestBody = Okio.buffer(exchange.createRequestBody(request, true));
                    request.body().writeTo(bufferedRequestBody);
                } else {
                    bufferedRequestBody = Okio.buffer(exchange.createRequestBody(request, false));
                    request.body().writeTo(bufferedRequestBody);
                    bufferedRequestBody.close();
                }
            } else {
                exchange.noRequestBody();
                if (!exchange.connection().isMultiplexed()) {
                    exchange.noNewExchangesOnConnection();
                }
            }
        } else {
            exchange.noRequestBody();
        }

        if (request.body() == null || !request.body().isDuplex()) {
            exchange.finishRequest();
        }

        if (!responseHeadersStarted) {
            exchange.responseHeadersStart();
        }

        if (responseBuilder == null) {
            responseBuilder = exchange.readResponseHeaders(false);
        }

        Response response = responseBuilder.request(request).handshake(exchange.connection().handshake()).sentRequestAtMillis(sentRequestMillis).receivedResponseAtMillis(System.currentTimeMillis()).build();
        int code = response.code();
        if (code == 100) {
            response = exchange.readResponseHeaders(false).request(request).handshake(exchange.connection().handshake()).sentRequestAtMillis(sentRequestMillis).receivedResponseAtMillis(System.currentTimeMillis()).build();
            code = response.code();
        }

        exchange.responseHeadersEnd(response);
        if (this.forWebSocket && code == 101) {
            response = response.newBuilder().body(Util.EMPTY_RESPONSE).build();
        } else {
            response = response.newBuilder().body(exchange.openResponseBody(response)).build();
        }

        if ("close".equalsIgnoreCase(response.request().header("Connection")) || "close".equalsIgnoreCase(response.header("Connection"))) {
            exchange.noNewExchangesOnConnection();
        }

        if ((code == 204 || code == 205) && response.body().contentLength() > 0L) {
            throw new ProtocolException("HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
        } else {
            return response;
        }
    }
```

1. 检查请求方法，用 exchange 处理request

2. 进行网络请求得到 response

3. 返回 response

# 参考

[OKHttp源码解析](https://www.jianshu.com/p/27c1554b7fee)
[拆 OkHttp](https://blog.piasy.com/2016/07/11/Understand-OkHttp/index.html)

