如对网络知识不熟悉，可跳到最后补一下网络基础
**文字总结：**
先创建我们的request里面包括我们的Host请求参数等，okhttpClient的.newCall(request)方法生成一个RealCall,RealCall方法是我们的重点，首先执行enqueue方法，RealCall类会调用一个Dispatcher的线程池类，把我们一个异步Call传给Dispatcher线程池，我们Dispatcher类首先判断一下真正执行的任务的数量<64，真在执行host的是否<5，然后把我们的异步Call调用线程池中执行，而异步Call的run()方法最终还是调用RealCall的execute方法，execute方法还是调用的一个得到响应的getResponseWithInterceptorChain()，调用我们的拦截器。

***
okhttp最基本的请求
  ```
添加依赖:
implementation 'com.squareup.okhttp3:okhttp:3.14.5'
//请求客户端
        OkHttpClient okHttpClient = new OkHttpClient();
        //使用builder模式生成request对象
        Request request = new Request.Builder()
                .url("https://i0.hdslb.com/bfs/article/2b2d2f26caa0ebf07dbe617be4a5ba919eaa0724.jpg@1320w_742h.webp")
                .build();
        //请求同步
        Response response = okHttpClient.newCall(request).execute();

        //请求异步，开启一个子线程，不会阻塞
        okHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });

  ```
### 先看看OkHttp最要的类之一RealCall
  ```
#RealCall类

//异步的调用方法，参数callBack是个接口实例，最终还是会调用RealCall的execute()方法
@Override public void enqueue(Callback responseCallback) {
synchronized (this) {
  if (executed) throw new IllegalStateException("Already Executed");
  executed = true;
}
transmitter.callStart();
client.dispatcher().enqueue(new AsyncCall(responseCallback));//把一个异步Call给线程池请求执行，最终还是调用execute方法
}

#线程池类dispatcher的enqueue
synchronized void enqueue(AsyncCall call) {
   //判断正在执行的数量<64,正在执行请求Host<5
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call); //线程池执行请求，实际回到RealCall.execure()方法
    } else {
      readyAsyncCalls.add(call);
    }
  }

//内部异步Call
final class AsyncCall extends NamedRunnable {
    @Override protected void execute() {
      boolean signalledCallback = false;
      transmitter.timeoutEnter();
      try {
        Response response = getResponseWithInterceptorChain();
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
}
public abstract class NamedRunnable implements Runnable {
......
@Override 
public final void run() {
  try {
    execute();//在线程池执行的时候，又回到RealCall.execute方法
  } 
 protected abstract void execute();
}


//主要作用是调用得到响应的getResponseWithInterceptorChain()方法
@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  try {
    client.dispatcher().executed(this); //加到同步请求的队列
    return getResponseWithInterceptorChain();//调用拦截器
  } finally {
    client.dispatcher().finished(this); //通知dispatcher任务执行完了。
  }
}
  ```

###1.创建客户端okhttpClient源码:
  ```
OkHttpClient okHttpClient = new OkHttpClient();

public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  public OkHttpClient() {
       this(new Builder());
  }
//Builder.builder()方法调用这个构造函数并把builder传进来
  OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
    ...
  }

public static final class Builder {
public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
    }
 public OkHttpClient build() {
      return new OkHttpClient(this);
    }
}
}
  ```
### 2.建立Request（http报文）
Request：包括：**请求地址url,请求方法method，请求头head，请求数据requestBody,标志位tag**
  ```
Request request = new Request.Builder().url("url").build();

public final class Request {
  final HttpUrl url;
  final String method;
  final Headers headers;
  final @Nullable RequestBody body;
  final Map<Class<?>, Object> tags;

  private volatile @Nullable CacheControl cacheControl; // Lazily initialized.
//build()最终调用这个方法
  Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tags = Util.immutableMap(builder.tags);
  }
//写好默认的方法和头，url给我们写
 public static class Builder {
    @Nullable HttpUrl url;
    String method;
    Headers.Builder headers;
    @Nullable RequestBody body;

    Map<Class<?>, Object> tags = Collections.emptyMap();

    public Builder() {
     //默认使用get方法
      this.method = "GET";
      this.headers = new Headers.Builder();
    }

    Builder(Request request) {
      this.url = request.url;
      this.method = request.method;
      this.body = request.body;
      this.tags = request.tags.isEmpty()
          ? Collections.emptyMap()
          : new LinkedHashMap<>(request.tags);
      this.headers = request.headers.newBuilder();
    }

//请求头部
public Builder header(String name, String value) {
      headers.set(name, value);
      return this;
    }

public Builder addHeader(String name, String value) {
      headers.add(name, value);
      return this;
    }
public Builder method(String method, RequestBody body) {
      if (method == null) throw new NullPointerException("method == null");
      if (method.length() == 0) throw new IllegalArgumentException("method.length() == 0");
      if (body != null && !HttpMethod.permitsRequestBody(method)) {
        throw new IllegalArgumentException("method " + method + " must not have a request body.");
      }
      if (body == null && HttpMethod.requiresRequestBody(method)) {
        throw new IllegalArgumentException("method " + method + " must have a request body.");
      }
      this.method = method;
      this.body = body;
      return this;
    }
 
    public Request build() {
      if (url == null) throw new IllegalStateException("url == null");
      return new Request(this);
    }
}
  ```

