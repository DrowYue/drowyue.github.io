---
title: dubbo特性介绍笔记

date: 2018-5-28 14:53:00

tags: dubbo
---

## 多协议

+ 不同服务不同协议

  ```xml
  <!-- 多协议配置 -->
  <dubbo:protocol name="dubbo" port="20880" />
  <dubbo:protocol name="rmi" port="1099" />
  <!-- 使用dubbo协议暴露服务 -->
  <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" protocol="dubbo" />
  <!-- 使用rmi协议暴露服务 -->
  <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" protocol="rmi" />
  ```

+ 多协议暴露服务

  ```xml
  <!-- 多协议配置 -->
  <dubbo:protocol name="dubbo" port="20880" />
  <dubbo:protocol name="hessian" port="8080" />
  <!-- 使用多个协议暴露服务 -->
  <dubbo:service id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" protocol="dubbo,hessian" />
  ```

  ​

## 多注册中心

+ 多注册中心注册

  ```xml
  <!-- 多注册中心配置 -->
  <dubbo:registry id="hangzhouRegistry" address="10.20.141.150:9090" />
  <dubbo:registry id="qingdaoRegistry" address="10.20.141.151:9010" default="false" />
  <!-- 向多个注册中心注册 -->
  <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="hangzhouRegistry,qingdaoRegistry" />
  ```

+ 不同服务使用不同注册中心

  ```xml
  <!-- 多注册中心配置 -->
  <dubbo:registry id="chinaRegistry" address="10.20.141.150:9090" />
  <dubbo:registry id="intlRegistry" address="10.20.154.177:9010" default="false" />
  <!-- 向中文站注册中心注册 -->
  <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="chinaRegistry" />
  <!-- 向国际站注册中心注册 -->
  <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" registry="intlRegistry" />
  ```



## 分组聚合

+ 配置

  ```xml
  provider.xml:
  <bean id="mergeService" class="com.alibaba.dubbo.examples.merge.impl.MergeServiceImpl"/>
  <dubbo:service group="merge" interface="com.alibaba.dubbo.examples.merge.api.MergeService" ref="mergeService"/>

  <bean id="mergeService2" class="com.alibaba.dubbo.examples.merge.impl.MergeServiceImpl2"/>
  <dubbo:service group="merge2" interface="com.alibaba.dubbo.examples.merge.api.MergeService" ref="mergeService2"/>
  ```

  ```xml
  consumer.xml:
  <dubbo:reference id="mergeService" interface="com.alibaba.dubbo.examples.merge.api.MergeService" group="*"/>
  ```

  ```java
  List<String> result = mergeService.mergeResult();
  ```

+ 合并结果扩展

  ```java
  com.alibaba.dubbo.rpc.cluster.merger.ArrayMerger
  com.alibaba.dubbo.rpc.cluster.merger.ListMerger
  com.alibaba.dubbo.rpc.cluster.merger.SetMerger
  com.alibaba.dubbo.rpc.cluster.merger.MapMerger
  ```

+ 自定义扩展

  指定合并策略， 缺省根据返回值类型自动匹配， 如果同类型有两个合并器时，需指定合并器的名称

  ```java
   @SPI
    public interface Merger<T> {
        T merge(T... items);
    }
  ```

  ```xml
  <dubbo:reference id="mergeService" interface="com.alibaba.dubbo.examples.merge.api.MergeService" group="*" merger="xxxMerger"/>
  ```
  https://dubbo.gitbooks.io/dubbo-dev-book/content/impls/merger.html



## 缓存

+ 在消费者端进行结果缓存，用于加速热数据的访问速度， Dubbo提供声明式缓存，以减少用户加缓存的工作量

  ```
  threadlocal=com.alibaba.dubbo.cache.support.threadlocal.ThreadLocalCacheFactory
  lru=com.alibaba.dubbo.cache.support.lru.LruCacheFactory
  jcache=com.alibaba.dubbo.cache.support.jcache.JCacheFactory
  ```

  ```xml
  <dubbo:reference id="cacheService" interface="com.alibaba.dubbo.examples.cache.api.CacheService" cache="true"/>
  ```

+ lru 基于最近最少使用原则删除多余缓存，保持最热的数据被缓存，默认保存1000。

+ threadlocal 当前线程缓存，比如一个页面渲染，用到很多 portal，每个 portal 都要去查用户信息，通过线程缓存，可以减少这种多余访问。

+ jcache 与 JSR107 集成，可以桥接各种缓存实现。

  ​

## 异步调用

