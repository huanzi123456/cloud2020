### 服务的调用 RestTemplate + ribbon(软集成到消费者)
restTeempalte的配置
```$xslt
@Configuration
public class ApplicationContextConfig {
    @Bean
//    @LoadBalanced   可以实现负载均衡调用
    public RestTemplate getRestTemplate() {
        return  new RestTemplate();
    }
}
```
ribbon已经被包含
```$xslt
已经集成了ribbon
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

负载均衡算法
IRule 接口 算法接口放在程序包外面,

主启动类
//@RibbonClient(name = "CLOUD-PAYMENT-SERVICE",configuration = MySelfRule.class)
通过配置实现调用
```

### openfeign实现微服务调用
pom.xml
```$xslt
 <!-- openfeign -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

yml
server:
  port: 80
#这里只把feign做客户端用，不注册进eureka
eureka:
  client:
    #表示是否将自己注册进EurekaServer默认为true
    register-with-eureka: false
    service-url:
      #defaultZone: http://localhost:7001/eureka
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/

#设置feign客户端超时时间（OpenFeign默认支持ribbon）
ribbon:
  #指的是建立连接所用的时间，适用于网络状况正常的情况下，两端连接所用的实际
  ReadTimeout: 5000
  #指的是建立连接后从服务器读取到可用资源所用的时间
  ConnectTimeout: 5000

logging:
  level:
    #feign日志以什么级别监控那个接口
    com.eiletxie.springcloud.service.PaymentFeignService: debug

主启动类
@EnableFeignClients

业务类service接口:
@FeignClient(value = "CLOUD-PAYMENT-SERVICE")

method ()调用相关服务的controller里面的接口
@GetMapping(value="/payment/get/{id}")
public CommonResult<Payment> getPaymentById(@PathVariable("id") Long id);

超时控制:(默认一秒) 具体看yml配置

日志打印: 
@Configuration
public class FeignConfig {
    @Bean
    Logger.Level feignLoggerLevel() {
        return  Logger.Level.FULL;
    }
}
yml开启
```