### Dispatcher线程池介绍
  ```
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

 public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }

synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

synchronized void finished(Call call) {
    if (!runningSyncCalls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
  }
  ```

### 核心重点拦截器方法得到响应
getResponseWithInterceptorChain()：
**文字总结拦截器的作用：**拦截器可以通过Chain参数拿到上一个的拦截器的request，并通过执行Chain接口的proceed(request)把request传给下一个拦截器并从一个拦截器中拿到response。这是一种责任链的设计模式。
  ```
//
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;
    ...
  }
}
  ```
  ```
//实现Interceptor.Chain接口的实例
public final class RealInterceptorChain implements Interceptor.Chain {

 @Override public Request request() {
    return request;
  }

  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, transmitter, exchange);
}

  public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
      throws IOException {
          RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
        index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
         Interceptor interceptor = interceptors.get(index);
         Response response = interceptor.intercept(next);
         ...
        return response 
}
}
  ```
### 重试拦截器：RetryAndFollowUpInterceptor
**文字总结：**处理重试的拦截器，会处理一些异常，如果这个异常不是致命的则从返回一个request给下级，如果是致命的则将错误给上级。
再重新生成request时会对状态码进行检查，比如重定向的307，就会从返回的响应中获取新的路径，生成一个新的request给下级重新发起一次请求，如果得到的request为null则返回现在的response给上级。
  ```
public final class RetryAndFollowUpInterceptor implements Interceptor {
public RetryAndFollowUpInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
     //一个多路复用
    Transmitter transmitter = realChain.transmitter();

     while (true) {
      transmitter.prepareToConnect(request);

      if (transmitter.isCanceled()) {
        throw new IOException("Canceled");
      }

      Response response;
      boolean success = false;
      try {
        //在try中执行拿到的response
        response = realChain.proceed(request, transmitter, null);
        success = true;
      } catch (RouteException e) {
        //判断是否是致命的异常，如果是则抛出错误给上一级，如果不是则continue下一次请求。
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
     //根据下级返回的response，判断状态码，并重新确定request
      Request followUp = followUpRequest(response, route);

     //如果新的request是null则返回response
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

}
  ```
### 桥接拦截器：BridgeInterceptor
**文字总结：**给请求头部做一些通用的设置，判断返回的response是否用到压缩并把它变成GzipSource，和保存cookie。
  ```
public final class BridgeInterceptor implements Interceptor {
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
   //得到上一级的request
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();
    //拿到RequestBody 
    RequestBody body = userRequest.body();
   //根据RequestBody设置一些请求首部的参数
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
    //判断请求首部参数有无并添加
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
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

    //从下级拿到Response
    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
     //判断是否是gzip压缩，生成GzipSource
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

  /** Returns a 'Cookie' HTTP request header with all cookies, like {@code a=b; c=d}. */
  private String cookieHeader(List<Cookie> cookies) {
    StringBuilder cookieHeader = new StringBuilder();
    for (int i = 0, size = cookies.size(); i < size; i++) {
      if (i > 0) {
        cookieHeader.append("; ");
      }
      Cookie cookie = cookies.get(i);
      cookieHeader.append(cookie.name()).append('=').append(cookie.value());
    }
    return cookieHeader.toString();
  }
}
  ```
### 缓存拦截器：CacheInterceptor
**文字总结：**缓存拦截器会根据请求的request和拿到的缓存生成一个缓存策略，缓存策略有需要请求的request和缓存response，根据缓存策略的request和response是否为空进行下一步的操作，比如两个都为null就返回504，request为Null则直接返回本地缓存，根据下级返回的response查看是否状态码是否为304。
在缓存可用的情况下，读取本地的缓存的数据，如果没有直接去服务器，如果有首先判断有没有缓存策略，然后判断有没有过期，如果没有过期直接拿缓存，如果过期了需要添加一些之前头部信息如：If-Modified-Since ，这个时候后台有可能会给你返回 304 代表你还是可以拿本地缓存，每次读取到新的响应后做一次缓存。

  ```
public final class CacheInterceptor implements Interceptor {
public CacheInterceptor(@Nullable InternalCache cache) {
    this.cache = cache;
  }

  @Override public Response intercept(Chain chain) throws IOException {
   //根据请求，判断是否有缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    //根据请求和得到的缓存拿到一个缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // 如果请求的 networkRequest 是空，缓存的 cacheResponse 是空我就返回 504。 指定该数据只从缓存获取
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

    // If we don't need the network, we're done.
    if (networkRequest == null) {
    //不用请求，直接返回缓存response
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    //否则，把networkRequest重新请求一遍
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // 如果本地的缓存不为空，再次请求得到的response的状态码为304，则复用之前的response给上级
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
}
  ```
