# BridgeInterceptor

重新把简陋的 user Request 组装成一个规范的 http request,主要做了如下两件事：

1. 接收上一层拦截器封装后的 request，然后自身对这个 request 进行处理，例如添加一些 header，处理后向下传递

2. 接收下一层拦截器传递回来的 response，然后自身对 response 进行处理，例如判断返回的 statusCode，然后进一步处理

接下来看看 BridgeInterceptor#intercept 如何处理来做这些处理的：

```
 @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request(); // 提取出chain中的request请求，即userRequest
    Request.Builder requestBuilder = userRequest.newBuilder();// 实例化一个 requestBuilder 对象

    RequestBody body = userRequest.body();
    // 如果有请求体
    if (body != null) {
      // 获取请求体的 type
      MediaType contentType = body.contentType();
      if (contentType != null) {
        // 构建userRequest中的 Content-Type
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        // 构建userRequest中的Content-Length
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

     // 当userRequest中没有制定host时,将使用request中的url作为host
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }
    // 当 userRequest 没有指定 Connection 时,默认将 Connection 设为 "Keep-Alive"
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    // 当 userRequest 没有指定 Accept-Encoding 或 Range 时,将 Accept-Encoding 默认设为 gzip
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    // 当userRequest.url()的 cookies 非空时,将 Cookie 设为 cookieHeader(cookies)
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }
    // 当 userRequest 没有指定 User-Agent时,默认设为 Version.userAgent() "okhttp/3.14.0"
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }
    // 此时将构造好的request,传给下一个拦截器进行网络请求的处理,然后得到一个 networkResponse
    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
    // 构建一个对用户友好的 responseBuilder
    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      // 如果使用了 transparentGzip , 将 networkResponse.body().source() 进行解压
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

上述代码主要做了如下几件事:

1. 提取出 chain 中的 userRequest

2. 实例化一个 requestBuilder, 其使用了构建者模式来一步一步的构建出一个 http request
   一般的 request 中，往往用户只会指定一个 URL 和 method，这个简单的 user request 是不足以成为一个 http request，
    我们还需要为它添加一些 header，如 Content-Length, Transfer-Encoding, User-Agent, Host, Connection, 和 Content-Type，
    如果这个 request 使用了 cookie，那我们还要将 cookie 添加到这个 request 中。

3. 将构建好的服务器所能识别的请求传递给下一个拦截器,然后等待响应

4. 得到 networkResponse 时，构建一个 responseBuilder 对象,使用构建者模式来构建一个对用户友好的 response


综上可知：BridgeInterceptor 使用了构建者模式来，将用户的 request 构建成一个 http request，然后将其传给下一个拦截器处理。等收到下一个拦截器的响应response后，使用构建者模式
构建出一个对用户友好的 response。


