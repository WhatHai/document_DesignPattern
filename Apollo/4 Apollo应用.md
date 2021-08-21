## 4 Apollo应用

### 4.1 Apollo工作原理

下图是Apollo架构模块的概览

![overall-architecture](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/overall-architecture.png)

#### 4.1.1 各模块职责

上图简要描述了Apollo的总体设计，我们可以从下往上看：

- Config Service提供配置的读取、推送等功能，服务对象是Apollo客户端
- Admin Service提供配置的修改、发布等功能，服务对象是Apollo Portal（管理界面）
- Eureka提供服务注册和发现，为了简单起见，目前Eureka在部署时和Config Service是在一个JVM进程中的
- Config Service和Admin Service都是多实例、无状态部署，所以需要将自己注册到Eureka中并保持心跳
- 在Eureka之上架了一层Meta Server用于封装Eureka的服务发现接口
- Client通过域名访问Meta Server获取Config Service服务列表（IP+Port），而后直接通过IP+Port访问服务，同时在Client侧会做load balance、错误重试
- Portal通过域名访问Meta Server获取Admin Service服务列表（IP+Port），而后直接通过IP+Port访问服务，同时在Portal侧会做load balance、错误重试
- 为了简化部署，我们实际上会把Config Service、Eureka和Meta Server三个逻辑角色部署在同一个JVM进程中

#### 4.1.2 分步执行流程

1. Apollo启动后，Config/Admin Service会自动注册到Eureka服务注册中心，并定期发送保活心跳。
2. Apollo Client和Portal管理端通过配置的Meta Server的域名地址经由Software Load Balancer(软件负载均衡器)进行负载均衡后分配到某一个Meta Server
3. Meta Server从Eureka获取Config Service和Admin Service的服务信息，相当于是一个Eureka Client
4. Meta Server获取Config Service和Admin Service（IP+Port）失败后会进行重试
5. 获取到正确的Config Service和Admin Service的服务信息后，Apollo Client通过Config Service为应用提供配置获取、实时更新等功能；Apollo Portal管理端通过Admin Service提供配置新增、修改、发布等功能

### 4.2 核心概念

1. **application (应用)**

这个很好理解，就是实际使用配置的应用，Apollo客户端在运行时需要知道当前应用是谁，从而可以去获取对应的配置

**关键字：appId**

1. **environment (环境)**

配置对应的环境，Apollo客户端在运行时需要知道当前应用处于哪个环境，从而可以去获取应用的配置

**关键字：env**

1. **cluster (集群)**

一个应用下不同实例的分组，比如典型的可以**按照数据中心分**，把上海机房的应用实例分为一个集群，把北京机房的应用实例分为另一个集群。

**关键字：cluster**

1. **namespace (命名空间)**

一个应用下不同配置的分组，可以简单地把namespace类比为文件，不同类型的配置存放在不同的文件中，如数据库配置文件，RPC配置文件，应用自身的配置文件等

**关键字：namespaces**

它们的关系如下图所示：

![image-20190919170234623](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190919170234623.png)

### 4.3 项目管理

#### 4.3.1 基础设置

1. 部门管理

apollo 默认部门有两个。要增加自己的部门，可在系统参数中修改：

- 进入系统参数设置

![image-20190716115120642](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716115120642.png)

- 输入key查询已存在的部门设置：organizations

  ![image-20190716115230543](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716115230543.png)

- 修改value值来添加新部门，下面添加一个微服务部门：

  ```json
   [{"orgId":"TEST1","orgName":"样例部门1"},{"orgId":"TEST2","orgName":"样例部门2"},{"orgId":"micro_service","orgName":"微服务部门"}]
  ```

1. 添加用户

apollo默认提供一个超级管理员: apollo，可以自行添加用户

- 新建用户张三

![image-20190720181559923](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190720181559923.png)

![image-20190720181647695](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190720181647695.png)

#### 4.3.2 创建项目

1. 打开apollo-portal主页：<http://localhost:8070/>
2. 点击“创建项目”：account-service

![image-20190716101236392](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716101236392.png)

1. 输入项目信息
   - 部门：选择应用所在的部门
   - 应用AppId：用来标识应用身份的唯一id，格式为string，需要和项目配置文件applications.properties中配置的app.id对应
   - 应用名称：应用名，仅用于界面展示
   - 应用负责人：选择的人默认会成为该项目的管理员，具备项目权限管理、集群创建、Namespace创建等权限

![image-20190716101502009](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716101502009.png)

1. 点击提交

创建成功后，会自动跳转到项目首页

![image-20190716101909047](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716101909047.png)

1. 赋予之前添加的用户张三管理account-service服务的权限

   - 使用管理员apollo将指定项目授权给用户张三

   ![image-20190720182520071](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190720182520071.png)

   - 将修改和发布权限都授权给张三

   ![image-20190720182858984](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190720182858984.png)

   - 使用zhangsan登录，查看项目配置

   ![image-20190720183023342](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190720183023342.png)

   - 点击account-service即可管理配置

#### 4.3.3 删除项目

如果要删除整个项目，点击右上角的“管理员工具–》删除应用、集群…”

