# okhttp.使用流程
  
  1.声明OkhttpClient对象：private static final OkHttpClient client = new OkHttpClient();
  
  2.生成请求体
  
  Request request = new Request.Builder()
  
                .url(parems)
                
                .addHeader(keyToken, tokenValue)
                
                .build();
                
  3.发起请求 :Response response = null;  response = client.newCall(request).execute();// 同步方法
  
  4.获取结果 :String string = response.body().string();
  
# response = client.newCall(request).execute() 详细流程
  
  1.client为OkHttpClient对象：OkHttpClient采用的是外观模式，提供给统一的接口供用户调用，与开发者解耦
  
  2.client.newCall（request）生成一个RealCall对象
  
  3.调用RealCall.execute()＝＝＝>RealCall类详解
 
# 重要类的说明
  
  ## RealCall类详解
  
    1.RealCall实现了call接口：
        
        public interface Call {
  
          Request request();//得到request内容

          Response execute() throws IOException; //同步请求

          void enqueue(Callback responseCallback);//异步请求

          void cancel();//取消====>继续学习

          boolean isCanceled();
          
          private Response getResponseWithInterceptorChain()；
        }
        
     2.execute()方法 ====>所有代码都运行在主线程中，包括数据的返回
  
        public Response execute() throws IOException {

          //交给Dispatcher执行，这里的executed只是添加到它的一个队列中，详细在后面介绍===>Dispathcer类详解

          client.dispatcher().executed(this);

          //核心发送请求获取响应在这里===>getResponseWithInterceptorChain方法核心讲解

          Response result = getResponseWithInterceptorChain();

          //后面继续跟踪
          client.dispatcher().finished(this);
        }
     
     3.getResponseWithInterceptorChain方法核心讲解＝＝＝这是一责任链设计模式
      
      private Response getResponseWithInterceptorChain() throws IOException {
        
        // 构建拦截器===>拦截器简介
        
        List<Interceptor> interceptors = new ArrayList<>();
        
        interceptors.addAll(client.interceptors());
        
        interceptors.add(retryAndFollowUpInterceptor);
        
        interceptors.add(new BridgeInterceptor(client.cookieJar()));
        
        interceptors.add(new CacheInterceptor(client.internalCache()));
        
        interceptors.add(new ConnectInterceptor(client));//网络链接拦截器===>网络连接拦截器详解
        
        if (!retryAndFollowUpInterceptor.isForWebSocket()) {
        
          interceptors.addAll(client.networkInterceptors());
          
        }
        
        interceptors.add(new CallServerInterceptor(//发送请求获取数据拦截器===>发送请求获取数据拦截器详解
        
            retryAndFollowUpInterceptor.isForWebSocket()));

        //构建拦截器链===>RealInterceptorChain类详解
        
        Interceptor.Chain chain = new RealInterceptorChain(
        
            interceptors, null, null, null, 0, originalRequest);
            
        //链执行    
        return chain.proceed(originalRequest);
        
      }
      
      4.void enqueue(Callback responseCallback);//异步请求 
      
      调用dispatcher的异步请求，此时代码仍然在主线程：client.dispatcher().enqueue(new AsyncCall(responseCallback));
      
      ### 拦截器介绍
      
        1.Interceptor接口:为了观察、修改请求和响应的；典型的做法是修改请求和响应的信息头headers
        
        public interface Interceptor {
        
          Response intercept(Chain chain) throws IOException; //具体的拦截过程

          interface Chain { //
          
            Request request();

            Response proceed(Request request) throws IOException;

            Connection connection();
          }
        }
        
        2.不同的拦截器有不同的功能，并且通过链链接在一起，构成一个责任链。它有很多实现类，起到不同的拦截作用，这样可以起到解耦的功能
        
      
  ## Dispathcer类详解
  
  public final class Dispatcher {
    
    private ExecutorService executorService; //线程池
    
    private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>(); // 准备执行的异步队列

    private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();//正在运行的异步队列

    private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>(); //正在运行的同步队列

    executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
       
       new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false)); //线程的创建
       
    synchronized void executed(RealCall call);
    
    synchronized void enqueue(AsyncCall call);
  }
  
  ### 方法介绍
  
  1.同步executed：runningSyncCalls.add(call);//同步执行的时候把call放到runningSyncCalls，代码运行在主线程

  2.异步enqueue:
    
    synchronized void enqueue(AsyncCall call) {
    
      if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
        
        runningAsyncCalls.add(call);
        
        executorService().execute(call);
      
      } else {
        
        readyAsyncCalls.add(call);
      
      }
  }
  
  
  所有代码运行在主线程，当运行到executorService().execute(call);时，线程池里面的线程执行call的execute方法
  
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
          
          responseCallback.onFailure(RealCall.this, e);
        
        }
      
      } finally {
        
        client.dispatcher().finished(this);
      
      }
   
   }
   
   跟同步方法不同的是相同的代码执行在线程池中的线程，当运行完之后，子线程回调我们的接口responseCallback，通常做法，我们应该在主线程声明一个handler，在responseCallback中调用handler发送消息到主线程的消息队列进而更新UI。至于是否需要这样做http://www.cnblogs.com/sunshine-anycall/p/5203788.html里面有解释
   
  ### 变量介绍
    
    1.executorService; //线程池＝＝只有在异步的时候才会用到，它去执行Realcall，同步的时候把Realcall放到正在运行的同步队列
    
    2.readyAsyncCalls;// 准备执行的异步队列
    
    3.runningAsyncCalls;//正在运行的异步队列
    
    4.runningSyncCalls; //正在运行的同步队列 
  
  ### 线程池介绍
    
    new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
       
       new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    
    1.创建的是CachedThreadPool:
      
      int corePoolSize: 0 ==核心线程数为0
                       
      int maximumPoolSize: 无穷 ===线程池线程最多数量为无穷
      
      long keepAliveTime： 60秒＝＝＝＝空闲线程最多等待60秒
       
      BlockingQueue<Runnable> workQueue：SynchronousQueue＝＝不存储元素的阻塞队列，每一个put方法必须等待一个take操作，否则不能继续添加
      
      可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程，也就是说有了任务就马上交给消费者消费
      
   2.结合readyAsyncCalls和runningAsyncCalls，当来一个任务的时候，通过Dispathcer的
   
   synchronized void enqueue(AsyncCall call) {
    
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      
      runningAsyncCalls.add(call);//添加到运行队列中
    
      executorService().execute(call);
   
   } else {
      
      readyAsyncCalls.add(call);//添加到等待队列中
   
    }
  
  }
      
  ## RealInterceptorChain类详解
    
    1.拦截器链，里面含有所有拦截器所有请求，最后调用call拦截器完成网络的收发
    
    2.类方法和成员：
    
    
      public final class RealInterceptorChain implements Interceptor.Chain {

        private final List<Interceptor> interceptors; //所有的拦截器

        private final int index; //拦截器索引

        private final Request request;//请求

        //拦截器处理
        public Response proceed(Request request, StreamAllocation streamAllocation, HttpStream httpStream,Connection connection) 

      }
    
    3.proceed类方法详解
    
      public Response proceed(Request request, StreamAllocation streamAllocation, HttpStream httpStream,Connection connection) throws IOException {

        if (index >= interceptors.size()) throw new AssertionError();

        calls++;

        // 找到下一个拦截器

        RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpStream, connection, index + 1, request);

        //目前的拦截器
        Interceptor interceptor = interceptors.get(index);

        //目前的拦截器进行处理，并把下一个拦截器传进去
        
        Response response = interceptor.intercept(next);

        return response;
      }

  
  ## ConnectInterceptor详解  网络连接拦截器
    
    public final class ConnectInterceptor implements Interceptor {

      @Override public Response intercept(Chain chain) throws IOException {
        
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        
        Request request = realChain.request();
        
        StreamAllocation streamAllocation = realChain.streamAllocation();

        boolean doExtensiveHealthChecks = !request.method().equals("GET");
        
        //通过StreamAllocation创建HttpStream====>StreamAllocation类详解,HttpStream类详解
        
        HttpStream httpStream = streamAllocation.newStream(client, doExtensiveHealthChecks);
        
        //通过StreamAllocation去连接池找是否有可复用的链接=====>ConnectionPool类详解
        
        RealConnection connection = streamAllocation.connection();

        //调用下一个拦截器，开始进行网络通信了
        return realChain.proceed(request, streamAllocation, httpStream, connection);
        
      }
      
    }
    
  ## StreamAllocation类详解＝＝＝》
  
    1.public final class StreamAllocation {
     
      public final Address address;//服务器地址
      
      private final ConnectionPool connectionPool;//连接池  

      private RealConnection connection;//真实链接
     
      public StreamAllocation(ConnectionPool connectionPool, Address address);  //目的是通过address在connectionPool中找到可复用的链接
      
      //寻找可用connection
      
      private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,boolean connectionRetryEnabled)
      
      public HttpStream newStream(OkHttpClient client, boolean doExtensiveHealthChecks);//创建HttpStream类进行通信
   
      public void acquire(RealConnection connection);//复用RealConnction后，为其添加一次StreamAllocation
      
      private void release(RealConnection connection);//使用完，减少一次
      
     }
     
   2.寻找可复用的RealConnection
   
    private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout，boolean connectionRetryEnabled) throws IOException {

      synchronized (connectionPool) {
         
          // 从线程池找可复用的Connection
          
          RealConnection pooledConnection = Internal.instance.get(connectionPool, address, this);
          
          //找到
          if (pooledConnection != null) {
            
            this.connection = pooledConnection;
            
            return pooledConnection;
          }

        }

        //没有找到，创建

        RealConnection newConnection = new RealConnection(selectedRoute);
        
        //为RealConnection添加一次网络请求数量
        
        acquire(newConnection);

       
        newConnection.connect(connectTimeout, readTimeout, writeTimeout, address.connectionSpecs(),connectionRetryEnabled);
        
        routeDatabase().connected(newConnection.route());

        return newConnection;
      }
   
   3.public HttpStream newStream(OkHttpClient client, boolean doExtensiveHealthChecks);
  
    这个方法里面要去connectionPool中寻找是否有可用的connection如果有就复用，并创建HttpStream，进行通信，RealConnection的通信都是
    
    靠HttpStream完成的；newStream会调用findConnection。
  
  ## RealConnection类详解
  
    1.Connection接口
    
      public interface Connection {

        Route route();

        Socket socket();

        Handshake handshake();

        Protocol protocol();
      }
    
   2.RealConnection实现Connection接口，并真正调用HttpStream，因为由于http版本的不同，接口会有所变化
   
   3.
   public final class RealConnection extends FramedConnection.Listener implements Connection {

      public Socket socket;//socket

      private Handshake handshake;

      private Protocol protocol;//协议
      
      public BufferedSource source;//输出流
      
      public BufferedSink sink;//输入流
      
      public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();//每个RealConnection被复用的StreamAllocation
      
      ｝
  
  ## HttpStream类详解===》http版本不同接口不同，比如http1,http2等，响应的类为Http1xStream，Http2xStream。
    
    public interface HttpStream {
    
      /** 创建请求的输出流*/
      
      Sink createRequestBody(Request request, long contentLength);

      /** 写请求的头部信息(Header) */
      
      void writeRequestHeaders(Request request) throws IOException;

      /** 将请求写入sokcet */
      
      void finishRequest() throws IOException;

      /** 读取网络返回的头部信息(header)*/
      
      Response.Builder readResponseHeaders() throws IOException;

      /** 返回网络返回的数据 */
      
      ResponseBody openResponseBody(Response response) throws IOException;

      // 异步的取消，并释放相关的资源。(connect pool的线程会自动完成)
       
      void cancel();
    }
    
  ##ConnectionPool类详解
    
     1. public final class ConnectionPool {
    
          //清理空闲connection的线程，在第一次添加RealConnection的时候启动，之后一直工作在背后
          
          private static final Executor executor = new ThreadPoolExecutor(0 ,Integer.MAX_VALUE, 60L , TimeUnit.SECONDS,new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));
          
          //清理connection的任务 
          
          private final Runnable cleanupRunnable;
          
          //realConnection的集合
          
          private final Deque<RealConnection> connections = new ArrayDeque<>();
          
          void put(RealConnection connection)；//添加connection
          
          RealConnection get(Address address, StreamAllocation streamAllocation)；//删除connection
        ｝
        
   ## CallServerInterceptor详解
   
   public Response intercept(Chain chain) throws IOException {
   
   //HttpStream写请求
   
    HttpStream httpStream = ((RealInterceptorChain) chain).httpStream();
    
    httpStream.writeRequestHeaders(request);

    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
    
      Sink requestBodyOut = httpStream.createRequestBody(request, request.body().contentLength());
      
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
      
      request.body().writeTo(bufferedRequestBody);
      
      bufferedRequestBody.close();
      
    }

    httpStream.finishRequest();
    
    //阻塞等待恢复回复
    
    Response response = httpStream.readResponseHeaders()
    
        .request(request)
        
        .handshake(streamAllocation.connection().handshake())
        
        .sentRequestAtMillis(sentRequestMillis)
        
        .receivedResponseAtMillis(System.currentTimeMillis())
        
        .build();

    if (!forWebSocket || response.code() != 101) {
    
      response = response.newBuilder()
      
          .body(httpStream.openResponseBody(response))
          
          .build();
    }

    return response;
  }
      
      
      
      
      
      
      
      
      
      
      
      
 
