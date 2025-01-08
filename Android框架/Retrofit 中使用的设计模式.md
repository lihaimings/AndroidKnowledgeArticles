
## 一、工厂模式（Factory Pattern）

**工厂模式** 是一种创建对象的设计模式，通过定义一个工厂方法来创建不同类型的对象，隐藏具体对象的创建逻辑。Retrofit 中工厂模式的应用体现在多个地方，比如 `CallAdapter.Factory` 和 `Converter.Factory`。

### 1. 示例：`Converter.Factory`

Retrofit 使用 `Converter.Factory` 创建用于请求体和响应体的序列化和反序列化器（例如 Gson、Moshi 等）。

#### 源码分析

`Converter.Factory` 是一个抽象类，具体的实现类负责返回 `Converter` 实例：

```java
public abstract class Converter<F, T> {
    public interface Factory {
        @Nullable
        public Converter<ResponseBody, ?> responseBodyConverter(
            Type type, Annotation[] annotations, Retrofit retrofit);
        
        @Nullable
        public Converter<?, RequestBody> requestBodyConverter(
            Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit);
    }
}
```

- `responseBodyConverter()` 和 `requestBodyConverter()` 用于创建请求和响应的转换器。
- GsonConverterFactory 等具体工厂实现了该接口，返回对应的 `Converter` 实例。

#### 工厂模式的作用

- 根据用户添加的工厂类型（如 Gson、Moshi）动态创建序列化器。
- 使用者只需要调用 `addConverterFactory()` 即可，无需关心其具体实现。

---

## 二、建造者模式（Builder Pattern）

**建造者模式** 用于创建复杂对象，通过一步步设置参数，最后生成目标对象。Retrofit 本身是一个典型的建造者模式的实现。

### 1. 示例：`Retrofit.Builder`

Retrofit 使用 `Builder` 类来帮助用户构建 `Retrofit` 实例：

```java
public static final class Builder {
    private Platform platform;
    private okhttp3.Call.Factory callFactory;
    private HttpUrl baseUrl;
    private List<Converter.Factory> converterFactories = new ArrayList<>();
    private List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
    private Executor callbackExecutor;
    private boolean validateEagerly;

    public Builder baseUrl(String baseUrl) {
        this.baseUrl = HttpUrl.get(baseUrl);
        return this;
    }

    public Builder addConverterFactory(Converter.Factory factory) {
        converterFactories.add(Utils.checkNotNull(factory, "factory == null"));
        return this;
    }

    public Retrofit build() {
        return new Retrofit(
            callFactory,
            baseUrl,
            converterFactories,
            callAdapterFactories,
            callbackExecutor,
            validateEagerly
        );
    }
}
```

#### 建造者模式的优势

1. **链式调用**：可以通过链式调用来设置不同的参数，提高代码的可读性。
2. **扩展性强**：用户可以灵活地添加多个 `Converter.Factory` 和 `CallAdapter.Factory`。
3. **构建逻辑清晰**：将复杂的构建过程封装到 `build()` 方法中。

---

## 三、代理模式（Proxy Pattern）

**代理模式** 是通过代理对象间接访问目标对象。Retrofit 通过动态代理机制，实现了接口方法与实际网络请求的解耦。

### 1. 示例：动态代理生成接口实现

当用户调用 `Retrofit.create(Class)` 时，Retrofit 使用 JDK 的动态代理生成接口的实现类。

#### 源码分析

```java
public <T> T create(final Class<T> service) {
    return (T) Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] { service },
        new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                if (method.getDeclaringClass() == Object.class) {
                    return method.invoke(this, args);
                }
                ServiceMethod<Object, Object> serviceMethod = loadServiceMethod(method);
                OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
                return serviceMethod.callAdapter.adapt(okHttpCall);
            }
        });
}
```

- 使用 `Proxy.newProxyInstance()` 创建接口的动态代理对象。
- 当调用接口方法时，进入 `invoke()` 方法，通过 `ServiceMethod` 和 `OkHttpCall` 执行实际的请求逻辑。

