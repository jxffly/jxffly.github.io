---
layout: post
title:  "spring-boot的启动"
date: 2017-11-23 22:21:49
tags: spring
---
# spring-boot的初相见

## 1.spring-boot的特点（https://github.com/jxffly/spring-boot-test）

本文是基于刘俊强老师在stuq学院的《spring-boot微服务》做的简化版的一个spring-boot介绍，把代码直接下载或者vcs系统导入项目，直接run  AccountServiceApplication的main方法，就可以启动，不需要配置容器，仅需要maven支持，声明一点现在spring不提供在线下载的方式下载spring的包，需要通过maven或者gradle的方式来下载依赖包

，代码是基于jdk1.8以及maven3.3.9

### 1.1 spring-boot提倡约定大于配置，但是支持个性化配置

spring-boot提供了很多开箱即用的配置，包括但不限于多环境发布的配置，datasource和redis配置，包括web容器的配置，极大的简化了服务的配置和发布，尤其是和微服务以及DDD(domai driven design)在一起更是风生水起，我们通过一些代码来一起看看这些新特性是怎么发挥作用的



### 1.2 单独启动

spring-boot提供了自己单独的启动器，可以不依赖于容器的调用，自己常驻线程，是适合做前后端分离的时候，作为后端api入口的单独服务存在，尤其适合多机器部署以及docker容器化大规模部署来，来经受较大的流量考验和高qps访问。而且适合自动伸缩的服务扩展。



### 1.3 build anything

spring-boot定义为build-anything，能够提供各种java生态的插件的集成环境配置，以及各种starter（启动器？脚手架？）

#### 1.4 运行监控

spring-boot提供了监控的启动器，包括但不限于，监控检查，最近的访问监控，bean的监控，jvm的信息的监控，和机器的硬件监控等等。

## 2.简单的示例

#### 2.0启动

请先把代码下载下来然后，AccountServiceApplication的main方法运行项目。感受一下spring-boot的运行方式

首先，spring支持默认开启全局配置，包括但不限于帮忙初始化各种配置生成的bean，比如我为什么在配置文件添加了datasource，数据库连接就已经建好，就是开启了全局话配置起的作用,会自动去加载引入的bean，这些都是spring-boot提供的装箱即用的功能，可以在引入对应的starter后，自动根据配置文件来初始化对应的bean。

#### 2.1配置文件

关于配置文件需要说一点，这里体现出了spring约定大于配置的一点，就是spring默认只加载application.yml以及application-**.yml（.properties .json等）里面的内容，其他的配置文件需要自己使用@PropertySource注解来自己引入。

对于由spring维护的属性会自动装载进入enviroment对象里面，可以自由的获取，相比其他的非自动加载的属性，自动加载的属性会被用来初始化自动装配的bean，这里比较饶舌，比如我引入了spring-jdbc-stater，这个时候，spring-jdbc的datatsource会自动被装载，如果我们在applicat.properties里面配置了数据源，那么datatsource会使用里面的属性来初始化自己，但是，如果我把配置文件卸载jdbc.properties，datasource不会使用配置文件里面的参数，这个时候你需要自己来初始化datasource，否则spring-boot会使用默认的参数初始化datatsource，而你的配置无法生效，切记！！！，其他的配置也一样，比如jedis的。

最最理想的情况，就是我的jedis、datasource、容器、消息中间件、amq、日志等等都可以走自动化配置，当然也可以自己创建自己的自动化配置属性，交由spirng-boot来托管启动。

以下是application.yml里面的内容

```yaml

server:
  port: 9999 #服务端口
  servlet:
    context-path: /account-service #项目的前缀，配置之后访问路径变化为 localhost:9999/account-service/any/any
#  display-name: /account-service
#  context-path: /account-service


logging:
  level:
    root: INFO #日志相关，全局的日志级别
    io.junq: DEBUG #io.liuq包下面的日志类型，root被覆盖了
  path: /data/logs/${spring.application.name} #日志的归档路径

spring:
  datasource: #这里是spirng-boot自动生成datasouce的地方，会自动发现classpath下面的数据库连接池类型，默认好像spring自带的连接池对象(datasource)，当然可以选择自己的比如c3p0或者druid等
    url: jdbc:mysql://localhost:3306/emall?characterEncoding=utf8&useSSL=true
    username: fly
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
  application:
    name: account-service

debug: false #是否开启框架调试，建议第一次查看springboot启动干了什么事情的时候可以打开

#可以开启对jvm的健康检查，等等
management:
  endpoint:
    metrics:
      enabled: true



```



