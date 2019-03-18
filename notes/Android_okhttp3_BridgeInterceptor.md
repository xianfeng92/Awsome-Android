# BridgeInterceptor

其主要作用是负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应.

官方给的注释如下:

```
 Bridges from application code to network code. First it builds a network request from a user
 request. Then it proceeds to call the network. Finally it builds a user response from the network
 response.

```

即把程序员写的一个请求,构造成服务器端可识别的请求,然后去做网络请求,最后将服务器返回的响应解析成对程序员友好的响应.


其核心方法是 intercept:

```
 @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request(); // 提取出chain中的request请求
    Request.Builder requestBuilder = userRequest.newBuilder();// 实例化一个 requestBuilder 对象

    RequestBody body = userRequest.body();
    // 如果有请求体
    if (body != null) {
      // 获取请求体的 type
      MediaType contentType = body.contentType();
      if (contentType != null) {
        // 构建userRequest中的Content-Type
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


1. 提取出 chain 中的 request_client

2. 实例化一个 requestBuilder, 根据其名字我们大致可以猜出,其将要使用构建者模式来构建一个服务器所能识别的request.
   
   很多时候当我们在写一个 request,我们只是简单的指定 url,以及请求类型. 等到 request 传到 BridgeInterceptor 时其会为我们做很多事件,如其会提供
   Content-Length、Connection、Host、User-Agent、Accept-Encoding 等等关于 request Header中的相关字段的默认值.

3. 将构建好的服务器所能识别的请求传递给下一个拦截器,然后等待请求后的响应

4. 得到 networkResponse 时,构建一个 responseBuilder 对象,根据其名字我们大致可以猜出,其将要使用构建者模式来构建一个对用户友好的 response



BridgeInterceptor 主要使用了构建者模式来构建用户的请求和服务器的响应. 对于用户来说,写一个请求就变的很容易;对于获取到的响应,我们也不会做太多额外的处理.

说到底,还是下面这段话:

```
 Bridges from application code to network code. First it builds a network request from a user
 request. Then it proceeds to call the network. Finally it builds a user response from the network
 response.
```



