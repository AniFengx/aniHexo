---
title: spring监控平台从零单排【一】
date: 2023-03-21 09:24:47
tags: [java,spring,monitor,Admin]
categories: [monitor]
---

公司之前已经有了简单的服务器监控工具，主要是针对linux服务器性能监控和mysql、redis数据库的监控，并没有做到对真实服务情况的监控，公司当前后端主要以java为主，所以缺少了jvm及接口性能的监控。这些可以通过java相关开源框架并结合nginx的日志来处理。

spring基本已经是java开发的主流框架，spring也提供了**Actuator**开源框架来收集应用的相关指标，如健康检查，审计，指标收集，并且支持**Prometheus**整合。

### Spring Boot Admin

**Actuator**提供各种端口来暴露收集的原始数据，并且支持自定义打点收集数据，但是这些并不能直观的展示到平台上，可以借助**Prometheus**传递到**grafana**上或自行接入，springboot也提供了一个开发好的ui监控平台展示**Actuator**的数据，这就是**Spring Boot Admin**。

<!-- more -->

下面简单介绍**Admin**的使用姿势

#### 集成

**Admin**需要一个单独的模块来运行监控平台，即server；已有的业务模块都会当做收集端，即client

##### server

1. 新建模块，直接在ide工具中new一个新模块即可，创建`application.java`文件和`application.yml`文件
2. 新模块pom文件中添加依赖

    ``` java
    <!--  支持admin-ui的关键配置  -->
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
        <version>2.5.5</version>
    </dependency>
    ```

    *我也没搞懂为啥admin的组织不在`springboot parent`全家桶里，还需要单独控制版本号*

    > 注意`spring-boot-admin-starter-server`的版本，和`spring-boot-starter-parent`保持一致，admin支持的版本号可以到maven官方仓库或一些[第三方maven仓库](https://central.sonatype.com/artifact/de.codecentric/spring-boot-admin-starter-server/3.0.2/versions "收集Maven版本号的网站")查看

3. 新模块`application.java`文件中添加注解`@EnableAdminServer`

    ``` java
    @Configuration
    @EnableAutoConfiguration
    @EnableAdminServer
    public class App {
        public static void main(String[] args){
            SpringApplication.run(App.class,args);
        }
    }
    ```

4. 运行新模块查看效果

![avatar](1.png)

##### client

1. 修改需要监控的业务模块的pom文件

    ``` java
        <!--  支持admin-ui的关键配置  -->
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-client</artifactId>
            <version>2.5.5</version>
        </dependency>
    ```

2. 修改`application.yml`文件，添加server的信息

    ``` yaml
    spring:
        #注册当前服务到admin-ui-server服务中
        boot:
            admin:
            client:
                url: http://localhost:8090
    management:
        endpoints:
            web:
            exposure:
                #暴露所有的收集信息给server
                include: "*"
        endpoint:
            health:
                #显示详细健康信息，例如数据库mysql和redis
                show-details: always
    ```

    > 尤其注意`spring.boot.admin.client.url`的配置，如果`server`配置了`server.servlet.context-path`，要注意url的地址后也要做相应的拼接

3. 启动业务模块，会自动请求`server`进行注册，`server`不需要重启，并且会自动轮询业务模块暴露的收集接口抓取信息

![avatar](2.png)

至此`Admin`就已经可以初步使用了

### 后记

接入过程中遇到了两个坑，分别记录下

#### client注册到server报错

报错信息如下

``` java
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct instance of `de.codecentric.boot.admin.server.domain.values.Registration` (no Creators, like default constructor, exist): cannot deserialize from Object value (no delegate- or property-based Creator)
```

原因是`application.java`启动类中没有正确添加注解，如果使用了`@SpringBootApplication`，会因为`@ComponentScan`的特性以当前类所在包来查找bean，导致不能正确注册`Admin`相关的组件

{% note warning %} 直接baidu的解决方法对代码侵入性过高，需要重写spring容器中注册的序列化方式类，最后查看`Admin`官网发现了华点 {% endnote %}

#### 在client中获取容器当前所有注册路径报错

当使用如下代码时，会提示有重复类型的组件在容器中

``` java
RequestMappingHandlerMapping mapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
```

报错信息为

``` java
Method xssFilterRegistration in com.decent.WebApplication required a single bean, but 2 were found:
	- requestMappingHandlerMapping: defined by method 'requestMappingHandlerMapping' in class path resource [org/springframework/web/servlet/config/annotation/DelegatingWebMvcConfiguration.class]
	- controllerEndpointHandlerMapping: defined by method 'controllerEndpointHandlerMapping' in class path resource [org/springframework/boot/actuate/autoconfigure/endpoint/web/servlet/WebMvcEndpointManagementContextConfiguration.class]
```

`Admin`也会注册一个`RequestMappingHandlerMapping`的子类`ControllerEndpointHandlerMapping`到容器中，需要使用根据bean名称来获取的bean的形式

``` java
RequestMappingHandlerMapping mapping = applicationContext.getBean("requestMappingHandlerMapping", RequestMappingHandlerMapping.class);
```