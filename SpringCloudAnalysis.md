#### 服务治理部分
##### 三个主要结构：服务注册中心(提供服务注册、发现的功能)、服务提供者(遵循Eureka通信的应用即可)、服务消费者(从注册中心获取消费列表消费服务)
##### 服务注册：服务单元通过服务名、位置的方式向服务中心进行注册(以下为服务清单示例，服务中心通过心跳机制监控清单的可用性，不可用的会进行剔除)
    服务名     位置
    服务A     192.168.0.100:8000,192.168.0.101:8000
    服务B     192.168.0.100:9000,192.168.0.101:9000,192.168.0.102:9000
##### 服务发现：通过向服务名(如服务A,并不知道具体的位置:192.168.0.100:8000)发起调用，调用后，调用方通过注册中心获得服务A的位置清单，并通过负载轮询机制获得一个服务位置
##### 注册中心配置:
    eureka.client.fetch-registry=false//注册中心不需要去检索服务
    eureka.client.register-with-eureka=false//不向注册中心注册自己，当搭建高可用的时候需要配置为true
##### 注册服务提供者server:
    @EnableDiscoveryClient //该注解配置在主类上,使注册中心能够发现自己
    spring.application.name=hello-service //为服务命名，即服务名
    eureka.client.serviceUrl.defaultZone=xxxx,xxxx //配置服务中心的地址 多个注册中心(高可用的情况下以,分割)
##### 高可用的实现(to-do:实践)
    eureka.client.serviceUrl.defaultZone=peer2 //peer1的配置
    eureka.client.serviceUrl.defaultZone=peer1 //peer2的配置
##### 服务消费:发现服务(Eureka的客户端完成)，消费服务(基于HTTP、TCP的Ribbon完成)
##### 服务提供者:注册、同步、续约
##### 1.注册的时候通过发送REST把自己注册到Server(注册中心上),同时提供自己的元数据,Server利用双层Map存储这些信息
    Map<服务名,<实例名(xxx机器:8081), info>>
    eureka.client.register-with-eureka=true //需要保持默认的true值
##### 2.同步的时候通过把请求转发到另外的注册中心保持一致
##### 3.续约(ReNew)的时候通过心跳机制通知Server自己还“活着”
    eureka.instance.lease-renewal-interval-in-seconds=30 //renew任务的调用间隔
    eureka.instance.lease-expiration-duration-in-seconds=90 //服务失效的时间
##### 服务消费者:获取服务、调用、下线
##### 1.获取服务的时候发送一个REST请求到注册中心获取注册中心的服务名单，注册中心提供一个只读的列表，并每隔30s更新一次
##### 2.调用服务的时候，通过服务名获取具体的实例名及实例信息，可以利用负载策略进行调用(如Ribbon默认采用轮询的策略)
##### 3.下线的时候，通过发送REST请求到注册中心将该服务置为DOWN，并把下线时间传播出去
##### Zone、Region的概念：一个Region包含多个Zone，每个服务客户端注册到一个Zone中，每个客户端对应一个Zone、一个Region中，访问的时候，优先调用同一个Zone的服务提供方(to-do:基于zone的容错技术)
##### 服务注册中心:失效剔除、自我保护
##### 服务中心会创建一个定时任务，每隔60s将超时(90s)的任务从任务清单中剔除
##### 源码分析：
    #注册
    if (clientConfig.shouldRegisterWithEureka()) {
      instanceInfoReplicator = new instanceInfoReplicator();//定义定时任务，参数略
    }
    
    instanceInfoReplicator.start();//启动定时任务
    
    public void run() {//instanceInfoReplicator的run方法
      discoveryClient.register();
    }
    
    boolean register() {
      httpResponse = eurekaTransport.registrationClient.register(instanceInfo);//该instanceInfo为客户端提交的元数据信息
    }
 ##### 同样的 服务获取、服务续约都是如此启动定时任务，服务续约和注册成对出现，启动一个心跳线程，同样的都是发送REST请求，而服务获取则会根据是否是第一次请求进行不同的请求
 ##### 注册中心在拿到instanceInfo之后对元数据注册信息进行验证，并利用下面方法注册
     registry.register(instanceinfo,true.equals("isReplcation")) 
     this.ctxt.publishEvent();//通过事件发布机制将该服务发布出去
     super.register(info);//调用父类的register方法注册服务
     ConcurrentHashMap<appName, <instanceId, instanceInfo>> map 
 #### 负载均衡Ribbon
 ##### 基于HTTP、TCP Spring Ribbon是在客户端通过存储可用的服务清单的方式进行负载均衡的
 ##### 源码分析
      public interface LoadBalancerClient() {
        choose();//挑选服务的实例
        
        execute();//利用执行的实例执行服务
        
        reconstructURI();//利用instanceInfo 和 使用逻辑名的URI 拼出host:port的具体形式
      }
 ##### LoadBanlancerAutoConfiguration是客户端负载均衡的自动配置类，定义了LoadBalancerInterceptor，拦截客户端请求，RestTemplateCustomizer对RestTemaplate请求添加拦截
    
    