然后 就是spring-boot约定最大的一点好处，完全不需要maven构建项目环境参数的打包系统，这些全部可以由spring-boot来做，比如下图就分别对应三套环境，dev，qa，test环境也就是application.(yml,properties,json)里面的参数是spring-boot一定会加载的配置项，appliction-xx配置文件下的配置项仅仅在当前环境激活的情况下才会被激活。比如，我在application.properties里面指定了参数 server-name=account-service不管在哪套环境都会加载这个配置项，如果在application-qa.yml里面配置了url=123.456.789,那么配置项只会在qa环境被激活的时候起效。

java -jar -Dspring.profiles.active=test account-service-1.0.jar （需要到jar包的目录，一般在target下面）比如这样来启动spring-boot项目，这个商户号只有test环境的配置项被激活，其他环境的不会激活，以此来激活相应的环境的参数。如果不指定任何环境，spring-boot会默认启动application-default.yml里面的环境，一般建议启动的时候配置默认的启动环境参数。

![spring-boot-profile](/image/spring-boot-properties.png)







#### 2.1.项目结构

![structure](/image/spring-boot-structure.png)

spring的maven里面引入了一系列的spring-boot-starter-**的启动器依赖，里面囊括了该组件需要的所有依赖，这些统一由spring来管理方便了jar的版本管理，使开发者从无尽的版本混乱当中抽离出来，并且引入的stater组件会在开启自动配置之后，会被spring-boot自动实例化。

但是如果一些必要的配置项不给定的时候，如果我们强行使用，会报缺少配置项的错误，下图是我加入jedis的stater，但是我没有加入reids的参数配置，那么jedispool会在使用的时候抛出异常，因为我没有为其配置配置参数

```
There was an unexpected error (type=Internal Server Error, status=500).
org.springframework.web.util.NestedServletException: Request processing failed; nested exception is org.springframework.data.redis.RedisConnectionFailureException: Cannot get Jedis connection; nested exception is redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource from the pool
```




```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.fly.test</groupId>
    <artifactId>account-service</artifactId>
    <version>1.0</version>
    <packaging>jar</packaging>

    <name>account-service</name>
    <description>account microservice for eMall</description>

    <parent>
        <groupId>com.fly.test</groupId>
        <artifactId>spring-boot-test</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <springfox.version>2.4.0</springfox.version>
        <joda-time.version>2.9.6</joda-time.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>joda-time</groupId>
            <artifactId>joda-time</artifactId>
            <version>${joda-time.version}</version>
        </dependency>
		 <dependency>
            <!--spring-boot的核心启动器，包括了负责吧配置文件组装为装箱即用主键的模块，其实其他的starter都依赖这个-->
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
              <!--这里使用jetty的嵌入式容器，需要排除默认的tomcat-->
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jetty</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.4.1</version>
        </dependency>

        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>1.3.0</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <!--开启各种监控-->
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-actuator</artifactId>
        </dependency>



        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>${springfox.version}</version>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>${springfox.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.commons/commons-lang3 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/com.google.guava/guava -->
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </dependency>


    </dependencies>

</project>

```



#### 2.2 主运行类

```java
package com.fly.boot.account;

import org.apache.commons.lang3.StringUtils;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.PropertySource;

import lombok.extern.slf4j.Slf4j;


/*@Configuration
@SpringBootApplication
@ComponentScan("io.junq.examples.emall")
*/

//这个是最重要的注解，包含了上面三个注解，负责标示这个类是负责启动容器的入口，由spring的线程来负责使服务常驻内存，并且会把引入的各种组件自动实力化
@SpringBootApplication(scanBasePackages = "com.fly.boot.account")
//自定义扫描额外的配置文件
@PropertySource(value = {"classpath:my.properties", "classpath:my2.properties"})
@Slf4j
public class AccountServiceApplication {

    private static String ENV = "spring.profiles.active";
	/**
	这里是一种不是很优雅的写法，因为周发布系统一般都是可用吧spring项目打成一个jar使用jar来运行，一般环境参数，诸如使用哪套环境配置来启动服务，这里在没有指定环境的情况下，默认启动qa的环境
	*/
    private static String DEFAULT_PROFILE = "qa";

	
    public static void main(String[] args) {

        //SpringApplication.run(AccountServiceApplication.class,args);
        SpringApplication springApplication = new 	  	   SpringApplication(AccountServiceApplication.class);
        String profile = System.getProperty(ENV);
        if (StringUtils.isEmpty(profile)) {
            log.warn("can not get any env setting,just use default:{}", DEFAULT_PROFILE);
            springApplication.setAdditionalProfiles(DEFAULT_PROFILE);
        }
        springApplication.run(args);
    }

}

```