### 连接拦截器：ConnectInterceptor
**文字总结：**

findHealthyConnection() 找一个连接，首先判断有没有健康的，没有就创建（建立Scoket,握手连接），连接缓存。
封装 HttpCodec 里面封装了 okio 的 Source（输入） 和 Sink (输出)，我们通过 HttpCodec 就可以操作 Socket的输入输出，我们就可以像服务器写数据和读取返回数据
  ```
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    Transmitter transmitter = realChain.transmitter();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
   //建立一个多路复用
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

    return realChain.proceed(request, transmitter, exchange);
  }
}
  ```
### CallServerInterceptor
文字总结：写数据和读取数据
写头部信息，写body表单信息等等
  ```
public final class CallServerInterceptor implements Interceptor {
  private final boolean forWebSocket;

  public CallServerInterceptor(boolean forWebSocket) {
    this.forWebSocket = forWebSocket;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Exchange exchange = realChain.exchange();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    exchange.writeRequestHeaders(request);

    boolean responseHeadersStarted = false;
    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        exchange.flushRequest();
        responseHeadersStarted = true;
        exchange.responseHeadersStart();
        responseBuilder = exchange.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        if (request.body().isDuplex()) {
          // Prepare a duplex body so that the application can send a request body later.
          exchange.flushRequest();
          BufferedSink bufferedRequestBody = Okio.buffer(
              exchange.createRequestBody(request, true));
          request.body().writeTo(bufferedRequestBody);
        } else {
          // Write the request body if the "Expect: 100-continue" expectation was met.
          BufferedSink bufferedRequestBody = Okio.buffer(
              exchange.createRequestBody(request, false));
          request.body().writeTo(bufferedRequestBody);
          bufferedRequestBody.close();
        }
      } else {
        exchange.noRequestBody();
        if (!exchange.connection().isMultiplexed()) {
          // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
          // from being reused. Otherwise we're still obligated to transmit the request body to
          // leave the connection in a consistent state.
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

    Response response = responseBuilder
        .request(request)
        .handshake(exchange.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // server sent a 100-continue even though we did not request one.
      // try again to read the actual response
      response = exchange.readResponseHeaders(false)
          .request(request)
          .handshake(exchange.connection().handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();

      code = response.code();
    }

    exchange.responseHeadersEnd(response);

    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      exchange.noNewExchangesOnConnection();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
}
  ```
**拦截器总结：**首先调用retry拦截它可以在传给下级request得到response的过程中对一些非致命的异常进行再一次的请求，对于下级返回的response对状态码进行判断做再次的请求或者给上一级，对于一些致命异常也是返回给上一级。第二个是BridgeInterceptor拦截器，它会对请求的首部做一些通用的设置，并对返回的response进行是否需要压缩的判断返回gzip和保存response的cookie。第三个是cache缓存拦截，会根据请求拿到本地的缓存，并根据请求和本地缓存拿到生成的缓存策略，根据缓存策略的request和response判断是直接返回缓存，还是继续请求，并根据返回的response判断状态码是否继续返回本地缓存。第四connect连接拦截器主要是与服务器建立TCL的握手连接，这里的连接会使用多路复用，就是没有断开连接之前可以有多个请求共享一个连接，端对端是使用socket连接的。第五：CallServer拦截器主要是读写数据。
### getResponseWithInterceptorChain得到最上级的resonse响应。
  ```
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors()); //增加自己定义的拦截器
  interceptors.add(new RetryAndFollowUpInterceptor(client));//重试拦截器
  interceptors.add(new BridgeInterceptor(client.cookieJar()));//桥接拦截器，为请求头部添加数据
  interceptors.add(new CacheInterceptor(client.internalCache()));//缓存拦截器，
  interceptors.add(new ConnectInterceptor(client));//连接拦截器，负责请求和服务器连接
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(forWebSocket));//网络服务端的连接

//最终把拦截器责任链表
  Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
      originalRequest, this, client.connectTimeoutMillis(),
      client.readTimeoutMillis(), client.writeTimeoutMillis());

  boolean calledNoMoreExchanges = false;
  try {
   //reTry拦截器返回的response
    Response response = chain.proceed(originalRequest);
    if (transmitter.isCanceled()) {
      closeQuietly(response);
      throw new IOException("Canceled");
    }
    return response;
  } catch (IOException e) {
    calledNoMoreExchanges = true;
    throw transmitter.noMoreExchanges(e);
  } finally {
    if (!calledNoMoreExchanges) {
      transmitter.noMoreExchanges(null);
    }
  }
}
```

