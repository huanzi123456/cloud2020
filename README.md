关于尚硅谷SpringCloud2020的学习项目

## 项目简介:

该项目是跟着尚硅谷周阳的Springcloud2020做出来的，里面涉及传统的 eureka、hystrix、ribbon，更是讲解了最新的 alibaba的 Nacos和sentinel、Seata，相当的给力。

首先在这里感谢阳哥，让我加深了对SpringCLoud的理解,写到吐的案例是真的让我不慌微服务的代码了。

该项目中有我按照视频的内容总结的思维导图，基本和阳哥的那个差不多，同样是mmap格式的。

与视频不同的是，我在思维导图中加入了一些我配置组件遇到的部分问题的解决方案，如果有和我一样，可以参考思维导图。

另外是推荐小伙伴们还是得多看官网的文档，视频只是引路人，很多东西是需要通过自己总结说出来才算正在掌握了的。

##项目结构解读:
### Eureka          ap  c一直性,a可用性  p分区容错性
+ `cloud-eureka-server7001`是eureka作为springcloud的注册中心,
与7002作为一个集群!
###服务注册,发现,调用
可查看7001read.md文件

```$xslt
eureka作为服务注册中心实现,服务注册,服务发现
    cap理论: a p
    自我保护:防止网络波动,删除了服务! 导致服务不可用!可以保存服务信息!

```
### zk代替Eureka     cp
整合步骤

1@EnableDiscoveryClient //该注解用于向使用consul或者zookeeper作为注册中心时注册服务
服务发现注解

2配置文件
```$xslt
首先安装zk,并且启动,关闭防火墙
server:
  port: 8004
#服务别名 --- 注册zookeeper到注册中心名称
spring:
  application:
    name: cloud-provider-payment
  cloud:
    zookeeper:
      connect-string: 120.79.219.152:2181

<!--SpringBoot整合Zookeeper客户端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    <exclusions>
        <!--先排除自带的zookeeper3.5.3-->
        <exclusion>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!--添加zookeeper3.4.14版本-->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.14</version>
</dependency>
```
### consul代替Eureka    cp
下载安装consul,consul --version
consul agent -dev
访问8500端口,web界面
```$xslt
pom.xml
<!--SpringCloud consul-server-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>

yml
server:
  port: 8006
spring:
  application:
    name: consul-provider-payment
#consul注册中心地址
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: ${spring.application.name}
```

### 服务降级,熔断,限流  hystrix -->停更
延迟,容错提高分布式系统弹性
服务雪崩: 调用链路出现问题
+ 降级fallback : 出现问题,给出一个备选的方案,不会  让客户体验不友好
```降级的情况: 程序运行异常,超时,服务熔断触发服务降级```
+ 熔断break: 达到服务器承载的上限,服务器避免挂掉,直接限制访问,调用服务降级的方法返回友好提示!
+ 限流flowlinit: 秒杀等高并发操作,保证有序,数量限制
#### 搭建服务cloud-provider-hystrix-payment8001   cloud-consumer-feign-hystrix-order80消费端
```$xslt
pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--监控-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- hystrix-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
<!--eureka client-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

server:
  port: 8001

yml 消费者端
server:
  port: 80
#这里只把feign做客户端用，不注册进eureka
eureka:
  client:
    #表示是否将自己注册进EurekaServer默认为true
    register-with-eureka: false
    service-url:
      #defaultZone: http://localhost:7001/eureka
      defaultZone: http://eureka7001.com:7001/eureka/
feign:  #开启对**的支持
  hystrix:
    enabled: true
业务类: 
     
    /**  服务提供端  
     * 超时访问，演示降级
     * @param id
     * @return
     */
    @HystrixCommand(fallbackMethod = "paymentInfo_TimeoutHandler",commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "5000")
    })
    public String paymentInfo_Timeout(Integer id) {
        int timeNumber = 3;
        try {
            TimeUnit.SECONDS.sleep(timeNumber);
        } catch (Exception e){
            e.printStackTrace();
        }
        return "线程池： " + Thread.currentThread().getName()
                + "   paymentInfo_OK,id:" + id + " 耗时(秒):" + timeNumber;
    }

    public String paymentInfo_TimeoutHandler(Integer id) {
        return "/(ToT)/调用支付接口超时或异常、\t" + "\t当前线程池名字" + Thread.currentThread().getName();
    }
---------------------------------------------------------------------------------------------------------------
消费端: 全局默认
@DefaultProperties(defaultFallback = "paymentInfo_Global_FallbackMethod")
public class OrderHystrixController {
    // 下面是全局fallback方法
    public String paymentInfo_Global_FallbackMethod() {
        return "Global异常处理信息，请稍后再试， /(ToT)/";
    }
    
    @HystrixCommand  #没有指明 使用全局
    public String paymentInfo_Timeout(@PathVariable("id") Integer id){
        String result = paymentHystrixService.paymentInfo_Timeout(id);
        return  result;
    }
------------------------------------------------------------------------------
使用实现类实现降级
 

------------------------------------------------------------------------------

启动类: 服务消费端
@EnableHystrix
```

