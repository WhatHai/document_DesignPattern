## 5 Apollo应用于分布式系统

在微服务架构模式下，项目往往会切分成多个微服务，下面将以万信金融P2P项目为例演示如何在项目中使用。

### 5.1 项目场景介绍

#### 5.1.1 项目概述

 万信金融是一款面向互联网大众提供的理财服务和个人消费信贷服务的金融平台，依托大数据风控技术，为用户提供方便、快捷、安心的P2P金融服务。本项目包括交易平台和业务支撑两个部分，交易平台主要实现理财服务，包括：借钱、出借等模块，业务支撑包括：标的管理、对账管理、风控管理等模块。项目采用先进的互联网技术进行研发，保证了P2P双方交易的安全性、快捷性及稳定性。

#### 5.1.2 各微服务介绍

本章节仅仅是为了演示配置中心，所以摘取了部分微服务，如下：

用户中心服务(consumer-service)：为借款人和投资人提供用户账户管理服务，包括：注册、开户、充值、提现等

UAA认证服务(uaa-service)：为用户中心的用户提供认证服务

统一账户服务(account-service)：对借款人和投资人的登录平台账号进行管理，包括：注册账号、账号权限管理等

交易中心(transaction-service)：负责P2P平台用户发标和投标功能

### 5.2 Spring Boot应用集成

下面以集成统一账户服务(account-service)为例

#### 5.2.1 导入工程

参考account-service、transaction-service、uaa-service、consumer-service工程，手动创建这几个微服务。

每个工程必须添加依赖：

```
<dependency>
  <groupId>com.ctrip.framework.apollo</groupId>
  <artifactId>apollo-client</artifactId>
  <version>1.1.0</version>
</dependency>
```

下边是account-service依赖，其它工程参考“资料”下的“微服务”。

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.pbteach</groupId>
    <artifactId>account-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>com.ctrip.framework.apollo</groupId>
            <artifactId>apollo-client</artifactId>
            <version>1.1.0</version>
        </dependency>

    </dependencies>

</project>
```

#### 5.2.2 必选配置

1. AppId：在Spring Boot application.properties或application.yml中配置

application.properties

```properties
   app.id=account-service
```

application.yml

```yaml
   app:
   	id: account-service
```

1. apollo.bootstrap

集成springboot，开启apollo.bootstrap，指定namespace

例子：

```
   apollo.bootstrap.enabled = true
   apollo.bootstrap.namespaces = application,micro_service.spring-boot-http,spring-rocketmq,micro_service.spring-boot-druid
```

1. Apollo Meta Server

Apollo支持应用在不同的环境有不同的配置，常用的指定方式有如下两种：

- 第一种：通过Java System Property的apollo.meta：`-Dapollo.meta=http://localhost:8080`

- 第二种：在resources目录下新建apollo-env.properties文件

  ```properties
   dev.meta=http://localhost:8080
   pro.meta=http://localhost:8081
  ```

1. 本地缓存路径

Apollo客户端会把从服务端获取到的配置在本地文件系统缓存一份，用于在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置，不影响应用正常运行。本地配置文件会以下面的文件名格式放置于配置的本地缓存路径下：{appId}+{cluster}+{namespace}.properties

![image-20190716144411934](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716144411934.png)

可以通过如下方式指定缓存路径，通过Java System Property的apollo.cacheDir：

```
   -Dapollo.cacheDir=/opt/data/apollo-config
```

1. Environment

通过Java System Property的env来指定环境：`-Denv=DEV`

1. Cluster（集群）

通过Java System Property的apollo.cluste来指定集群：`-Dapollo.cluster=DEFAULT`

也可以选择使用之前新建的SHAJQ集群：`-Dapollo.cluster=SHAJQ`

1. 完整的VM Options如下：

```shell
   -Denv=DEV -Dapollo.cacheDir=/opt/data/apollo-config -Dapollo.cluster=DEFAULT
```



#### 5.2.3 启用配置

在咱们应用的启动类添加`@EnableApolloConfig`注解即可：

```java
@SpringBootApplication(scanBasePackages = "com.pbteach.account")
@EnableApolloConfig
public class AccountApplication {

	public static void main(String[] args) {
		SpringApplication.run(AccountApplication.class, args);
	}
}
```

#### 5.2.4 应用配置

1. 将local-config/account.properties中的配置添加到apollo中

