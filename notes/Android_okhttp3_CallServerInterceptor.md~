

```
public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain)chain;
        Exchange exchange = realChain.exchange();
        Request request = realChain.request();
        long sentRequestMillis = System.currentTimeMillis();
        //写入请求头
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

        Response response = responseBuilder.request(request).handshake(exchange.connection().handshake()).sentRequestAtMillis(sentRequestMillis).
          receivedResponseAtMillis(System.currentTimeMillis()).build();
        int code = response.code();
        if (code == 100) {
            response = exchange.readResponseHeaders(false).request(request).handshake(exchange.connection().handshake())
            .sentRequestAtMillis(sentRequestMillis).receivedResponseAtMillis(System.currentTimeMillis()).build();
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

在OkHttp里面读取数据主要是通过以下四个步骤来实现的:

1. 写入请求头
2. 写入请求体
3. 读取响应头
4. 读取响应体