#### 代理模式的作用

- 动态生成接口实现类，避免手写实现。
- 将接口方法与底层网络请求解耦，提供统一的访问方式。

---

## 四、策略模式（Strategy Pattern）

**策略模式** 定义了一组算法，将每个算法封装起来，使它们可以互相替换。Retrofit 的 `CallAdapter` 和 `Converter` 就是策略模式的典型应用。

### 1. 示例：`CallAdapter`

`CallAdapter` 决定了网络请求的返回类型（如 `Call<T>`、`Observable<T>` 等）。

#### 源码分析

```java
public interface CallAdapter<R, T> {
    Type responseType();

    T adapt(Call<R> call);

    abstract class Factory {
        public abstract @Nullable CallAdapter<?, ?> get(
            Type returnType, Annotation[] annotations, Retrofit retrofit);
    }
}
```

- `CallAdapter.Factory` 返回不同的 `CallAdapter` 实现。
- 不同的 `CallAdapter` 决定了请求的返回值类型。

#### 策略模式的作用

- 通过不同的 `CallAdapter` 实现，支持多种返回类型（如同步请求、异步请求、RxJava、LiveData 等）。
- 用户可以通过扩展 `CallAdapter.Factory` 实现自定义逻辑。

---

## 五、责任链模式（Chain of Responsibility）

**责任链模式** 将多个处理器串联成一条链，按顺序依次处理请求。Retrofit 的拦截器（Interceptor）就是责任链模式的应用。

### 1. 示例：拦截器链

OkHttp 内置了一套责任链机制，Retrofit 直接继承并使用了这一机制。

#### 源码分析

在 OkHttp 的 `RealCall` 类中，发起请求时会调用拦截器链：

```java
@Override
public Response execute() throws IOException {
    return client.newCall(this).proceed();
}

@Override
public Response proceed(Request request) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    // 调用下一个拦截器
    Interceptor next = interceptors.get(index);
    RealInterceptorChain nextChain = new RealInterceptorChain(
        interceptors, index + 1, request, call, connectTimeoutMillis, readTimeoutMillis);
    return next.intercept(nextChain);
}
```

#### 拦截器链的组成

1. **用户自定义拦截器**：可添加日志、加解密、缓存逻辑等。
2. **核心拦截器**：OkHttp 的默认拦截器处理连接池、重试等逻辑。

#### 责任链模式的作用

- 实现请求和响应的统一处理。
- 增强代码的可扩展性，用户可以自定义拦截器来实现特定功能。

---

## 六、单例模式（Singleton Pattern）

**单例模式** 保证一个类仅有一个实例，并提供全局访问点。在 Retrofit 中，OkHttpClient 就是单例模式的应用。

### 1. 示例：`OkHttpClient`

Retrofit 推荐开发者在全局范围内共享一个 `OkHttpClient` 实例，以节省资源。

```java
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new LoggingInterceptor())
    .build();

Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .client(client)
    .build();
```

---

## 总结

Retrofit 的设计中运用了多个经典的设计模式，具体如下：

| **设计模式**       | **应用场景**                                                                 |
|--------------------|------------------------------------------------------------------------------|
| 工厂模式           | `Converter.Factory` 和 `CallAdapter.Factory` 的创建。                       |
| 建造者模式         | `Retrofit.Builder` 的链式调用与配置。                                        |
| 代理模式           | 动态代理生成接口实现，解耦接口与请求逻辑。                                  |
| 策略模式           | `CallAdapter` 和 `Converter`，支持多种返回类型和数据格式。                   |
| 责任链模式         | 拦截器机制，用于处理请求和响应的多步逻辑。                                   |
| 单例模式           | 全局共享 `OkHttpClient` 实例，提升性能并节省资源。                          |

这些设计模式的结合，不仅提高了代码的灵活性和扩展性，也让开发者在使用时更加简洁明了，堪称网络库设计的典范。