```
   swagger.enable=true
   sms.enable=true

   spring.http.encoding.charset=UTF-8
   spring.http.encoding.force=true
   spring.http.encoding.enabled=true
   server.use-forward-headers=true
   server.tomcat.protocol_header=x-forwarded-proto
   server.servlet.context-path=/account-service
   server.tomcat.remote_ip_header=x-forwarded-for

   spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
   spring.datasource.druid.stat-view-servlet.allow=127.0.0.1,192.168.163.1
   spring.datasource.druid.web-stat-filter.session-stat-enable=false
   spring.datasource.druid.max-pool-prepared-statement-per-connection-size=20
   spring.datasource.druid.max-active=20
   spring.datasource.druid.stat-view-servlet.reset-enable=false
   spring.datasource.druid.validation-query=SELECT 1 FROM DUAL
   spring.datasource.druid.stat-view-servlet.enabled=true
   spring.datasource.druid.web-stat-filter.enabled=true
   spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
   spring.datasource.druid.stat-view-servlet.deny=192.168.1.73
   spring.datasource.url=jdbc\:mysql\://127.0.0.1\:3306/p2p_account?useUnicode\=true
   spring.datasource.druid.filters=config,stat,wall,log4j2
   spring.datasource.druid.test-on-return=false
   spring.datasource.druid.web-stat-filter.profile-enable=true
   spring.datasource.druid.initial-size=5
   spring.datasource.druid.min-idle=5
   spring.datasource.druid.max-wait=60000
   spring.datasource.druid.web-stat-filter.session-stat-max-count=1000
   spring.datasource.druid.pool-prepared-statements=true
   spring.datasource.druid.test-while-idle=true
   spring.datasource.password=pbteach0430
   spring.datasource.username=root
   spring.datasource.druid.stat-view-servlet.login-password=admin
   spring.datasource.druid.stat-view-servlet.login-username=admin
   spring.datasource.druid.web-stat-filter.url-pattern=/*
   spring.datasource.druid.time-between-eviction-runs-millis=60000
   spring.datasource.druid.min-evictable-idle-time-millis=300000
   spring.datasource.druid.test-on-borrow=false
   spring.datasource.druid.web-stat-filter.principal-session-name=admin
   spring.datasource.druid.filter.stat.log-slow-sql=true
   spring.datasource.druid.web-stat-filter.principal-cookie-name=admin
   spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
   spring.datasource.druid.aop-patterns=com.pbteach.wanxinp2p.*.service.*
   spring.datasource.druid.filter.stat.slow-sql-millis=1
   spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*
```



1. spring-http命名空间在之前已通过关联公共命名空间添加好了，现在来添加spring-boot-druid命名空间

![image-20190919184516450](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190919184516450.png)

1. 添加本地文件中的配置到对应的命名空间，然后发布配置

![image-20190919184636687](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190919184636687.png)

1. 在account-service/src/main/resources/application.properties中配置apollo.bootstrap.namespaces需要引入的命名空间

```properties
  app.id=account-service
  apollo.bootstrap.enabled = true
  apollo.bootstrap.namespaces = application,micro_service.spring-boot-http,spring-rocketmq,spring-boot-druid

  server.port=63000
```

#### 5.2.5 读取配置

1. 启动应用

![image-20190716160200147](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716160200147.png)

1. 访问：<http://127.0.0.1:63000/account-service/hi>，确认Spring Boot中配置的context-path是否生效

![image-20190716145112160](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716145112160.png)

通过/account-service能正常访问，说明apollo的配置已生效

![image-20190716145246670](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716145246670.png)

1. 确认spring-boot-druid配置

   - 为了快速确认可以在AccountController中通过@Value获取来验证

   ```java
       @GetMapping("/db-url")
       public String getDBConfig(@Value("${spring.datasource.url}") String url) {
    return url;
       }
   ```

   - 访问<http://127.0.0.1:63000/account-service/db-url>，显示结果

   ![image-20190920100715535](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190920100715535.png)

#### 5.3.6 创建其它项目

参考account-service将其它项目也创建完成。

### 5.4 生产环境部署

当一个项目要上线部署到生产环境时，项目的配置比如数据库连接、RocketMQ地址等都会发生变化，这时候就需要通过Apollo为生产环境添加自己的配置。

#### 5.4.1 企业部署方案

在企业中常用的部署方案为：Apollo-adminservice和Apollo-configservice两个服务分别在线上环境(pro)，仿真环境(uat)和开发环境(dev)各部署一套，Apollo-portal做为管理端只部署一套，统一管理上述三套环境。

具体如下图所示：

![deployment](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/deployment.png)

