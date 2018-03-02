# 接口
  
  1.call接口，表示一个请求＝＝＝》RealCall
  
  2.OkhttpClinet＝＝＝＝》客户端使用.newCall(request).execute()/equeue()发起请求
  
  3.execute()===>同步方法，调用该方法线程阻塞，知道结果返回
  
  4.equeue()＝＝＝》异步方法，调用完，不阻塞，线程池执行任务，执行完之后，回调接口（运行在线程池中的线程中）
  
# 分发器（dispatcher）
  
  1.分发任务到等待队列中，如果是同步方式，放到同步等待队列中，如果是异步方式，放到异步等待队列中，并开启线程池执行任务
  
  2.如果是同步，调用线程执行任务，阻塞等待返回结果，finish
  
  3.如果是异步任务，线程池执行任务，回调用户接口，finish
  
  4.异步任务有两个等待队列，runningAsyncCalls（正在执行的任务队列）和readyAsyncCalls（已经准备好的任务队列）
  
  5.线程池采用来了任务就分配线程的线程池，保证快速执行
  
# 拦截器

  1.拦截器是建立链接获取数据的主要逻辑处理，它的主要工作是修改观察请求和回复的，典型的使用是修改请求和回复头
  
  2.主要的拦截器
  
    List<Interceptor> interceptors = new ArrayList<>();
    
    interceptors.addAll(client.interceptors());//客户端自己定义的应用层拦截器
    
    interceptors.add(retryAndFollowUpInterceptor);//重连重定位拦截器
  
    interceptors.add(new BridgeInterceptor(client.cookieJar()));//桥接拦截器
    
    interceptors.add(new CacheInterceptor(client.internalCache()));//缓存拦截器
    
    interceptors.add(new ConnectInterceptor(client));//链接拦截器
    
    if (!retryAndFollowUpInterceptor.isForWebSocket()) {
      
      interceptors.addAll(client.networkInterceptors());//客户端自己定义的网络拦截器
    
    }
    
    interceptors.add(new CallServerInterceptor(retryAndFollowUpInterceptor.isForWebSocket()));//网络链接拦截器
  
# RetryAndFollowUpInterceptor 重练和重定位拦截器
  
  1.连接失败后重练
  2.连接成功后，需要重新二次定位请求
  
  @Override public Response intercept(Chain chain) throws IOException {
    
    Request request = chain.request();

    streamAllocation = new StreamAllocation(client.connectionPool(), createAddress(request.url()));//创建streamAllocation

    while (true) {

      try {
        
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);//进行网络

      } catch (RouteException e) {
        
        continue;//连接失败，继续连接
        
      } catch (IOException e) {
        
        continue;//连接失败，继续连接
     
      } 
     
     }

      //连接成功，构建重定位处理
 
      Request followUp = followUpRequest(response);

      //没有后续操作，返回结果
      
      if (followUp == null) {
        
        return response;
      
      }

  }

# BridgeInterceptor 桥接拦截器
  
  1.把应用层的request转换成网络层的request
  
  2.把网络层的request转换成应用层的request
  
  3.代码就不贴了，主要是转换信息头的内容
  
  
# CacheInterceptor 缓存拦截器
  
  ## 大概流程
    
    1.client发起请求
    
    2.拦截器判断本地是否有缓存文件，如果没有发起无条件请求
    
    3.请求回来后，查看服务器带不带条件
    
    4.如果不带返回数据，此时客户端可以缓存到本地文件
    
    5.下次请求时，如果有缓存文件，那么发起有条件请求
    
    6.此时服务器返回有条件响应（这应该在头里面有）
    
    7.如果条件没有改变，返回的是not modify直接从缓存中读取
    
    8.如果发生变化，读取新数据
  
  
# ConnectInterceptor 链接拦截器
  
  1.主要是连接远程服务器，生存HttpStream，建立socket
  
  HttpStream httpStream = streamAllocation.newStream(client, doExtensiveHealthChecks);//每次请求生成一个HttpStream

  RealConnection connection = streamAllocation.connection();＝＝＝》RealConnection//在new Stream时找到connection从连接池中
  
  2.public HttpStream newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {

      try {
      
        RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
            
            writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);//找到可用Connection

        HttpStream resultStream;
        
        //根据http版本不同创建HttpStream
        
        if (resultConnection.framedConnection != null) {
        
          resultStream = new Http2xStream(client, this, resultConnection.framedConnection);
          
        } else {
        
          resultStream = new Http1xStream(client, this, resultConnection.source, resultConnection.sink);
        }

    }
    
    3.找可用connection
    
    private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
    
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)｛
      
      while (true) {
      
        RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,connectionRetryEnabled); 
           
        return candidate;
      }
    }

    4.
    
      private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
          
          boolean connectionRetryEnabled) throws IOException {

          synchronized (connectionPool) {

          // 线程池寻找可复用connection
          RealConnection pooledConnection = Internal.instance.get(connectionPool, address, this);
         
         if (pooledConnection != null) {

            return pooledConnection; //找到返回
          
          }

        }

        RealConnection newConnection = new RealConnection(selectedRoute);//否则新建
        
        acquire(newConnection);//把RealConnection与StreamAllocation关联，一个StreamAllocation对应一个连接，但是connection可以被复用，所以一个connection可以对应多个StreamAllocation而且，一个StreamAllocation可以对应多个HttpStream，因为对于一个服务器的连接，会有多次请求

        synchronized (connectionPool) {
          
          Internal.instance.put(connectionPool, newConnection);//放到连接池中
        
        }

        newConnection.connect(connectTimeout, readTimeout, writeTimeout, address.connectionSpecs(),connectionRetryEnabled);
        

        return newConnection;
        
      }
      
    5.newConnection.connect
      
      public void connect(int connectTimeout, int readTimeout, int writeTimeout,
          List<ConnectionSpec> connectionSpecs, boolean connectionRetryEnabled) {
        
          buildConnection(connectTimeout, readTimeout, writeTimeout, connectionSpecSelector);
            
      }
    
    6.
    
    private void buildConnection(int connectTimeout, int readTimeout, int writeTimeout,
      ConnectionSpecSelector connectionSpecSelector) throws IOException {
      
      connectSocket(connectTimeout, readTimeout);//连接socket
      
      establishProtocol(readTimeout, writeTimeout, connectionSpecSelector);
    
    }
    
    7.
    
    private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
        
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);//socket连接＝＝socket.connect(address, connectTimeout);
      
      source = Okio.buffer(Okio.source(rawSocket));//read buffer
      
      sink = Okio.buffer(Okio.sink(rawSocket));//write buffer
     
    }


  
# CallServerInterceptor 发送和接收拦截器
  
  1.发送请求并接收回复
  
   @Override public Response intercept(Chain chain) throws IOException {

    httpStream.writeRequestHeaders(request);//写请求头

    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {//写请求体
      
      Sink requestBodyOut = httpStream.createRequestBody(request, request.body().contentLength());
      
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
      
      request.body().writeTo(bufferedRequestBody);
      
      bufferedRequestBody.close();
    
    }

    httpStream.finishRequest();

    Response response = httpStream.readResponseHeaders() //读响应头
        
        .request(request)
        
        .handshake(streamAllocation.connection().handshake())
        
        .sentRequestAtMillis(sentRequestMillis)
        
        .receivedResponseAtMillis(System.currentTimeMillis())
        
        .build();

    if (!forWebSocket || response.code() != 101) {
      
      response = response.newBuilder()//读响应体
          
          .body(httpStream.openResponseBody(response))
          
          .build();
    }
    
    return response;
  
  }

    
    
