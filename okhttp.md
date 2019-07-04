##基本用法介绍
okhttp一直是一个应用非常广泛的网络框架。
首先看一下okhttp的基本用法

```
		var client = OkHttpClient()
        var request = Request.Builder().url("http://www.baidu.com").get().build()
        var call = client.newCall(request)
        call.execute()
        call.enqueue(object : Callback {
            override fun onFailure(call: Call?, e: IOException?) {
            }

            override fun onResponse(call: Call?, response: Response?) {
                System.out.println(response?.body().toString())
            }

        })
```
这里为了方便 我把同步请求和异步请求写在一起了
##okhttp的构造方法
那么正式开始源码分析，首先看第一行代码也就是okhttp的构造方法
```
var client = OkHttpClient()
```
点进去看
```
  public OkHttpClient() {
    this(new Builder());
  }
  
OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
    this.protocols = builder.protocols;
    this.connectionSpecs = builder.connectionSpecs;
    this.interceptors = Util.immutableList(builder.interceptors);
    this.networkInterceptors = Util.immutableList(builder.networkInterceptors);
    this.eventListenerFactory = builder.eventListenerFactory;
    this.proxySelector = builder.proxySelector;
    this.cookieJar = builder.cookieJar;
    this.cache = builder.cache;
    this.internalCache = builder.internalCache;
    this.socketFactory = builder.socketFactory;

    boolean isTLS = false;
    for (ConnectionSpec spec : connectionSpecs) {
      isTLS = isTLS || spec.isTls();
    }

    if (builder.sslSocketFactory != null || !isTLS) {
      this.sslSocketFactory = builder.sslSocketFactory;
      this.certificateChainCleaner = builder.certificateChainCleaner;
    } else {
      X509TrustManager trustManager = systemDefaultTrustManager();
      this.sslSocketFactory = systemDefaultSslSocketFactory(trustManager);
      this.certificateChainCleaner = CertificateChainCleaner.get(trustManager);
    }

    this.hostnameVerifier = builder.hostnameVerifier;
    this.certificatePinner = builder.certificatePinner.withCertificateChainCleaner(
        certificateChainCleaner);
    this.proxyAuthenticator = builder.proxyAuthenticator;
    this.authenticator = builder.authenticator;
    this.connectionPool = builder.connectionPool;
    this.dns = builder.dns;
    this.followSslRedirects = builder.followSslRedirects;
    this.followRedirects = builder.followRedirects;
    this.retryOnConnectionFailure = builder.retryOnConnectionFailure;
    this.connectTimeout = builder.connectTimeout;
    this.readTimeout = builder.readTimeout;
    this.writeTimeout = builder.writeTimeout;
    this.pingInterval = builder.pingInterval;

    if (interceptors.contains(null)) {
      throw new IllegalStateException("Null interceptor: " + interceptors);
    }
    if (networkInterceptors.contains(null)) {
      throw new IllegalStateException("Null network interceptor: " + networkInterceptors);
    }
  }
```
咋一看好像有点长，其实只是做了一个操作，OkHttpClient()的构造方法调用了了自己带参数的构造方法
this(new Builder());同时新建了一个Builder对象。
在带builder参数的构造方法中，进行了赋值，那么继续看new Builder()方法，看下这些值是哪里来的
```
 public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
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
      pingInterval = 0;
    }
```
可以看到okhttpclient在这里初始化了很多的值，有的值是可以修改的，有些值是私有的。特别注意一下第一个值
```
dispatcher = new Dispatcher();
```
可以叫他为任务调度器，这是okhttp一个很重要的功能，这里只需要知道，它在这里已经初始化了