#### 服务熔断
cloud-provider-hystrix-payment8001 
```$xslt
// 服务熔断
    @HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback",commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),              //是否开启断路器
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),    //请求数达到后才计算
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000"), //休眠时间窗
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"),  //错误率达到多少跳闸
    })
    public String paymentCircuitBreaker(@PathVariable("id") Integer id) {
          if(id < 0){
              throw  new RuntimeException("****id 不能为负数");
          }
          String serialNumber = IdUtil.simpleUUID();
          return  Thread.currentThread().getName() + "\t" + "调用成功，流水号：" + serialNumber;
    }

    public String paymentCircuitBreaker_fallback(@PathVariable("id") Integer id){
         return "id 不能为负数,请稍后再试， o(╥﹏╥)o id: " + id;
    }

达到一定的错误: 直接拒接访问!时间窗口过了之后,半开状态:调用通过,恢复服务调用!

```
#### 图形界面使用:dashboard9001
```$xslt
pom.xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>

主启动类:
@EnableHystrixDashboard
/**
 * 此配置是为了服务监控而配置，与服务容错本身无关，springcloud升级后的坑
 * ServletRegistrationBean因为SpringBoot的默认路径不是 “/hystrix.stream"
 * 只要在自己的项目里配置上下的servlet就可以了
 */
@Bean
public ServletRegistrationBean getServlet() {
    HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet() ;
    ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
    registrationBean.setLoadOnStartup(1);
    registrationBean.addUrlMappings("/hystrix.stream");
    registrationBean.setName("HystrixMetricsStreamServlet");
    return  registrationBean;
}

利用9001监控 8001
被监控的工程必须要有web ,actuator

访问:
http://localhost:8001/hystrix.stream
```

##服务网关(日志,)
 请查看9527read.md
 
##统一配置config 中心  外部配置
管理配置文件,可以与github整合
新建springcloud-config
pom.xml  3344
```$xslt
<!--config server-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
 <!-- 添加消息总线RabbitMQ支持 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

yml
```$xslt
server:
  port: 3344

spring:
  application:
    name: cloud-config-center
  cloud:
    config:
      server:
        git:
          #Github上的git仓库名字
          #uri: git@github.com:EiletXie/config-repo.git 
          uri: https://github.com/EiletXie/config-repo.git
          ##搜索目录.这个目录指的是github上的目录
          search-paths:
            - config-repo
      ##读取分支
      label: master

#rabbit相关配置 15672是web管理界面的端口，5672是MQ访问的端口
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/

#rabbitmq相关设置 ，暴露 bus刷新配置的端点
management:
  endpoints:
    web:
      exposure:
        include: 'bus-refresh'
```
访问: http://localhost:3344/master/config-dev.yml

-------------------------------------------------------------------------
### 3355拉取配置 3344获取到文件  (修改之后不能立马刷新,动态刷新,消息总线)
需要bootstrap.yml
```$xslt
server:
  port: 3355

spring:
  application:
    name: config-client
  cloud:
    #Config客户端配置
    config:
      label: master #分支名称
      name: config #配置文件名称
      profile: dev #读取后缀名称 上诉3个综合就是 master分支上 config-dev.yml
      uri: http://localhost:3344
      #http://localhost:3344/master/config-dev.yml
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/

#暴露监控端点
management:
  endpoints:
    web:
      exposure:
        include: "*"

pom
<!-- 添加消息总线RabbitMQ支持 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>

@RefreshScope
controller加上  注解!

然后请求:http:localhost:3355/actuator/refresh 
```

## 消息驱动stream:mq细节隐藏
(略)

## sleuth 分布式请求链路跟踪
调用链路的分析
兼容zipkin

下载zipkin jar包
zipkin-server-2.12.9-exec.jar
运行java -jar zip......
localhost:9411/zipkin
```$xslt
pom
<!-- 包含了sleuth zipkin 数据链路追踪-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>

  #监控的数据经过9411
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      #采样取值介于 0到1之间，1则表示全部收集
      probability: 1
```










    


## 官网文档传送门：

SpringCloud: https://spring.io/projects/spring-cloud/

这个网址是各springcloud组件的配置介绍，自己搭建组件环境可以考虑看这个。

Seata： https://seata.io/zh-cn/docs/overview/what-is-seata.html

分布式事务解决的框架，文档介绍很详细，推荐。

Nacos： https://nacos.io/zh-cn/docs/what-is-nacos.html

那个替代Eureka和Config的男人。

Sentinel：[https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D](https://github.com/alibaba/Sentinel/wiki/介绍)

在Hystrix基础上增加了流控规则和持久化,alibaba体系的一员。