我们会发现主运行类是一个main方法，这个是spring-boot服务的入口，和传统的sprng-mvc的项目不一样，spring传统的项目是容器负责来启动容器，然后再监听容器启动的事件来唤起spirng的整个框架，这里是直接运行spring容器，如果需要制作前后端一起的单独应用，可以使用thymleaf的模板引擎，但是不推荐使用spring-boot制作单独的前后单一起的应用，建议前后端分离，前端做单独的容器部署，中间使用nginx或者其他负债均衡插件做负债均衡，然后在经过网关到达底层服务，由多态机器集群来提供服务，这也是现在流行的微服务的一贯思想，蚂蚁吃大象，便于弹性伸缩。

SpringApplication实例来启动项目前，可以做很多自定义的操作，比如我们可以可以在springApplication.run(args)启动前，自定义banner的样式，添加额外的属性文件，自定义各种resource等等。

#### 2.3 api层

```java
package com.fly.boot.account.web;

import com.fly.boot.account.domain.User;
import com.fly.boot.account.service.AccountService;

import org.hibernate.validator.constraints.Length;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import java.util.Collections;
import java.util.Map;

import io.swagger.annotations.Api;
import io.swagger.annotations.ApiImplicitParam;
import io.swagger.annotations.ApiImplicitParams;
import io.swagger.annotations.ApiOperation;
import io.swagger.annotations.ApiParam;
import io.swagger.annotations.ApiResponse;
import io.swagger.annotations.ApiResponses;


/**
 * 用户账号API入口
 *
 * @author jinxiaofei
 */
@RestController("用户相关API")
@RequestMapping(value = "/v1", produces = {MediaType.APPLICATION_JSON_UTF8_VALUE})
@Api(value = "测试Swagger2",description="简单的API")
public class UserRestController {

    private final static Logger LOGGER = LoggerFactory.getLogger(UserRestController.class);

    @Autowired
    private AccountService accountService;



    @Autowired
    private TestDO testDO;


    @ApiOperation(value = "创建用户", notes = "")
    @ApiImplicitParams({})
    @ApiResponses({@ApiResponse(code = 200, message = "用户创建成功"),})
    @RequestMapping(method = RequestMethod.POST, value = "/users")
    public User createUser(@ApiParam(name = "user", value = "待创建的用户对象json实例", required = true) @RequestBody @Validated User user) {
        LOGGER.debug("try to create user by user: " + user);
        return accountService.createUser(user);
    }


    @ApiOperation(value = "查询用户", notes = "")
    @ApiImplicitParams({@ApiImplicitParam(name = "userId", paramType = "path", value = "用户id", required = true, dataType = "String")})
    @RequestMapping(method = RequestMethod.GET, value = "/users/{userId}")
    @Validated
    public User findUserByDisplayId(@PathVariable @Length(message = "length to short", min = 3) String userId) {
        LOGGER.debug("try to find user by display_id: " + userId);
        return accountService.findUserByDisplayId(userId);
    }


    @RequestMapping(value = "/hello")
    public Map<String, String> findProperties() {
        LOGGER.debug("try to find environment prop");
        return Collections.singletonMap("value", testDO.toString());
    }

}

```

这个是常见的api接口，里面有很多杂七杂八的自动的swagger自动api生成的代码，这个是常见的spring-mvc的api的接口定义，当我们启动上面的main方法的时候，然后访问下面连接就可以看到整个服务已经运行起来了

http://localhost:9999/account-service/v1/hello

可以看到下面的内容，这里面可能涉及到一个知识点，就是使用了spring-mvc的返回监控，使用了统一的响应体拦截，来构造统一的返回格式，把map变成了 如下图的返回body体,这个在生产环境当中很重要，这种拦截仅支持容器化的接口，其在requet的调用链上面进行拦截然后变更了返回的相应体，是一种代价很小的返回值格式化的做法，也是一般生产环境当中的做法