首先查询出要删除的项目，点击“删除应用”

![1570248995936](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/1570248995936.png)

### 4.4 配置管理

下边在account-service项目中进行配置。

#### 4.4.1 添加发布配置项

1. 通过表格模式添加配置

   - 点击新增配置

   ![image-20190917113735222](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917113735222.png)

   - 输入配置项：sms.enable，点击提交

   ![image-20190917113829772](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917113829772.png)

2. 通过文本模式编辑

Apollo除了支持表格模式，逐个添加、修改配置外，还提供文本模式批量添加、修改。 这个对于从已有的properties文件迁移尤其有用

- 切换到文本编辑模式

  ![image-20190917113953166](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917113953166.png)

- 输入配置项，并点击提交修改

  ![image-20190917114051248](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917114051248.png)

1. 发布配置

![image-20190917114118871](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917114118871.png)

#### 4.4.2 修改配置

1. 找到对应的配置项，点击修改

![image-20190917114952136](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917114952136.png)

1. 修改为需要的值，点击提交
2. 发布配置

#### 4.4.3 删除配置

1. 找到需要删除的配置项，点击删除

![image-20190917140811049](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917140811049.png)

1. 确认删除后，点击发布

![image-20190917140920348](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917140920348.png)

#### 4.4.4 添加Namespace

Namespace作为配置的分类，可当成一个配置文件。

以添加rocketmq配置为例，添加“spring-rocketmq” Namespace配置rocketmq相关信息。

1. 添加项目私有Namespace：spring-rocketmq

进入项目首页，点击左下脚的“添加Namespace”，共包括两项：关联公共Namespace和创建Namespace，这里选择“创建Namespace”

![image-20190716135746538](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716135746538.png)

1. 添加配置项

```properties
   rocketmq.name-server = 127.0.0.1:9876
   rocketmq.producer.group = PID_ACCOUNT
```

![image-20190716141257636](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716141257636.png)

1. 发布配置

![image-20190716141330583](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716141330583.png)

#### 4.4.5 公共配置

##### 4.4.5.1 添加公共Namespace

在项目开发中，有一些配置可能是通用的，我们可以通过把这些通用的配置放到公共的Namespace中，这样其他项目要使用时可以直接添加需要的Namespace

1. 新建common-template项目

![image-20190716104213113](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716104213113.png)

1. 添加公共Namespace：spring-boot-http

进入common-template项目管理页面：<http://localhost:8070/config.html?#/appid=common-template>