### RealConnect类
  ```
ublic final class RealConnection extends Http2Connection.Listener implements Connection {
  private static final String NPE_THROW_WITH_NULL = "throw with null exception";
  private static final int MAX_TUNNEL_ATTEMPTS = 21;

  public final RealConnectionPool connectionPool;
  private final Route route;

  // The fields below are initialized by connect() and never reassigned.
//以下字段由connect（）初始化，并且从不重新分配。

  /** 低级TCP套接字. */
  private Socket rawSocket;

  private Socket socket;
  //通过这个对象握手
  private Handshake handshake;
  private Protocol protocol; //协议
  private Http2Connection http2Connection; //http2.0
  private BufferedSource source; //输入流
  private BufferedSink sink; //输出流
  
  ```

**总结：**OkHttp的底层是通过Java的Socket发送HTTP请求与接受响应的(这也好理解，HTTP就是基于TCP协议的)，但是OkHttp实现了连接池的概念，即对于同一主机的多个请求，其实可以公用一个Socket连接，而不是每次发送完HTTP请求就关闭底层的Socket，这样就实现了连接池的概念。而OkHttp对Socket的读写操作使用的OkIo库进行了一层封装。

![image.png](https://upload-images.jianshu.io/upload_images/7730759-59d42bd05934e7cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## http的相关知识
HTTP,FTP,DNS,TCP,UDP,IP
OSI七层协议：**应用层、表示层、会话层、传输层、网络层、数据链路层、物理层**
五层：**应用层、传输层、网络层、数据链路层、物理层**
![image.png](https://upload-images.jianshu.io/upload_images/7730759-74a352e7f09a31fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### TCP的三次握手和四次分手
**文字总结：**
三次握手之后才开始发送数据
客户端先发一条信息给服务端说能听到吗
服务端就发送一条信息并带ACK给客户端说我能听到，你能听到吗
客户端就发送一条信息给服务端说我能听到

四次挥手：
挂了
好的。挂了
好的
![image.png](https://upload-images.jianshu.io/upload_images/7730759-ef7dc1398e8746a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Http报文
请求报文(Qequest)：请求头(首部)+空行+请求数据
响应报文(Response)：响应头(首部)+空行+响应数据
###Http首部
![image.png](https://upload-images.jianshu.io/upload_images/7730759-47e8f03e3a4d5dc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/7730759-2359a4e14fe27a67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**请求首部**
Accept：用户代理可处理的媒体类型
Accept-Charset：优先的字符集
Accept-Language：优先的语言（自然语言）
Accept-Encoding：优先的内容编码
If-Modified-Since：比较资源的更新时间
If-Range：资源未更新时发送实体 Byte 的范围请求
Cookie: 设置Cookie

**响应首部**
Cache-Control：控制缓存的行为
set-Cookie: 设置Cookie
Location：令客户端重定向至指定 URI
Expires：实体主体过期的日期时间
Last-Modified：资源的最后修改日期时间
Status Code: 响应的状态码

**cookie:**服务端发过来的信息，在客户端保存并再请求的时候发给服务端，cookie有个过期时间
**session**:在服务端保存客户端的信息，断开链接时则失效
**token:**服务端给客户端的一个id身份号码。

### Http缓存
cache-Control(缓存策略)：Public、private、no_cache、max_age、no-store(不缓存)
Expires(缓存的过期策略)：指名了缓存数据有效的绝对时间，告诉客户端到了这个时间点（比照客户端时间点）后本地缓存就作废了。
如果缓存过期再去请求服务器时，不一定拿到数据（304）

### Http状态码
1xx: Infomational (信息状态码) ，接收的请求正在处理
2xx: Succeed(成功)，请求正常处理完毕,如 200，204
3xx: Redirection(重定向)，需要进行附加操作，一般是没有响应数据返回的，如 304（Not,modified）307
4xx: Client Error (客户端的错误)，服务器无法处理请求，如 404，405
5xx: Server Error (服务端的错误)，服务器处理请求出错，如 500，502

### Http和Https的区别
https=http+加密验证

http的缺点：
1.数据没有加密传输，可能被监听
2.不验证通信方的身份容易被伪装
3.无法验证报文的完成性可能被篡改

TLS/SSL协议：
加密：对称加密(AES,DES)+非对称加密(RSA,DSA)
证书：建立连接的速度会被拖慢，TCP 8次握手

### Http1.0和Http2.0的区别
http2.0:
1.使用二进制传输不是文本
2.可以多路复用
3.报头使用了压缩
4.让服务器可以主动推送到客户端