下面以添加生产环境部署为例

#### 5.4.2 创建数据库

创建生产环境的ApolloConfigDB：**每添加一套环境就需要部署一套ApolloConfgService和ApolloAdminService**

source apollo/ApolloConfigDB_PRO__initialization.sql

#### 5.4.3 配置启动参数

1. 设置生产环境数据库连接
2. 设置ApolloConfigService端口为：8081，ApolloAdminService端口为8091

```shell
echo

set url="localhost:3306"
set username="root"
set password="mysqlpwd"

start "configService-PRO" java -Dserver.port=8081 -Xms256m -Xmx256m -Dapollo_profile=github -Dspring.datasource.url=jdbc:mysql://%url%/ApolloConfigDBPRO?characterEncoding=utf8 -Dspring.datasource.username=%username% -Dspring.datasource.password=%password% -Dlogging.file=.\logs\apollo-configservice.log -jar .\apollo-configservice-1.3.0.jar
start "adminService-PRO" java -Dserver.port=8091 -Xms256m -Xmx256m -Dapollo_profile=github -Dspring.datasource.url=jdbc:mysql://%url%/ApolloConfigDBPRO?characterEncoding=utf8 -Dspring.datasource.username=%username% -Dspring.datasource.password=%password% -Dlogging.file=.\logs\apollo-adminservice.log -jar .\apollo-adminservice-1.3.0.jar
```

1. 运行runApollo-PRO.bat

#### 5.4.4 修改Eureka地址

更新生产环境Apollo的Eureka地址：

```sql
USE ApolloConfigDBPRO;

UPDATE ServerConfig SET `Value` = "http://localhost:8081/eureka/" WHERE `key` = "eureka.service.url";
```

#### 5.4.5 调整ApolloPortal服务配置

服务配置项统一存储在ApolloPortalDB.ServerConfig表中，可以通过`管理员工具 - 系统参数`页面进行配置：apollo.portal.envs - 可支持的环境列表

![image-20190918171029321](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918171029321.png)

默认值是dev，如果portal需要管理多个环境的话，以逗号分隔即可（大小写不敏感），如：

```
dev,pro
```

#### 5.4.6 启动ApolloPortal

Apollo Portal需要在不同的环境访问不同的meta service(apollo-configservice)地址，所以我们需要在配置中提供这些信息。

```shell
-Ddev_meta=http://localhost:8080/ -Dpro_meta=http://localhost:8081/
```

1. 关闭之前启动的ApolloPortal服务，使用runApolloPortal.bat启动多环境配置

```shell
   echo

   set url="localhost:3306"
   set username="root"
   set password="123"

   start "ApolloPortal" java -Xms256m -Xmx256m -Dapollo_profile=github,auth -Ddev_meta=http://localhost:8080/ -Dpro_meta=http://localhost:8081/ -Dserver.port=8070 -Dspring.datasource.url=jdbc:mysql://%url%/ApolloPortalDB?characterEncoding=utf8 -Dspring.datasource.username=%username% -Dspring.datasource.password=%password% -Dlogging.file=.\logs\apollo-portal.log -jar .\apollo-portal-1.3.0.jar
```

1. 启动之后，点击account-service服务配置后会提示环境缺失，此时需要补全上边新增生产环境的配置

![image-20190917173224057](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917173224057.png)

1. 点击补缺环境

![image-20190917173313452](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917173313452.png)

1. 补缺过生产环境后，切换到PRO环境后会提示有Namespace缺失，点击补缺

![image-20190917173939122](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917173939122.png)

![image-20190917174026819](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917174026819.png)

1. 从dev环境同步配置到pro

![image-20190917174207284](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917174207284.png)

![image-20190917174239218](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917174239218.png)

#### 5.4.7 验证配置

1. 同步完成后，切换到pro环境，修改生产环境rocketmq地址后发布配置

![image-20190917174432737](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917174432737.png)