那么第一行代码okhttpClient的构造方法就看完了。总的来说 就是做了一些初始化的操作
接下来看第二行代码
##构造request
```
 var request = Request.Builder().url("http://www.baidu.com").get().build()
```
首先点进去看Request.Builder()
```
public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
```
代码很少，可以看到okhttp默认会给你的request方法设置为get方式请求，同时新建了一个Heads.builder()
接下来.url("http://www.baidu.com")就是设置请求的url
.get()是设置请求的方式，所以同时也会有.post().put()等等
最后.build()点进去看下
```
public Request build() {
      if (url == null) throw new IllegalStateException("url == null");
      return new Request(this);
    }
 Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tag = builder.tag != null ? builder.tag : this;
  }
```
这里就和第一行代码的功能很像了，就是进行一些初始化的操作
那么总结一下
```
var request = Request.Builder().url("http://www.baidu.com").get().build()
```
这句话的主要工作就是构造了一个request，同时可以设置request的请求方式，需要请求的url，并在build方法里面进行了一些初始化的操作，最后得到我们设置好的request
##构建call用来执行同步或者异步的请求
接下来看第三行代码
```
 var call = client.newCall(request)
```
传入了我们构造好的request，点进去看一下
```
@Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }
```
我将这些相关的方法，放到了一起，方便大家观看，但是其实这是属于不同界面的方法
首先看newcall方法，创建了一个RealCall,这个RealCall才是真正的call也就是说真正执行同步请求和异步请求的call就是
RealCall，这里的代码很简单，也是初始化一些操作，并且传入了request到了RealCall中，
注意这里初始化了一个拦截器
 this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
 这个拦截器叫做重定向拦截器，我会在后面统一分析这些拦截器，拦截器也是okhttp里面非常重要的一个功能，加上前面说的任务调度器Dispatcher，这里已经有俩个重点的，这俩个也是我觉得在okhttp中最核心的功能
 那好继续分析，这里只需要注意这个拦截器在这里初始化了，留下一个印象而已
##同步请求
前面分析的3行代码看来都只是做了一些初始化的工作，和一些基本的设置。那么真正的功能和难点都是在具体的请求中了，同时不管是同步请求还是异步请求，前面的三步操作都是相同的。接下来首先分析同步的请求
```
call.execute()
```
点进去看下

