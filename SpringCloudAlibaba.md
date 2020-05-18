spring 停更问题!

##Nacos(Naming and Configuration) cp  ap 自动切换 易于构建云原生应用的动态服务发现,配置管理的服务管理平台
注册中心和配置中心的组合
Nacos = Eureka + Config +bus

下载安装
运行 startup.sh startup.cmd
访问:localhost:8848/nacos
(使用docker ecex -it 容器id /bin/bash 进入容器开启)

###服务注册发现(服务调用:自带负载均衡)
pom

    <!-- SpringCloud ailibaba nacos-->
    <dependency>
        <!-- SpringCloud ailibaba nacos-->
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!--监控-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
yml
    
    server:
      port: 9001
    
    spring:
      application:
        name: nacos-payment-provider
      cloud:
        nacos:
          discovery:
            #注册到8848
            server-addr: localhost:8848 #配置Nacos地址
    
    management:
      endpoints:
        web:
          exposure:
            include: '*'

主启动类:
    
    @EnableDiscoveryClient
    
------------------------------------------------------------------------
###配置中心
```$xslt
pom
 <!-- nacos config-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<!-- SpringCloud ailibaba nacos-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

bootstrap.yml

server:
  port: 3377
spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.73.128:8848 #Nacos服务注册中心地址
      config:
        server-addr: 192.168.73.128:8848 #Nacos作为配置中心地址
        file-extension: yaml #指定yaml格式配置
#        group: TEST_GROUP
#        namespace: 6fecc7ae-f02f-49ef-ace7-80d2f671df77

#${prefix}-${spring.profile.active}.${file-extension}
# ${spring.application.name}-${spring.profile.active}.${file-extension}
# nacos-config-client-dev.yaml

application.yml
spring:
  profiles:
    active: dev
  #active: dev #表示开发环境

主启动类
@EnableDiscoveryClient
@SpringBootApplication
public class NacosConfigClientMain3377 {
    public static void main(String[] args) {
        SpringApplication.run(NacosConfigClientMain3377.class,args);
    }
}

@RefreshScope // 支持Nacos的动态刷新功能
controller

nacos去创建配置文件
nacos-config-client-dev.yml
config:
    info: ****** 
```

-------------------------------------------------------------------------
namespace命名空间:
    public 
group
    (默认)default_group
cluster
    default
    
###集群与持久化!
    请求-->nginx集群--->nacos集群--->mysql集群
nacos默认使用嵌入式数据库derby,数据存下一致性问题!
可以采用集中式存储来支持集群化部署!目前支持mysql

nacos-mysql.sql执行脚本!建立数据库!
application.properties
最后添加:
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=root

linux:
修改start.sh
while getopts ":m:f:s:p:" opt
do
    case $opt in
        m)
            MODE=$OPTARG;;
        f)
            FUNCTION_MODE=$OPTARG;;
        s)
            SERVER=$OPTARG;;
		p) 
			PORT=$OPTARG;
        ?)
        echo "Unknown parameter"
        exit 1;;
    esac
done

nohup $JAVA -Dserver.port=${PORT} ${JAVA_OPT} nacos.nacos >> ${BASE_DIR}

启动
./startup.sh  -port 3333

修改cluster.conf 集群ip
10.10.109.214
11.16.128.34
11.16.128.36

nginx -----> 学习!
修改nginx.conf
    
    #gzip  on;
	#代理转发上游
	upstream cluster{
		server ip:port;
		server ip:port;
		server ip:port;
	}

    server{
	listen       1111;
	server_name  localhost;
	ssi on;
	ssi_silent_errors on;
	location / {
		#alias   D:/360Downloads/xc-ui-pc-static-portal/;
		#index  index.html;
		#代理转发
		proxy_pass http://cluster;
		}
	}
	
##sentinel 服务熔断,限流,降级    (待完善)
下载sentinel-dashboard-1.7.0.jar
运行: java -jar ****
8080端口

###工程整合
pom
```$xslt
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
<!-- SpringCloud ailibaba sentinel-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```
yml
```$xslt
server:
  port: 8401

spring:
  application:
    name: cloudalibaba-sentinal-service
  cloud:
    nacos:
      discovery:
        #Nacos服务注册中心地址
        server-addr: localhost:8848
    sentinel:
      transport:
        #配置Sentin dashboard地址
        dashboard: localhost:8080
        # 默认8719端口，假如被占用了会自动从8719端口+1进行扫描，直到找到未被占用的 端口
        # 交互端口
        port: 8719
      #添加sentinel 流控规则持久化 到nacos配置
      #启动访问: 流控规则恢复  
      datasource:
        ds1:
          nacos:
            server-addr: localhost:8848
            dataId: cloudalibaba-sentinel-service
            groupId: DEFAULT_GROUP
            data-type: json
            rule-type: flow

management:
  endpoints:
    web:
      exposure:
        include: '*'

feign:
  sentinel:
    enabled: true #激活Sentinel 对Feign的支持
```




##seate分布式事物   TCC事物补偿机制
全局事物id
Tc事物协调者(leader): 负责协调事物的提交,回滚
 
Tm事物管理器(决定者): 控制全局事物的边界,负责开启一个全局的事物,
并最终发起全局的提交或全局事物的回滚

RM资源管理器(控制本地事物):控制分支事物,接收事物协调者的指令,驱动分支(本地)事物的提交和回滚!

下载安装:
运行
操作:

建立数据库 --->data.sql

修改seata配置文件-->file.conf
自定义事物组名称
```$xslt
service {
  #transaction service group mapping
  #修改自定义事物组名称
  vgroup_mapping.my_test_tx_group = "fsp_tx_group"
  #only support when registry.type=file, please don't set multiple addresses
  default.grouplist = "127.0.0.1:8091"
  #disable seata
  disableGlobalTransaction = false
}
```

事物日志的存储模式
```$xslt
## transaction log store, only used in seata-server
store {
  ## store mode: file、db 事物日志的存储模式
  mode = "db"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
  }
  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "dbcp"
    ## mysql/oracle/h2/oceanbase etc.
    db-type = "mysql"
    driver-class-name = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata"
    user = "root"
    password = "huanzi"
  }
}
```

####业务
业务数据库(对应自己建好)
每个业务库下简历一个日志表--->undo.sql

工程编码: seata-storage-service2002
pom
```$xslt
<!-- nacos -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<!-- https://mvnrepository.com/artifact/com.google.errorprone/error_prone_core -->
<dependency>
    <groupId>com.google.errorprone</groupId>
    <artifactId>error_prone_core</artifactId>
    <version>0.92</version>
</dependency>

<!-- seata-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.seata</groupId>
            <artifactId>seata-all</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-all</artifactId>
    <version>1.0.0</version>
</dependency>
```
yml
```$xslt
server:
  port: 2001

spring:
  application:
    name: seata-order-service
  cloud:
    alibaba:
      seata:
        # 自定义事务组名称需要与seata-server中的对应
        tx-service-group: fsp_tx_group
    nacos:
      discovery:
        server-addr: 192.168.73.128:8848
  datasource:
    # 当前数据源操作类型
    type: com.alibaba.druid.pool.DruidDataSource
    # mysql驱动类
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/seata_order?useUnicode=true&characterEncoding=UTF-8&serverTimezone=GMT%2B8
    username: root
    password: huanzi
feign:
  hystrix:
    enabled: false
logging:
  level:
    io:
      seata: info

mybatis:
  mapperLocations: classpath*:mapper/*.xml
```
放入修改的file.cong,registry.conf

业务编写:


@GlobalTransactional
业务开始的方法加上

原理:
    























