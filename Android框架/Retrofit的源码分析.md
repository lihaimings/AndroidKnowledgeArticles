我们将从以下四个核心点出发：
1. 接口动态代理：Retrofit 是如何通过接口将网络请求封装的？
2. 请求构建：注解是怎么被解析并生成网络请求的？
3. 网络请求执行：Retrofit 如何结合 OkHttp 发起请求？
4. 数据转换：如何将请求参数序列化，以及响应数据反序列化？

---

## 一、接口动态代理：Retrofit.create(Class)

我们平时使用 Retrofit 时，最常见的一步是调用 `Retrofit.create()` 方法，将定义的接口转化为实际的网络请求对象：

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build();

ApiService service = retrofit.create(ApiService.class);
```

这里的 `create()` 方法到底干了什么？它利用了 Java 的动态代理机制，将接口方法的调用和实际的网络请求绑定起来。

### 1. `create()` 的实现

`Retrofit.create()` 方法的核心是用 Java 自带的 `Proxy.newProxyInstance()` 动态生成一个接口的实现类。当我们调用接口里的方法时，实际会被拦截并交由 `InvocationHandler` 处理。

```java
@SuppressWarnings("unchecked")
public <T> T create(final Class<T> service) {
    // 校验接口是否合法
    Utils.validateServiceInterface(service);

    // 动态代理
    return (T) Proxy.newProxyInstance(service.getClassLoader(),
            new Class<?>[]{service},
            (proxy, method, args) -> {
                // 如果调用的是默认方法，直接执行
                if (method.isDefault()) {
                    return Utils.invokeDefaultMethod(method, service, proxy, args);
                }
                // 加载 ServiceMethod
                ServiceMethod<?> serviceMethod = loadServiceMethod(method);
                // 调用接口方法时执行网络请求
                return serviceMethod.invoke(args);
            });
}
```

简单来说，这里做了三件事：
1. 校验接口是否合法（是否包含 Retrofit 支持的注解，比如 `@GET`、`@POST` 等）。
2. 通过动态代理拦截接口方法调用。
3. 将接口方法解析成一个 `ServiceMethod`，负责后续的请求构建和执行。

---

## 二、请求构建：注解解析与请求生成

Retrofit 的核心功能之一是通过注解（如 `@GET`、`@POST` 等）来描述网络请求，省去了手动拼接 URL 和构建参数的麻烦。那么，这些注解是怎么解析的？又是如何生成网络请求的呢？

### 1. `ServiceMethod` 的解析逻辑

每个接口方法会被解析成一个 `ServiceMethod` 对象，这个对象包含了所有关于请求的信息，比如 URL、HTTP 方法、请求参数等。

注解解析的逻辑主要在 `RequestFactory` 中完成：

```java
static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    Annotation[] methodAnnotations = method.getAnnotations(); // 方法级注解
    Annotation[][] parameterAnnotations = method.getParameterAnnotations(); // 参数级注解

    for (Annotation annotation : methodAnnotations) {
        if (annotation instanceof GET) {
            httpMethod = "GET";
            relativeUrl = ((GET) annotation).value();
        } else if (annotation instanceof POST) {
            httpMethod = "POST";
            relativeUrl = ((POST) annotation).value();
        }
        // 解析更多注解，比如 Header、Multipart 等
    }

    for (int i = 0; i < parameterAnnotations.length; i++) {
        // 根据参数注解解析请求参数（如 Query、Body）
    }
}
```

这里的注解解析过程大致可以分为两步：
1. 解析方法级注解：比如 `@GET("/users")` 定义了请求类型和 URL。
2. 解析参数级注解：比如 `@Query("id")` 和 `@Body`，定义了具体的请求参数。

最终，这些解析结果会被封装成一个 `Request` 对象，交由 OkHttp 发起请求。

---



## 三、网络请求执行

在 Retrofit 中，网络请求的执行是通过 OkHttp 来实现的。Retrofit 并没有重新发明轮子，而是将请求的实际执行交由 OkHttp 完成。这部分逻辑主要涉及到以下几个关键点：

### 1. Retrofit 的 `Call` 接口

Retrofit 定义了一个 `Call` 接口，用来抽象化网络请求：

```java
public interface Call<T> {
    Response<T> execute() throws IOException; // 同步执行
    void enqueue(Callback<T> callback);      // 异步执行
    boolean isExecuted();
    void cancel();
    boolean isCanceled();
    Call<T> clone();
}
```

开发者通常使用的是 `Call`，但实际上 Retrofit 的 `Call` 是通过 `OkHttpCall` 来实现的。

---

### 2. `OkHttpCall` 的实现

`OkHttpCall` 是 `Call` 接口的默认实现类，它封装了 OkHttp 的 `Call` 对象：

```java
final class OkHttpCall<T> implements Call<T> {
    private final ServiceMethod<T, ?> serviceMethod; // 解析好的请求信息
    private final Object[] args;                    // 方法调用时传入的参数
    private volatile boolean canceled;

