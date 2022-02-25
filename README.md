``1.zuul学习
今日内容
================

- **能够说出熔断器的作用**
- **能够使用 Zuul 配置后台微服务网关**
- **能够使用 Zuul 配置前台微服务网关**
- **能够使用 SpringCloudConfig 统一管理微服务配置**
- **能够使用 SpringCloudBus 实现微服务配置实时更新**

1. 熔断器 Hystrix
=================

1.1. 雪崩效应
-----------------------

在微服务架构中通常会有多个服务层调用，基础服务的故障可能会导致级联故障，进而造成整个系统不可用的情况，这种现象被称为服务**雪崩效应**。服务雪崩效应是一种因“服务提供者”的不可用导致“服务消费者”的不可用,并将不可用逐渐放大的过程。

-   如果下图所示：A 作为服务提供者，B 为 A 的服务消费者，C 和 D 是 B
    的服务消费者。A

不可用引起了 B 的不可用，并将不可用像滚雪球一样放大到 C 和 D时，雪崩效应就形成了。

![](media/1.png)

如何避免产生这种雪崩效应呢？我们可以使用 Hystrix 来实现熔断器。

### 1.1.1 雪崩效应的常见场景

- 硬件故障：如服务器宕机，机房断电，光纤被挖断等
  - 多机房容灾，异地多活
- 流量激增：如异常流量，重试加大流量等
  - 服务扩容，流量控制，限流、关闭等
- 缓存穿透：一般发生在重启应用，所有缓存失效时，以及短时间内大量缓存失效时。大量的缓存不命中，使请求直接全部来到后端服务，造成服务的超负载运行，引起服务不可用
  - 缓存预热，缓存异步加载
- 程序BUG：如程序逻辑导致内存泄漏
  - 修改程序bug，及时释放资源。

1.2. 什么是 Hystrix
-------------------