```
  Response execute() throws IOException;
```
发现好像只是一个call接口的方法，还记得我们前面说过吗，真正的call是realCall，所以我们可以直接在realCall中找execute方法，也可以点击图片的这个按钮，直接跳转到它的真正的实现类
![这里写图片描述](https://img-blog.csdn.net/20180711160848703?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```
 @Override public Response execute() throws IOException {
    synchronized (this) {
     //如果executed为true会抛出异常，这里的作用就是指让executed执行一次，如果已经在运行了，就会抛出这个异常
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    //通知eventListener调用callStart方法
    eventListener.callStart(this);
    try {
    //这里出现了重点，dispatcher类的executed方法被执行
      client.dispatcher().executed(this);
      //另外一个重点，调用了一系列的拦截器链，得到了result
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
    //通知eventListener调用callFailed方法
      eventListener.callFailed(this, e);
      throw e;
    } finally {
    //不管成功和失败都是执行的dispatcher的finished方法.
      client.dispatcher().finished(this);
    }
  }
```
这是realcall类中的的execute方法，我会着重分析里面比较重要的语句，添加了注释，一些不是很重要的就不分析了，因为深究其中的细节会变的特别的繁琐

我们首先看下
```
client.dispatcher().executed(this);
```
```
/** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```
只做了一个操作，将这个call添加到了runningSyncCalls中
那么这个runningSyncCalls是什么呢，
我们来看下Dispatcher这个类中定义的一些属性
```
private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;
  
 /** Executes calls. Created lazily. */
  private @Nullable ExecutorService executorService;
  
  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```
俩个异步队列，一个同步队列，
异步队列一个为准备队列readyAsyncCalls，一个为运行队列runningAsyncCalls，同时还维护了一个线程池executorService，竟然是线程池，当然有最大线程数，可以看到maxRequests = 64;也就是说最大的request数量为64个，那么如果超过这个数量了，就会存储在readyAsyncCalls队列中，等待执行。
好接下来继续回到realcall的execute方法中
```
Response result = getResponseWithInterceptorChain();
```
这里通过调用一系列的拦截器链得到了response，okhttp中有一系列的拦截器，他们链式调用，每个拦截器执行他们特定的功能，最后得到response，每个拦截器后面会具体分析，这里只是简单的提一下，就像上面的Dispatcher任务调度器一样，这里都只是简单介绍，后面在涉及到的时候都会具体分析，这些只分析了相关的一些属性和方法
方法的最后调用了
```
client.dispatcher().finished(this);
```
我们点进去看一下
```
  /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
	    //在这个call执行完毕之后，在runningSyncCalls中移除它
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      //队列调度具体看下面方法的分析
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }

private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();
		
		//当runningCallsForHost的大小小于maxRequestsPerHost的时候 讲这个call从readyAsyncCalls中拿出来添加到runningAsyncCalls中，同时移除readyAsyncCalls中的对象，这里也就是上面说的，俩个队伍之间的作用
      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```
根据方法名我们可以看出这里是收尾工作，主要看我注释的介绍，
到这里整个同步方法就介绍完了，同步方法，相对于异步方法会简单一些，接下来我们开始看异步执行
##异步请求
```
call.enqueue(object : Callback {
            override fun onFailure(call: Call?, e: IOException?) {
            }

            override fun onResponse(call: Call?, response: Response?) {
                System.out.println(response?.body().toString())
            }

        })
```
异步和同步一个明显的区别就是调用异步方法的时候我们需要传入一个callback回调
点进去看下
```
 void enqueue(Callback responseCallback);
```
还是一个接口，按照上面的方法找到它的具体实现类
```
 @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
主要看最后一句
client.dispatcher().enqueue(new AsyncCall(responseCallback));
首先看AsyncCall
```
final class AsyncCall extends NamedRunnable {
   ...

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```
它继承了NamedRunnable我们看下NamedRunnable，
```
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = Util.format(format, args);
  }

  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```
AsyncCall本身没有重写run方法，在父类NamedRunnable中可以看到，run方法中调用了execute，所以最后还是会执行到AsyncCall的execute方法中，同时这里需要知道AsyncCall它是一个Runnable
好这个传递进去的参数已经大概知道了，接下来点进去Dispatcher的enqueue方法

```
synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```
可以看到这里runningAsyncCalls添加了call同时线程池开始执行，也就是AsyncCall的execute方法的会执行
```
 @Override protected void execute() {
      boolean signalledCallback = false;
      try {
      //调用拦截器链
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          //回调方法
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
      //和同步一样最后都会执行finished方法
        client.dispatcher().finished(this);
      }
    }
```
和同步相比，也没有多很多操作嘛
其实在这个过程中，我们分析了很多Dispatcher里面的方法，那么这个Dispatcher任务调度器，我们来总结一下它的作用
##Dispatcher任务调度器
内部维护了一个线程池，俩个异步队列，一个同步队列，同时同步方法和异步方法都会调用调度器中相应的方法，在方法中，对队列进行相应的增加，删除，调度，同时线程池也开始维护，那么整个Dispatcher的作用我们就知道了，他在okhttp的请求过程中承担了一个很重的作用
那么这个重点在我们分析同步异步的执行过程中就大概分析完了。
还记得我们前面有一个方法没有分析
```
//调用拦截器链
Response response = getResponseWithInterceptorChain();
```

##拦截器链getResponseWithInterceptorChain()
接下来我们分析这个拦截器链点进去看下
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
可以看到这里新建了一系列的拦截器，我们首先看最后的代码
```
  Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
```
拦截器队列传递了进去，并且调用了chain.proceed方法
我们点进去看下
```
    Response proceed(Request request) throws IOException;
```
是了接口，跟踪到实现类
```
@Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
   .....

    // Call the next interceptor in the chain.
    //注意这里也是new一个RealInterceptorChain但是有一个很大的不同是index + 1也就是说在这个地方拿到的是下一个拦截器，返回response给上一个拦截器，并且继续执行下一个拦截器，这样每个拦截器都会执行完，同时每个拦截器都会对resonse进行相对应的操作并且返回，这就是拦截器的作用
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    //调用每个拦截器的intercept方法
    Response response = interceptor.intercept(next);

    ...

    return response;
  }
```
这里我们重点关注我注释的那俩行
```
   // Call the next interceptor in the chain.
    //注意这里也是new一个RealInterceptorChain但是和getResponseWithInterceptorChain方法中有一个很大的不同是index + 1也就是说在这个地方传入的是下一个拦截器，同时最后会返回response给上一个拦截器，这样每个拦截器都会执行完，同时每个拦截器都会对resonse进行相对应的操作并且返回，这就是拦截器的作用
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    //调用每个拦截器的intercept方法
    Response response = interceptor.intercept(next);
```
看完最后的这俩个方法，我们继续来分析拦截器

我们按照顺序来分析
第一个
interceptors.addAll(client.interceptors());
点进去
```
public List<Interceptor> interceptors() {
    return interceptors;
  }
```
直接返回了一个interceptors那么这个interceptors哪里来的呢
继续跟踪在OkHttpClient(Builder builder) 的构造方法中我们找到了这个东西
```
this.interceptors = Util.immutableList(builder.interceptors);
```
那么这个拦截器其实是我们自定义的拦截器，在Builder的构造方法中可以传入

接下来分析第二个
```
interceptors.add(retryAndFollowUpInterceptor);
```
这里直接添加了这个拦截器，好像没有初始化? 还记得我们前面说过的吗
```
 private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }
```
在realcall的构造方法中，我们已经初始化过了这个拦截器，也就是重定向拦截器。我们看下这个拦截器是干嘛用的

###RetryAndFollowUpInterceptor重定向拦截器

```
@Override public Response intercept(Chain chain) throws IOException {
   ....
   .....
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }
	  ....
      ....
    }
  }
```
我只留了一些重要的代码，这个RetryAndFollowUpInterceptor的作用就用来进行重连，也就是在网络请求失败的情况下，会自动进行重连，上面的代码的意思就是当重连的次数大于最大的次数就释放release,我们不可能一直重连，会很影响兴许能，那么这个重连的次数是多少呢  private static final int MAX_FOLLOW_UPS = 20;最大重连次数为20
StreamAllocation是分配流的一个类，这个类很多也不是本篇的重点就不具体分析了
RetryAndFollowUpInterceptor这个拦截器的作用就说完了，这是一个用来进行网络重连的拦截器，具体的细节有兴趣的读者可以自己去看一下，这里就不扣细节了，不然俩天也扣不完

接下来看下一个拦截器
###BridgeInterceptor 桥接拦截器
还是只看他的intercept方法, 这个拦截器你大概一扫就知道是干嘛的
```

@Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

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
```
我前面说了 这个拦截器你大概一扫就知道是干嘛的，
很明显这是对头部信息进行一系列设置，比如Content-Length，Content-Type，Accept-Encoding等等


###CacheInterceptor 缓存拦截器
```

final InternalCache cache;

  public CacheInterceptor(InternalCache cache) {
    this.cache = cache;
  }
```
缓存拦截器，在缓存拦截器中定义了InternalCache，它的实现类cache是缓存拦截器的主要功能所在，主要有put方法和get方法，通过DiskLruCache算法写入，读取缓存，用Entry类来存储所有的缓存信息

###ConnectInterceptor连接拦截器
首先还是看下代码
```
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```
这里又出现了StreamAllocation，从前面可以知道我们在第一个拦截器里面初始化了这个StreamAllocation
，而真正使用是在这个拦截器中，通过
streamAllocation.newStream获得httpCodec用来处理request和response，同时streamAllocation.connection()获取真正的和网络进行IO通信的RealConnection，最后把这俩个重要的对象，传递给下一个拦截器，这就是连接拦截器核心的作用

###CallServerInterceptor
这是最后一个拦截器，是用来请求网络连接，处理响应用的，最后得到response并且返回出来



 