```json
{
"code": 10000,
"message": "成功",
"data": {
"value": "hello world"
}
}
```

我们主要关注一下，createUser的方法，里面使用了spring自带的参数校验框架，由spring使用hibernate-validator来负责参数验证，如果出现异常由统一的异常拦截来处理，这里是一般常见的处理方法，一般出现逻辑异常都是由异常来中断，这样做的好处在很长的调用链内部，可以立即中断当前流程，而不要一层层的吧错误信息通过错误码的判断，一级级传递上去。

```java
package com.fly.boot.account.web;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fly.boot.account.common.BaseResult;
import com.fly.boot.account.common.ErrorCode;
import com.fly.boot.account.domain.MyException;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

import java.util.List;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;


/**
 * @author jinxiaofei
 */
//这个是对于返回值的统一封装
@ControllerAdvice(basePackages = "com.fly.boot.account")
public class RestAPIExceptionHandler implements ResponseBodyAdvice<Object> {

    private static final Logger LOGGER = LoggerFactory.getLogger(RestAPIExceptionHandler.class);

    private static final ObjectMapper mapper = new ObjectMapper();


    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }


    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request,
            ServerHttpResponse response) {

        BaseResult<Object> baseResult = BaseResult.getBaseResult();

        if (body instanceof BaseResult) {
            return body;

        }
        baseResult.setData(body);
        return baseResult;
    }


    @ExceptionHandler(Exception.class)
    @ResponseBody
    //统一异常拦截
    public BaseResult exceptionHandler(HttpServletRequest request, HttpServletResponse response, Throwable t) {
        LOGGER.error("request occur an error,url:{},params:{}", request.getRequestURI(), request.getParameterMap(), t);
        return exceptionBuilder(t);

    }


    private BaseResult exceptionBuilder(Throwable t) {
        BaseResult<Object> baseResult = BaseResult.getBaseResult();
        if (t instanceof MyException) {
            MyException t1 = (MyException) t;
            baseResult.setCode(t1.getCode());
            baseResult.setMessage(t1.getMsg());
            return baseResult;
        }else if (t instanceof RuntimeException) {
            baseResult.setCode(ErrorCode.ERROR_INNER.getCode());
            baseResult.setMessage(t.getMessage());
            return baseResult;
        }else if (t instanceof MethodArgumentNotValidException) {
            baseResult.setCode(ErrorCode.PARAMETER_ERROR.getCode());
            baseResult.setMessage(handlerParamsException((MethodArgumentNotValidException) t));
            return baseResult;
        }else {
            baseResult.setCode(ErrorCode.SERVER_ERROR.getCode());
            baseResult.setMessage("fatal exception ,call admin");
            return baseResult;
        }
    }


    private String handlerParamsException(MethodArgumentNotValidException e) {
        List<FieldError> allErrors = e.getBindingResult().getFieldErrors();
        ObjectNode errorNode = mapper.createObjectNode();
        allErrors.forEach(objectError -> errorNode.put(objectError.getField(), objectError.getDefaultMessage()));
        return errorNode.toString();
    }

}

```

## 3.总结

spring-boot的slogan是build-anything,而且个人认为spring-boot借助现在的微服务的秋风已经在越来越来的生产环境当中使用，可以预见spring-boot会成为很多项目启动器，由spirng-boot来完成项目的启动和配置加载，这样大大减轻了在一些一些开发当中常见问题的处理，比如环境的处理，日志处理，监控的处理和发布的处理。

spring-boot吧各种通用的模块抽象出来，包括环境隔离，属性注入，日志，健康监控，形成统一配置，大大简化了开发流程，并且提供很多项目的继承的扩展点，可以随意的集成各种热门的组件，比如日志系统，消息系统，orm框架，鉴权，容器等，搜索系统等。

spring-boot一反以前spring的宽容的处理方式，为大家约定了一套生产环境统一的模板，简化各种开发流程，这是开发者的福音，但是spring又不强制使用，总是在各个地方留下各种扩展点，为个性化配置留了一条出路。



由于能力有限难免出现各种错误，希望大家一起学习探讨，轻喷！！！