![img](http://www.pbteach.com/image-20190716104443976.png)

![image-20190716104856041](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716104856041.png)

1. 添加配置项并发布

```properties
   spring.http.encoding.enabled = true
   spring.http.encoding.charset = UTF-8
   spring.http.encoding.force = true
   server.tomcat.remote_ip_header = x-forwarded-for
   server.tomcat.protocol_header = x-forwarded-proto
   server.use-forward-headers = true
   server.servlet.context-path = /
```

![image-20190917141904861](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917141904861.png)

##### 4.4.5.2 关联公共Namespace

1. 打开之前创建的account-service项目
2. 点击左侧的添加Namespace

![img](http://www.pbteach.com/image-20190716111321555.png)

1. 添加Namespace

![image-20190716111600622](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716111600622.png)

1. 根据需求可以覆盖引入公共Namespace中的配置，下面以覆盖server.servlet.context-path为例

![image-20190716111914846](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716111914846.png)

1. 修改server.servlet.context-path为：/account-service

![image-20190716112948132](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716112948132.png)

1. 发布修改的配置项

![image-20190716113330392](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716113330392.png)

### 4.5 多项目配置

通常一个分布式系统包括多个项目，所以需要多个项目，下边以一个P2P金融的项目为例，添加交易中心微服务transaction-service。

1. 添加交易中心微服务transaction-service

![image-20190917171820325](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917171820325.png)

1. 关联公共Namespace

任务应用都可以关联公共Namespace。

![image-20190917171916225](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917171916225.png)

1. 覆盖配置，修改交易中心微服务的context-path为：/transaction

![image-20190917172050995](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190917172050995.png)

1. 发布修改后的配置



### 4.6 集群管理

在有些情况下，应用有需求对不同的集群做不同的配置，比如部署在A机房的应用连接的RocketMQ服务器地址和部署在B机房的应用连接的RocketMQ服务器地址不一样。另外在项目开发过程中，也可为不同的开发人员创建不同的集群来满足开发人员的自定义配置。

#### 4.6.1 创建集群

1. 点击页面左侧的“添加集群”按钮

![image-20190716142112080](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716142112080.png)

1. 输入集群名称SHAJQ，选择环境并提交：添加上海金桥数据中心为例

![image-20190716142454845](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716142454845.png)

1. 切换到对应的集群，修改配置并发布即可

![image-20190716142906278](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716142906278.png)

#### 4.6.2 同步集群配置

同步集群的配置是指在同一个应用中拷贝某个环境下的集群的配置到目标环境下的目标集群。

1. 从其他集群同步已有配置到新集群

   - 切换到原有集群
   - 展开要同步的Namespace，点击同步配置

   ![image-20190716143608936](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716143608936.png)

   - 选择同步到的新集群，再选择要同步的配置

   ![image-20190716143755503](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716143755503.png)

   - 同步完成后，切换到SHAJQ集群，发布配置

   ![image-20190716143933267](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190716143933267.png)

#### 4.6.3 读取配置

读取某个集群的配置，需要启动应用时指定具体的应用、环境和集群。

-Dapp.id=应用名称

-Denv=环境名称

-Dapollo.cluster=集群名称

-D环境_meta=meta地址

```
-Dapp.id=account-service -Denv=DEV -Dapollo.cluster=SHAJQ -Ddev_meta=http://localhost:8080 
```

### 4.7 配置发布原理

在配置中心中，一个重要的功能就是配置发布后实时推送到客户端。下面我们简要看一下这块是怎么设计实现的。

![release-message-notification-design-8613796](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/release-message-notification-design-8613796.png)

上图简要描述了配置发布的主要过程：

1. 用户在Portal操作配置发布
2. Portal调用Admin Service的接口操作发布
3. Admin Service发布配置后，发送ReleaseMessage给各个Config Service
4. Config Service收到ReleaseMessage后，通知对应的客户端

#### 4.7.1 发送ReleaseMessage

Admin Service在配置发布后，需要通知所有的Config Service有配置发布，从而Config Service可以通知对应的客户端来拉取最新的配置。

从概念上来看，这是一个典型的消息使用场景，Admin Service作为producer（生产者）发出消息，各个Config Service作为consumer（消费者）消费消息。通过一个消息队列组件（Message Queue）就能很好的实现Admin Service和Config Service的解耦。

在实现上，考虑到Apollo的实际使用场景，以及为了尽可能减少外部依赖，我们没有采用外部的消息中间件，而是通过数据库实现了一个简单的消息队列。

具体实现方式如下：

1. Admin Service在配置发布后会往ReleaseMessage表插入一条消息记录，消息内容就是配置发布的AppId+Cluster+Namespace

```sql
   SELECT * FROM ApolloConfigDB.ReleaseMessage
```

![image-20190918142607113](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918142607113.png)

消息发送类：[DatabaseMessageSende](https://github.com/ctripcorp/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/DatabaseMessageSender.java)

![image-20190918143135480](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918143135480.png)

1. Config Service有一个线程会每秒扫描一次ReleaseMessage表，看看是否有新的消息记录

消息扫描类：[ReleaseMessageScanner](https://github.com/ctripcorp/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/ReleaseMessageScanner.java)

![image-20190918143454666](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918143454666.png)

1. Config Service如果发现有新的消息记录，那么就会通知到所有的消息监听器

![image-20190918143832024](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918143832024.png)

然后调用消息监听类的handleMessage方法：[NotificationControllerV2](https://github.com/ctripcorp/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/controller/NotificationControllerV2.java)

![image-20190918144207638](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918144207638.png)

1. NotificationControllerV2得到配置发布的AppId+Cluster+Namespace后，会通知对应的客户端

![release-message-design](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/release-message-design.png)

#### 4.7.2 Config Service通知客户端

上一节中简要描述了NotificationControllerV2是如何得知有配置发布的，那NotificationControllerV2在得知有配置发布后是如何通知到客户端的呢？

实现方式如下：

1. 客户端会发起一个Http请求到Config Service的`notifications/v2`接口[NotificationControllerV2](https://github.com/ctripcorp/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/controller/NotificationControllerV2.java)

![image-20190918145100432](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918145100432.png)

客户端发送请求类：[RemoteConfigLongPollService](https://github.com/ctripcorp/apollo/blob/master/apollo-client/src/main/java/com/ctrip/framework/apollo/internals/RemoteConfigLongPollService.java)

![image-20190918150801655](http://www.pbteach.com/post/java_distribut/apollo_yq_doc/image-20190918150801655.png)

1. NotificationControllerV2不会立即返回结果，而是把请求挂起。考虑到会有数万客户端向服务端发起长连，因此在服务端使用了async servlet([Spring DeferredResult](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html))来服务Http Long Polling请求。
2. 如果在60秒内没有该客户端关心的配置发布，那么会返回Http状态码304给客户端。
3. 如果有该客户端关心的配置发布，NotificationControllerV2会调用DeferredResult的[setResult](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html#setResult-T-)方法，传入有配置变化的namespace信息，同时该请求会立即返回。客户端从返回的结果中获取到配置变化的namespace后，会立即请求Config Service获取该namespace的最新配置。

#### 4.7.3 客户端读取设计

除了之前介绍的客户端和服务端保持一个长连接，从而能第一时间获得配置更新的推送外，**客户端还会定时从Apollo配置中心服务端拉取应用的最新配置**。

- 这是一个备用机制，为了防止推送机制失效导致配置不更新
- 客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回304 - Not Modified
- 定时频率默认为每5分钟拉取一次，客户端也可以通过在运行时指定System Property: `apollo.refreshInterval`来覆盖，单位为分钟

#### 