1. 配置项目使用pro环境，测试配置是否生效

   - 在apollo-env.properties中增加pro.meta=[http://localhost:8081](http://localhost:8081/)
   - 修改account-service启动参数为：-Denv=pro

   ```shell
    -Denv=pro -Dapollo.cacheDir=/opt/data/apollo-config -Dapollo.cluster=DEFAULT
   ```

   - 访问<http://127.0.0.1:63000/account-service/mq> 验证RocketMQ地址是否为上边设置的PRO环境的值

   ![image-20190917175922694](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917175922694.png)

   ### 5.5 灰度发布

   #### 5.5.1 定义

 灰度发布是指在黑与白之间，能够平滑过渡的一种发布方式。在其上可以进行A/B testing，即让一部分用户继续用产品特性A，一部分用户开始用产品特性B，如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面来。

#### 5.5.2 Apollo实现的功能

1. 对于一些对程序有比较大影响的配置，可以先在一个或者多个实例生效，观察一段时间没问题后再全量发布配置。
2. 对于一些需要调优的配置参数，可以通过灰度发布功能来实现A/B测试。可以在不同的机器上应用不同的配置，不断调整、测评一段时间后找出较优的配置再全量发布配置。

#### 5.5.3 场景介绍

apollo-quickstart项目有两个客户端：

1. 172.16.0.160
2. 172.16.0.170

![image-20190918112227794](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918112227794.png)

**灰度目标**

当前有一个配置timeout=2000，我们希望对172.16.0.160灰度发布timeout=3000，对172.16.0.170仍然是timeout=2000。

![image-20190917185254574](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917185254574.png)

#### 5.5.4 创建灰度

1. 点击application namespace右上角的`创建灰度`按钮

![image-20190917185518909](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917185518909.png)

1. 点击确定后，灰度版本就创建成功了，页面会自动切换到`灰度版本`Tab

![image-20190917185607164](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917185607164.png)

#### 5.5.5 灰度配置

1. 点击`主版本的配置`中，timeout配置最右侧的`对此配置灰度`按钮

![image-20190917190008666](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917190008666.png)

1. 在弹出框中填入要灰度的值：3000，点击提交

![image-20190917190101491](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917190101491.png)

![image-20190917190152482](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917190152482.png)

#### 5.5.6 配置灰度规则

1. 切换到`灰度规则`Tab，点击`新增规则`按钮

![image-20190917190259452](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917190259452.png)

1. 在弹出框中`灰度的IP`下拉框会默认展示当前使用配置的机器列表，选择我们要灰度的IP，点击完成

![image-20190917190347229](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917190347229.png)

![image-20190917190422866](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917190422866.png)

如果下拉框中没找到需要的IP，说明机器还没从Apollo取过配置，可以点击手动输入IP来输入，输入完后点击添加按钮

![image-20190917190517170](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917190517170.png)

#### 5.5.7 灰度发布

1. 启动apollo-quickstart项目的GrayTest类输出timeout的值

vm options: `-Dapp.id=apollo-quickstart -Denv=DEV -Ddev_meta=http://localhost:8080`

```java
   public class GrayTest {

   	// VM options:
   	// -Dapp.id=apollo-quickstart -Denv=DEV -Ddev_meta=http://localhost:8080
   	public static void main(String[] args) throws InterruptedException {
   		Config config = ConfigService.getAppConfig();
   		String someKey = "timeout";

   		while (true) {
   			String value = config.getProperty(someKey, null);
   			System.out.printf("now: %s, timeout: %s%n", LocalDateTime.now().toString(), value);
   			Thread.sleep(3000L);
   		}
   	}
   }
```

![image-20190918112613550](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918112613550.png)

1. 切换到`配置`Tab，再次检查灰度的配置部分，如果没有问题，点击`灰度发布`

![image-20190918090818119](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918090818119.png)

1. 在弹出框中可以看到主版本的值是2000，灰度版本即将发布的值是3000。填入其它信息后，点击发布

![image-20190918091132242](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918091132242.png)

1. 发布后，切换到`灰度实例列表`Tab，就能看到172.16.0.160已经使用了灰度发布的值

![image-20190918112718129](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918112718129.png)

![image-20190918112829636](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918112829636.png)

#### 5.5.8 全量发布

如果灰度的配置测试下来比较理想，符合预期，那么就可以操作`全量发布`。

全量发布的效果是：

1. 灰度版本的配置会合并回主版本，在这个例子中，就是主版本的timeout会被更新成3000
2. 主版本的配置会自动进行一次发布
3. 在全量发布页面，可以选择是否保留当前灰度版本，默认为不保留。

![image-20190918112958555](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918112958555.png)

![image-20190918113019626](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918113019626.png)

![image-20190918113037104](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918113037104.png)

#### 5.5.9 放弃灰度

如果灰度版本不理想或者不需要了，可以点击`放弃灰度`

![image-20190918103327131](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918103327131.png)

#### 5.5.10 发布历史

点击主版本的`发布历史`按钮，可以看到当前namespace的主版本以及灰度版本的发布历史

![image-20190918103427822](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918103427822.png)

#### See also