Hystrix [hɪst'rɪks]的中文含义是豪猪, 因其背上长满了刺,而拥有自我保护能力。

![](media/2.png)

Hystrix 能使你的系统在出现依赖服务失效的时候，通过隔离系统所依赖的服务，防止服务级联失败，同时提供失败回退机制，更优雅地应对失效，并使你的系统能更快地从异常中恢复。了解熔断器模式请看下图：

![](media/3.png)

1.3. 快速体验
-------------

Feign 本身支持 Hystrix，不需要额外引入依赖。

（1）修改 ktc\_qa 模块的 application.yml ，开启 hystrix

```yaml
feign:   # 开启熔断器
  hystrix:
    enabled: true
```



（2）在 com.ktc.qa.client 包下创建 impl包，包下创建熔断实现类，实现自接口 LabelClient

```java
package com.ktc.qa.client.impl;

import com.ktc.qa.client.LabelClient;
import entity.Result;
import entity.StatusCode;
import org.springframework.stereotype.Component;

@Component              //把该组件注入到spring容器中
public class LabelClientImpl implements LabelClient {
    @Override
    public Result findById(String id) {

        return new Result(false, StatusCode.ERROR,"熔断器生效啦.....");
    }
}

```



（3）修改 LabelClient 的注解

```java
package com.ktc.qa.client;

import com.ktc.qa.client.impl.LabelClientImpl;
import entity.Result;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@FeignClient(value = "ktc-base",fallback = LabelClientImpl.class)//调用微服的名称
public interface LabelClient {

    @RequestMapping(value = "/label/{id}", method = RequestMethod.GET)
    public Result findById(@PathVariable("id") String id);

}

```

（4）测试运行

重新启动问答微服务，测试看熔断器是否运行 。

2.微服务网关 Zuul
=================

2.1. 为什么需要微服务网关
-------------------------

不同的微服务一般有不同的网络地址，而外部的客户端可能需要调用多个服务的接口才能完成一个业务需求。比如一个电影购票的收集APP，可能会调用电影分类微服务，用户微服务，支付微服务等。如果客户端直接和微服务进行通信，会存在以下问题：

-   客户端会多次请求不同微服务，增加客户端的复杂性

-   存在跨域请求，在一定场景下处理相对复杂

-   认证复杂，每一个服务都需要独立认证

-   难以重构，随着项目的迭代，可能需要重新划分微服务，如果客户端直接和微服务通信，那么重构会难以实施


>   上述问题，都可以借助微服务网关解决。微服务网关是介于客户端和服务器端之间的中间层，所有的外部请求都会先经过微服务网关。

2.2. 什么是 Zuul
----------------

Zuul 是 Netflix 开源的微服务网关，他可以和 Eureka,Ribbon,Hystrix等组件配合使用。Zuul组件的核心是一系列的过滤器，这些过滤器可以完成以下功能：

-   身份认证和安全: 识别每一个资源的验证要求，并拒绝那些不符的请求

审查与监控：

-   动态路由：动态将请求路由到不同后端集群  

-   压力测试：逐渐增加指向集群的流量，以了解性能

-   负载分配：为每一种负载类型分配对应容量，并弃用超出限定值的请求

-   静态响应处理：边缘位置进行响应，避免转发到内部集群

-   多区域弹性：跨域 AWS Region 进行请求路由，旨在实现 ELB(ElasticLoadBalancing)使用多样化

Spring Cloud 对 Zuul 进行了整合和增强。

使用 Zuul 后，架构图演变为以下形式：

![](media/4.png)

2.3. Zuul 路由转发
------------------

### 2.3.1. 管理后台微服务网关

（1）创建子模块 ktc\_manager，pom.xml 引入 eureka-client 和 zuul的依赖

```xml
<!--Eureka客户端依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!--微服务网关Zuul依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```



（2）创建 application.yml

```yaml
server:
  port: 9011
spring:
  application:
    name: ktc-manager
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10101/eureka
  instance:
    prefer-ip-address: true #使用IP注册而不是主机名(注册多个微服时用IP区分)
zuul:
  routes:
    ktc-base:
      path: /base/**
      serviceId: ktc-base
    ktc-qa:
      path: /qa/**
      serviceId: ktc-qa
    ktc-user:               #路由Key
      path: /user/**        #配置请求 URL 的请求规则
      serviceId: ktc-user   #指定 Eureka 注册中心中的服务 id

```



（3）编写启动类

```java
package com.ktc.manager;

import org.apache.catalina.Manager;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@SpringBootApplication
@EnableZuulProxy                //开启网关代理
@EnableEurekaClient			//eureka客户端
public class ManagerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ManagerApplication.class);
    }

} 
```



### 2.3.2. 网站前台的微服务网关

（1）创建子模块 ktc\_web，pom.xml 引入依赖 zuul

```xml
<!--Eureka客户端依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!--微服务网关Zuul依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```



（2）创建 application.yml

```yaml
server:
  port: 9012
spring:
  application:
    name: ktc-web
eureka:
  client:
    service-url:  # eureka的注册地址
      defaultZone: http://127.0.0.1:10101/eureka
  instance:
    prefer-ip-address: true
zuul:
  routes:       #配置网关路由转发
    ktc-base:   #基础微服
      path: /base/**        #配置请求 URL 的请求规则
      serviceId: ktc-base   #注册到eureka中的微服名称
    ktc-qa:
      path: /qa/**
      serviceId: ktc-qa
    ktc-user:
      path: /user/**
      serviceId: ktc-user
```



（3）编写启动类

```java
package com.ktc.web;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@SpringBootApplication
@EnableZuulProxy            //开启网关代理
@EnableEurekaClient			//eureka客户端
public class WebApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebApplication.class);
    }
}

```



2.4. Zuul 过滤器
----------------

### 2.4.1. Zuul 过滤器快速体验

我们现在在 ktc_web 创建一个简单的 zuul 过滤器

```java
package com.ktc.web;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.exception.ZuulException;
import org.springframework.stereotype.Component;

@Component
public class WebFilter extends ZuulFilter {

    /**
     * 执行时机
     * pre:     在进入微服网关之间执行
     * route:   在执行微服务网关时执行
     * post:    在执行微服务网关之后执行
     * error:   在执行微服务出错执行
     * @return
     */
    @Override
    public String filterType() {
        return "pre";
    }

    /**
     * 执行顺序
     * 数字越大,优先级越低
     * @return
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * 是否执行该过滤器
     * true:执行
     * false:不执行
     * @return
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * 过滤器执行逻辑
     * @return 返回null代表放行,访问对应的微服务
     * @throws ZuulException
     */
    @Override
    public Object run() throws ZuulException {
        System.out.println("Zuul过滤器生效啦！");
        return null;
    }
}

```



启动 ktc\_web 会发现过滤器已经执行

filterType：返回一个字符串代表过滤器的类型，在 zuul中定义了四种不同生命周期的过滤器类型，具体如下：

- pre：在请求路由之前被调用
- route：在路由请求时候被调用
- post：在route和error过滤器之后被调用
- error：处理请求时发生错误时被调用
- filterOrder：通过int值来定义过滤器的执行顺序
- shouldFilter：返回一个boolean类型来判断该过滤器是否要执行，true：执行，false：不执行

### 2.4.2. 管理后台过滤器实现 Token 校验

修改 ktc\_manager 的过滤器, 因为是管理后台使用，所以需要在过滤器中对token 进行验证。

（1）ktc\_manager 引入 ktc_common 依赖 ，因为需要用到其中的 JWT工具类

```xml
<dependency>
    <groupId>com.ktc</groupId>
    <artifactId>ktc_common</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

（2）修改 ktc\_manager 配置文件 application.yml

```yml
server:
  port: 9011
spring:
  application:
    name: ktc-manager
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10101/eureka
  instance:
    prefer-ip-address: true #使用IP注册而不是主机名(注册多个微服时用IP区分)
zuul:
  routes:
    ktc-article:
      path: /article/**
      serviceId: ktc-article
    ktc-gathering:
      path: /gathering/**
      serviceId: ktc-gathering
    ktc-recruit:               #用户微服
      path: /recruit/**        #配置请求 URL 的请求规则
      serviceId: ktc-recruit   #指定 Eureka 注册中心中的服务 id
    ktc-user:
      path: /user/**        #配置请求 URL 的请求规则
      serviceId: ktc-user   #指定 Eureka 注册中心中的服务 id
jwt:
  config:
    key: dfbz_      #token秘钥
    prefix: dfbz_

```

（3）修改 ktc\_manager 的启动类,添加 bean

```java
@Bean
public JwtUtil jwtUtil(){
    return new JwtUtil();
}
```

（4）ktc\_manager 编写过滤器类

```java
package com.ktc.manager;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;
import io.jsonwebtoken.Claims;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;
import util.JwtUtil;

import javax.servlet.http.HttpServletRequest;

@Component
public class ManagerFilter extends ZuulFilter {

    @Autowired
    private Environment env;

    @Autowired
    private JwtUtil jwtUtil;

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {

        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();

        // 获取请求头数据
        String auth = request.getHeader("auth");

        String prefix = env.getProperty("jwt.config.prefix");

        // 设置响应数据的格式
        context.getResponse().setContentType("text/html;charset=utf8");

        if (auth == null || !auth.startsWith(prefix)) {
            //不是管理员
            //中止请求
            context.setSendZuulResponse(false);

            //设置正确响应内容
            context.setResponseBody("无权访问");

            return null;
        }


        //获取token字符串
        String token = auth.substring(prefix.length());

        Claims claims = jwtUtil.parseJWT(token);

        if (!claims.get("role").toString().equals("admin")) {
            //中止请求
            context.setSendZuulResponse(false);

            //设置正确响应内容
            context.setResponseBody("你不是管理员哦");
        }

        return null;
    }
}
```

测试访问查询文章列表：

URL：http://localhost:9011/article/article

method：GET

3.集中配置组件 SpringCloudConfig 
=================================

3.1. SpringCloudConfig 简介
---------------------------

在分布式系统中，由于服务数量巨多，为了方便服务配置文件**统一管理**，**实时更新**，所以需要分布式配置中心组件。在Spring Cloud 中，有分布式配置中心组件 spring cloud config，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程 Git仓库中。在 spring cloud config 组件中，分两个角色，

一是 config server，

二是config client。

Config Server是一个可横向扩展、集中式的配置服务器，它用于集中管理应用程序各个环境下的配置，默认使用 Git 存储配置文件内容，也可以使用 SVN存储，或者是本地文件存储。Config Client 是 Config Server 的客户端，用于操作存储在 Config Server中的配置内容。微服务在启动时会请求 Config Server获取配置文件的内容，请求到后再启动容器。详细内容看在线文档：https://springcloud.cc/spring-cloud-config.html

3.2. 配置服务端
---------------

### 3.2.1. 将配置文件提交到码云

使用 GitHub时，国内的用户经常遇到的问题是访问速度太慢，有时候还会出现无法连接的情况。如果我们希望体验Git 飞一般的速度，可以使用国内的 Git 托管服务 ——码云（gitee.com）。

-   GitHub 相比，码云也提供免费的 Git
    仓库。此外，还集成了代码质量检测、项目演示等功能。对于团队协作开发，码云还提供了项目管理、代码托管、文档管理的服务。

步骤：

（1）浏览器打开 gitee.com，注册用户 ，注册后登陆码云管理控制台

![](media/5.png)

（2）创建项目 ktc-config (点击右上角的加号 ，下拉菜单选择创建项目)

（3）上传配置文件，将 ktc\_base 工程的 application.yml 改名为base-dev.yml 后

上传

![图片1](media/6.png)



可以通过拖拽的方式将文件上传上去

![](media/7.png)

上传成功后列表可见

![](media/8.png)

可以再次编辑此文件

![](media/9.png)

文件命名规则：{application}-{profile}.yml 或{application}-{profile}.properties application 为应用名称 profile 指的开发环境（用于区分开发环境，测试环境、生产环境等）

（4）复制 git 地址 ,备用

![](media/10.png)

地址为：https://gitee.com/gz_dfbz/KTC.git

### 3.2.2. 配置中心微服务

（1）创建工程模块 配置中心微服务 ktc_config ,pom.xml 引入依赖

```xml
<!--SpringCloudConfig依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```



（2）创建启动类 ConfigServerApplication

```java
package com.ktc.config;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@EnableConfigServer         //开启SpringCloudConfig集中配置服务器
@SpringBootApplication
public class ConfigApplication {

    public static void main(String[] args) {

        SpringApplication.run(ConfigApplication.class);

    }
}

```



（3）编写配置文件 application.yml

```yml
server:
  port: 10102
spring:
  application:
    name: ktc-config
  cloud:
    config:
      server:
        git:
          uri: 你的码云项目链接地址
```



(4）浏览器测试：http://localhost:10102/base-dev.yml 可以看到配置内容

3.3. 配置客户端
---------------

（1）在 ktc\_base 工程添加依赖

```xml
<!--config集中配置客户端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```



（2）添加 bootstrap.yml ,删除 application.yml

```yaml
spring:
  cloud:
    config:
      uri: http://127.0.0.1:10102  # 配置中心服务端地址
      name: base      # 配置文件的应用名称
      profile: dev    # 配置文件的环境名称
      label: master   # 在远程仓库的分支名称
```



（3）测试： 启动工程 ktc\_eureka ktc\_config ktc\_base，看是否可以正常运行

http://localhost:9001/label

4.消息总线 SpringCloudBus
==========================

4.1. SpringCloudBus 简介
------------------------

如果我们更新码云中的配置文件，那客户端工程是否可以及时接受新的配置信息呢？我们现在来做有一个测试，修改一下码云中的配置文件中 mysql 的端口，然后测试http://localhost:9001/label数据依然可以查询出来，证明修改服务器中的配置并没有更新立刻到工程，只有重新启动程序才会读取配置。那我们如果想在不重启微服务的情况下更新配置如何来实现呢? 我们使用SpringCloudBus 来实现配置的自动更新。

### 4.1.1 Bus总线刷新流程

![](media/bus.png)

4.2. 代码实现
-------------

### 4.2.1. 配置服务端

（1）修改 ktc\_config 工程的 pom.xml，引用依赖

```xml
<!--消息总线bus-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
</dependency>

<!--rabbitmq-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

（2）修改 application.yml ，添加配置

```yaml
server:
  port: 10102
spring:
  application:
    name: ktc-config
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/gz_dfbz/KTC.git
  rabbitmq:
    host: 192.168.12.137
management: #暴露触发消息总线的地址
  endpoints:
    web:
      exposure:
        include: bus-refresh
```



### 4.2.2. 配置客户端

我们还是以基础模块为例，加入消息总线

（1）修改 ktc\_base 工程 ，引入依赖

```xml
<!--消息总线bus依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```



（2）在码云的配置文件中添加配置 rabbitMQ 的地址：

```yaml
rabbitmq:
	host: 192.168.12.133
```



（3）启动 ktc\_eureka 、ktc\_config 和 ktc\_base
看是否正常运行

（4）修改码云上的配置文件 ，将数据库连接 IP 改为 127.0.0.1，在本地部署一份数据库。

（5）postman 测试 Url: http://127.0.0.1:10102/actuator/bus-refresh Method:post

（6）再次观察输出的数据是否是读取了本地的 mysql 数据

### 4.2.3. 自定义配置的读取

（1）修改码云上的配置文件，增加自定义配置

```yaml
dfbz:
	config: 
		info: 测试bus总线刷新
```



（2）在 ktc\_base 工程中新建 controller

```yaml
@Value("${dfbz.config.info}")
private String info;

@RequestMapping(value = "/testBus", method = RequestMethod.GET)
public Result testBus() {

    return new Result(true, StatusCode.OK, "查询成功", info);
}
```

（3）运行测试看是否能够读取配置信息 ，OK

（4）修改码云上的配置文件中的自定义配置

```yaml
dfbz:
	config: 
		info: 测试bus总线刷新-----2
```

（5）通 过 postman 测 试 Url: http://127.0.0.1:12000/actuator/bus-refresh

Method:post

测试后观察,发现并没有更新信息。

这是因为我们的 controller 少了一个注解@RefreshScope 此注解用于刷新配置

```java
@RestController
@RequestMapping("/label")
@CrossOrigin
@RefreshScope // 刷新自定义配置
public class LabelController {
```



添加后再次进行测试 。

### 4.2.4. 完成KTC工程的配置集中管理

（1）将每一个工程的配置文件提取出来，重命名

（2）将这些文件上传到码云

（3）修改每一个微服务工程，pom.xml 中添加依赖

```xml
<!--config集中配置客户端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>

<!--消息总线bus依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
</dependency>

<!--rabbitmq依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>

<!--实时更新-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```



（4）删除每一个微服务的 application.yml

（5）为每一个微服务添加 bootstrap.yml （参考 ktc_base 工程）

（6）修改码云上的配置文件添加 rabbitmq 地址

```yaml
rabbitmq:
	host: 192.168.12.133
```
