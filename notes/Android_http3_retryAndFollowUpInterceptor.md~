# retryAndFollowUpInterceptor

一些常用的状态码:

* 100~199：指示信息，表示请求已接收，继续处理

* 200~299：请求成功，表示请求已被成功接收、理解、接受

* 300~399：重定向，要完成请求必须进行更进一步的操作

* 400~499：客户端错误，请求有语法错误或请求无法实现

* 500~599：服务器端错误，服务器未能实现合法的请求


有时我们请求的 URL 已经被移走了，此时 server response 会返回 301 状态码和一个重定向的新 URL，此时 okhttp 会自动访问新 URL。
当然如果 response 返回的状态码为 5xx，okhttp 可能会尝试获取一个新的路径进行重新连接。

## retry

```
 @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Transmitter transmitter = realChain.transmitter();

    int followUpCount = 0;
    Response priorResponse = null;
   //开启循环，如果请求完成或遇到异常则退出
    while (true) {
      transmitter.prepareToConnect(request);

      if (transmitter.isCanceled()) {
        throw new IOException("Canceled");
      }

      Response response;
      boolean success = false;
      try {
        response = realChain.proceed(request, transmitter, null);
        success = true;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), transmitter, false, request)) {
          throw e.getFirstConnectException();
        }
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, transmitter, requestSendStarted, request)) throw e;
        continue;
      } finally {
        // The network call threw an exception. Release any resources.
        if (!success) {
          transmitter.exchangeDoneDueToException();
        }
      }
      ...
```

通过上述代码可知，当请求过程中有 Exception 发生时，okhttp 会尝试去判断是否可以去 retry。

以下几种情况，okhttp是会去 retry：

* 应用层如果禁止 retry ，则不会重试

* 如果是协议的问题，则不会重试

* 捕获到RouteException，传入 RouteException 的getLastConnectException，若lastException为 SocketTimeoutException 可重试，否则不可重试。
  即出现连接超时，可尝试下一个路由重试

* SSLHandshakeException ssl握手异常，若是由来自X509TrustManager的CertificateException引起则不可重试

* SSLPeerUnverifiedException ssl证书校验异常不可重试

* 没有更多的路由去尝试，则不会重试


如果请求中没有异常发生，retryAndFollowUpInterceptor#intercept 会得到一个 response,接下来，其会解析 response看其是否需要重定向，
主要代码如下：

```
    ...
  // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Exchange exchange = Internal.instance.exchange(response);
      Route route = exchange != null ? exchange.connection().route() : null;
      // followUpRequest 方法内部会判断服务端返回的状态码来决定是否重定向，若返回新的request表示需要重定向，返回null表示不进行重定向。
      Request followUp = followUpRequest(response, route);

      if (followUp == null) {
        if (exchange != null && exchange.isDuplex()) {
          transmitter.timeoutEarlyExit();
        }
        return response;
      }

      RequestBody followUpBody = followUp.body();
      if (followUpBody != null && followUpBody.isOneShot()) {
        return response;
      }

      closeQuietly(response.body());
      if (transmitter.hasExchange()) {
        exchange.detachWithViolence();
      }

      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      request = followUp;
      priorResponse = response;
    }
  }
```

1. 如果 priorResponse 不为空，则说明前面已经获取到了响应，这里会把当前获取的Response和先前的Response结合起来

2. 调用 followUpRequest 查看 response 是否需要重定向，如果不需要重定向则返回 response。followUpRequest主要是根据 响应码(responseCode)和
   响应头(header)，查看是否需要重定向，并重新设置请求。当然，如果是正常响应则直接返回Response停止循环。

## 小结

   retryAndFollowUpInterceptor 主要通过一个循环来获取 response。如果获取到的 response 需要重试或者重定向，则会构建一个新的 request 再次去向服务器请求，
   直到获取到一个正常的 response 或者 满足不可重试的条件时，会退出循环。



