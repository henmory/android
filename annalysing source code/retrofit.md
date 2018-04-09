1.retrofit是一个对okhttp的网络请求封装，它不是一个框架，但是很多人说它是一个网络请求的框架

2.为什么要使用retrofit？

  1.retorfit很好的封装了okhttp，使网络请求和封装分开
  
  2.它把传统的网络请求转换成了java接口，并通过注解的方式或者这些参数，使得参数的获取和网络请求解藕
  
  3.同时，它支持rxjava和转换器
  
  4.因为rxjava是一种响应式链式编程，支持异步编程，避免地狱
  
  5.正因为retrofit可以灵活配置，使得它在网络请求这块应用非常的广

3.retrofit是怎么把参数获取和网络请求分开的
  
  1.通过注解的方式，把请求转换成java接口
  
  2.利用反射和jdk动态代理机制结合注解，可以把注解转换成一个obsesrvable接口===CallObservable
    
    1.把method转换成一个ServiceMethod，它里面还有method的相关信息（注解，参数，callAdapter，responseConverter）  
    
      ServiceMethod<Object, Object> serviceMethod =(ServiceMethod<Object, Object>) loadServiceMethod(method);
    
    2.通过methodService和args生成一个okhttp  OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
    
    3.把okhttpcall进一步封装成call
    
    4.通过calladapter转换成需要的call（observable）
  
  3.使得参数的获取直接通过调用相应的接口就可以实现
  
  4.同时可以在使用retrofit的时候，灵活设置calladapter和convert