+ 基于 NIO 的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务， 相对多线程开销较小

  ![](http://7xjcj5.com1.z0.glb.clouddn.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180523203445.png)

  ```xml
  <dubbo:reference id="asyncService" interface="com.alibaba.dubbo.examples.async.api.AsyncService" async="true"/>
  ```



## 本地存根

+ 远程服务后，客户端通常只剩下接口，而实现全在服务器端，但提供方有些时候想在客户端也执行部分逻辑，那么就在服务消费者这一端提供了一个Stub类，然后当消费者调用provider方提供的dubbo服务时，客户端生成 Proxy 实例，这个Proxy实例就是我们正常调用dubbo远程服务要生成的代理实例，然后消费者这方会把 Proxy 通过构造函数传给消费者方的Stub ，然后把 Stub 暴露给用户，Stub 可以决定要不要去调 Proxy。会通过代理类去完成这个调用，这样在Stub类中，就可以做一些额外的事，来对服务的调用过程进行优化或者容错的处理。

  ![](http://7xjcj5.com1.z0.glb.clouddn.com/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180524132735.png)



## 本地伪装

+ 通常用于服务降级，比如某验权服务，当服务提供方全部挂掉后，客户端不抛出异常，而是通过 Mock 数据返回授权失败。

+ Mock是Stub 的一个子集，便于服务提供方在客户端执行容错逻辑

  ```xml
  <dubbo:reference interface="com.foo.BarService" mock="true" />
  <dubbo:reference interface="com.foo.BarService" mock="com.foo.BarServiceMock" />
  ```

  ```java
  package com.foo;
  public class BarServiceMock implements BarService {
      public String sayHello(String name) {
          // 你可以伪造容错数据，此方法只在出现RpcException时被执行
          return "容错数据";
      }
  }
  ```

  ```java
  Offer offer = null;
  try {
      offer = offerService.findOffer(offerId);
  } catch (RpcException e) {
     logger.error(e);
  }
  ```

  ```xml
  <dubbo:reference interface="com.foo.BarService" mock="return null" />
  ```



## 并发控制

+ 客户端并发控制

  ```xml
  <!-- 设置com.test.UserServiceBo接口中所有方法，每个方法最多同时并发请求10个请求 -->
  <dubbo:reference id="userService" interface="com.test.UserServiceBo" group="dubbo" version="1.0.0" timeout="3000" actives="10"/>
  ```

  ```xml
  <!-- 设置接口中的单个方法的并发请求个数 -->
  <dubbo:reference id="userService" interface="com.test.UserServiceBo" group="dubbo" version="1.0.0" timeout="3000">
  	<dubbo:method name="sayHello" actives="10" />
  </dubbo:reference>
  ```

  如果客户端请求该方法并发超过了10则客户端会被阻塞，等客户端并发请求数量少于10的时候，该请求才会被发送						到服务提供方服务器，客户端并发控制是使用ActiveLimitFilter过滤器来控制的。

+ 服务端并发控制

  ```xml
  <!-- 设置接口中所有方法，每个方法最多同时并发处理10个请求 -->
  <dubbo:service interface="com.test.UserServiceBo" ref="userService" group="dubbo"  version="1.0.0" timeout="3000" executes="10"/>
  ```

  ```xml
  <!-- 设置接口中的单个方法的并发请求个数 -->
  <dubbo:service interface="com.test.UserServiceBo" ref="userService" group="dubbo" version="1.0.0" timeout="3000" >
    <dubbo:method name="sayHello" executes="10" />
  </dubbo:service>
  ```

  服务提供方设置并发数量后，如果同时请求数量大于了设置的executes的值，则会抛出异常，而不是像消费端设置actives时候，会等待。服务提供方并发控制是使用ExecuteLimitFilter过滤器实现的。

+ Load Balance 均衡

  ```xml
  <dubbo:reference interface="com.foo.BarService" loadbalance="leastactive" />
  <dubbo:service interface="com.foo.BarService" loadbalance="leastactive" />
  ```

  可选值：random，roundrobin，leastactive（随机、轮循、最少活跃连接数）



## 连接控制

+ 服务端连接控制

  ```xml
  <dubbo:provider protocol="dubbo" accepts="10" />
  <dubbo:protocol name="dubbo" accepts="10" />
  ```

+ 客户端连接控制

  如果是长连接，例如 Dubbo 协议，connections 表示该服务对每个提供者建立的长连接数

  ```xml
  <dubbo:reference interface="com.foo.BarService" connections="10" />
  <dubbo:service interface="com.foo.BarService" connections="10" />
  ```

  com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#getClients

  com.alibaba.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke

  [Dubbo源代码实现五：RPC中的服务消费方实现](http://manzhizhen.iteye.com/blog/2354986)



## 线程栈自动dump

+ 当业务线程池满时，我们需要知道线程都在等待哪些资源、条件，以找到系统的瓶颈点或异常点。dubbo通过Jstack自动导出线程堆栈来保留现场，方便排查问题默认策略:

  - 导出路径，user.home标识的用户主目录
  - 导出间隔，最短间隔允许每隔10分钟导出一次

  指定导出路径：

  ```xml
  # dubbo.properties
  dubbo.application.dump.directory=/tmp

  <dubbo:application>
      <dubbo:parameter key="dump.directory" value="/tmp" />
  </dubbo:application>
  ```