    private okhttp3.Call rawCall;  // OkHttp 的 Call
    private Throwable creationFailure; // 创建错误
    private boolean executed; // 标记是否已经执行过
}
```

`OkHttpCall` 的执行逻辑分为同步和异步两种。

#### 同步执行：`execute()`

同步请求是通过 OkHttp 的 `Call.execute()` 方法完成的。源码如下：

```java
@Override
public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
        if (executed) throw new IllegalStateException("Already executed.");
        executed = true;

        // 构建 OkHttp 的 Call 对象
        call = rawCall == null ? (rawCall = createRawCall()) : rawCall;
    }

    if (canceled) {
        call.cancel();
    }

    // 调用 OkHttp 的 execute() 方法
    okhttp3.Response rawResponse = call.execute();

    // 转换响应体
    return parseResponse(rawResponse);
}
```

流程解析：
1. 检查是否重复执行（一个 `Call` 对象只能执行一次）。
2. 构建 OkHttp 的 `Call` 对象，调用 OkHttp 的 `execute()` 方法执行请求。
3. 请求成功后，通过 `parseResponse()` 转换响应数据。

#### 异步执行：`enqueue()`

异步请求则是通过 OkHttp 的 `enqueue()` 方法完成：

```java
@Override
public void enqueue(final Callback<T> callback) {
    okhttp3.Call call;

    synchronized (this) {
        if (executed) throw new IllegalStateException("Already executed.");
        executed = true;

        // 构建 OkHttp 的 Call 对象
        call = rawCall == null ? (rawCall = createRawCall()) : rawCall;
    }

    if (canceled) {
        call.cancel();
    }

    // 调用 OkHttp 的 enqueue() 方法
    call.enqueue(new okhttp3.Callback() {
        @Override
        public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
            try {
                // 转换响应
                Response<T> response = parseResponse(rawResponse);
                callback.onResponse(OkHttpCall.this, response);
            } catch (Throwable t) {
                callback.onFailure(OkHttpCall.this, t);
            }
        }

        @Override
        public void onFailure(okhttp3.Call call, IOException e) {
            callback.onFailure(OkHttpCall.this, e);
        }
    });
}
```

流程解析：
1. 和同步请求一样，检查重复执行并创建 `rawCall`。
2. 调用 OkHttp 的 `enqueue()`，将请求放入队列异步执行。
3. 收到响应后，通过 `parseResponse()` 转换数据，然后回调给开发者。

---

### 3. `rawCall` 的创建：`createRawCall()`

在同步和异步请求中，都会调用 `createRawCall()` 方法构建 OkHttp 的 `Call` 对象：

```java
private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args); // 构建请求
    return serviceMethod.callFactory.newCall(request); // 创建 OkHttp Call
}
```

这里涉及两个关键点：
1. `serviceMethod.toRequest(args)`：将接口方法的参数转换为 OkHttp 的 `Request`。
2. `callFactory.newCall(request)`：创建 OkHttp 的 `Call` 对象。

---

### 4. 响应解析：`parseResponse()`

收到响应后，Retrofit 会将 OkHttp 的 `Response` 转换为 Retrofit 的 `Response`：

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    T body = serviceMethod.toResponse(rawBody); // 转换响应体
    return Response.success(body, rawResponse);
}
```

其中，`toResponse()` 方法会将原始的 `ResponseBody` 转换为用户定义的类型，具体的逻辑依赖于数据转换器（见下一部分）。

---

## 四、数据转换：序列化与反序列化

Retrofit 支持多种数据格式的序列化和反序列化（如 JSON、XML 等），主要依赖于 `Converter` 进行实现。

### 1. `Converter` 接口

`Converter` 是一个通用的转换接口，用于序列化请求参数和反序列化响应数据：

```java
public interface Converter<F, T> {
    T convert(F value) throws IOException;
}
```

- **请求序列化**：将请求参数转换为 `RequestBody`。
- **响应反序列化**：将 `ResponseBody` 转换为对象。

---

### 2. 请求体的序列化

当我们在接口方法中传递一个 `@Body` 参数时，Retrofit 会使用 `RequestBodyConverter` 将它序列化：

```java
final class GsonRequestBodyConverter<T> implements Converter<T, RequestBody> {
    private final Gson gson;
    private final Type type;

    GsonRequestBodyConverter(Gson gson, Type type) {
        this.gson = gson;
        this.type = type;
    }

    @Override
    public RequestBody convert(T value) throws IOException {
        // 将对象转换为 JSON 字符串
        String json = gson.toJson(value);
        return RequestBody.create(MediaType.parse("application/json; charset=UTF-8"), json);
    }
}
```

流程解析：
1. 使用 Gson 将对象转换为 JSON 字符串。
2. 将 JSON 字符串封装为 OkHttp 的 `RequestBody`。

---

### 3. 响应体的反序列化

当我们调用接口方法并收到响应时，Retrofit 会使用 `ResponseBodyConverter` 将数据反序列化：

```java
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
    private final Gson gson;
    private final Type type;

    GsonResponseBodyConverter(Gson gson, Type type) {
        this.gson = gson;
        this.type = type;
    }

    @Override
    public T convert(ResponseBody value) throws IOException {
        try (Reader reader = value.charStream()) {
            // 将 JSON 字符串转换为对象
            return gson.fromJson(reader, type);
        }
    }
}
```

流程解析：
1. 从 `ResponseBody` 中获取 JSON 字符串。
2. 使用 Gson 将 JSON 字符串反序列化为指定类型的对象。

---

### 4. 数据转换器的注册

在使用 Retrofit 时，我们通常会通过 `addConverterFactory()` 方法注册数据转换器。例如：

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build();
```

`GsonConverterFactory` 的作用是为请求和响应生成合适的 `Converter` 实例。以下是其核心实现：

```java
@Override
public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
    return new GsonResponseBodyConverter<>(gson, type);
}

@Override
public Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    return new GsonRequestBodyConverter<>(gson, type);
}
```

---

## 总结

Retrofit 的网络请求执行依赖于 OkHttp，结合 `OkHttpCall` 实现了同步和异步的请求逻辑；数据转换则通过 `Converter` 接口完成，支持请求体的序列化和响应体的反序列化。