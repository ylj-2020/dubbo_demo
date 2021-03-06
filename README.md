# Dubbo

## 简介

1. Dubbo 前生是阿里的Java RPC框架，可以和Spring无缝集成，目前已经捐献给Apache 管理
2. java中类似的RPC框架有RMI,Hession
3. Dubbo提供了三大核心功能
    1. 面向接口的远程过程调用
    2. 智能容错和负载均衡 
    3. 服务自动注册和发现

## SOA

1. 面向服务的架构
2. 可以让服务实现编排，实现业务的灵活组建
3. 如下图中的基础用户服务和订单服务可以被两个不同的系统共同依赖

![image-20210114203849583](https://yljnote.oss-cn-hangzhou.aliyuncs.com/2021-01-14-123850.png)



### 架构

![dubbo架构](http://dubbo.apache.org/imgs/user/dubbo-architecture.jpg)

节点角色说明

| 节点        | 角色说明                               |
| ----------- | -------------------------------------- |
| `Provider`  | 暴露服务的服务提供方                   |
| `Consumer`  | 调用远程服务的服务消费方               |
| `Registry`  | 服务注册与发现的注册中心               |
| `Monitor`   | 统计服务的调用次数和调用时间的监控中心 |
| `Container` | 服务运行容器                           |

## Register注册中心-zookeeper

- zookeeper是一个树状结构的目录服务

- 支持变更推送，目录有变更，根据长连接推送给消费方

    ![image-20210114222743873](https://yljnote.oss-cn-hangzhou.aliyuncs.com/2021-01-14-142744.png)

## 安装注意点

1. 复制conf/zoo_sample.cfg 为zoo.cfg
2. 将其中的dataDir修改到/usr/local/zookeeper-3.4.6/data ，data为新建数据文件夹

## 常用命令

1. 启动 

    ```sh
    ./zkServer.sh start
    ```



## 监控服务部署

- 将监控war包部署后，配置dubbo.properties文件中zk的地址，启动后会自动监控
- http://121.4.53.107:8080/dubbo-admin-2.6.0/

## 使用

- 导坐标

    ```xml
    <!-- dubbo相关 -->
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>dubbo</artifactId>
                <version>2.6.0</version>
            </dependency>
    
    <!-- zookeeper注册中心 -->
            <dependency>
                <groupId>org.apache.zookeeper</groupId>
                <artifactId>zookeeper</artifactId>
                <version>3.4.7</version>
            </dependency>
            <dependency>
                <groupId>com.github.sgroschupf</groupId>
                <artifactId>zkclient</artifactId>
                <version>0.1</version>
            </dependency>
    
    <!-- 动态代理，构建字节码 -->
            <dependency>
                <groupId>javassist</groupId>
                <artifactId>javassist</artifactId>
                <version>3.12.1.GA</version>
            </dependency>
    
    <!-- 对象传输，序列化 -->
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>fastjson</artifactId>
                <version>1.2.47</version>
            </dependency>
    ```

- 提供方配置

    ```xml
        <!--每个dubbo应用(服务提供方和服务消费方)都必须指定一个唯一的名称-->
        <dubbo:application name="dd_provider"></dubbo:application>
        <!--指定服务的注册中心-->
        <dubbo:registry address="zookeeper://121.4.53.107:2181"></dubbo:registry>
        <!--配置协议和端口-->
        <dubbo:protocol name="dubbo" port="20881"></dubbo:protocol>
    
    ## 服务发布方式一：包扫描方式发布服务
        <!--指定包扫描，用于发布dubbo服务-->
        <dubbo:annotation package="cn.ylj.api.impl"></dubbo:annotation>
    
    ## 服务发布方式二：标签配置发布服务
     <!-- 配置方式发布服务   -->
        <bean id="helloService" class="cn.ylj.api.impl.HelloServiceImpl"></bean>
        <!--  动态代理生成IHelloService代理类，目标对象是id="helloService"的bean，增强的功能是dubbo的功能 -->
        <dubbo:service interface="cn.ylj.api.IHelloService" ref="helloService">				</dubbo:service>
    ```

- 消费方配置

    ```xml
    <dubbo:application name="consumer"></dubbo:application>
    <dubbo:registry address="zookeeper://121.4.53.107:2181"></dubbo:registry>
    
    ## 服务获取方式一：
    <dubbo:annotation package="cn.ylj.controller"></dubbo:annotation>
    
    ## 服务获取方式二：
    ## 配置方式获取服务，dubbo会将生成代理对象在IoC容器中，使用服务和使用本地一样@Autowired就行
    <dubbo:reference id="helloService" interface="cn.ylj.api.IHelloService"></dubbo:reference>
        <context:component-scan base-package="cn.ylj.controller"></context:component-scan>
    
    <!-- 启动时服务未启动 消费端也能正常启: 项目部署时打开true   -->
    <dubbo:consumer check="false"></dubbo:consumer>
    ```



## dubbo传输协议简介

- 传输协议一般在服务的提供方配置

- 其中Dubbo支持的有：dubbo,rmi,hessian,http,webservice,redis等

- dubbo推荐的是dubbo协议，采用了单一长连接和NIO异步通讯

    适用于

    1. 小数据量，大并发的服务调用
    2. 消费者机器 远远大于服务者机器的场景

    不适合

    1. 传大文件，视频等业务场景

- 当然可以给每一个RPC服务指定 传输的协议

## 负载均衡 (Load Balance)

- dubbo使用的软负载均衡，不是像nginx一样，有个中心去分配请求

    而是由服务的双方根据请求的响应情况，自动协调。

- 负载均衡策略：1. 随机(默认) 2.轮询 3.最少活跃 3.一致性hash

    ```java
    @Service(loadbalance = "random")
    public class HelloServiceImpl implements HelloService{
      
    }
    ```

## Dubbo RPC过程的事务控制

- Spring的@Transaction注解的原理是 创建JDK代理对象，而这个代理对象的全限定名为 `com.sun.proxy.$PrxyXX`, 而dubbo的@Service修饰后，发布服务会根据修饰的服务类的全限定名去找，就找不到了。服务发布就会失败。

- 解决： 

    1. 用cglib动态代理，代理后全限定名不变

        ```xml
        <!--  事务控制，创建事务动态代理对象用cglib  -->
            <tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="true"></tx:annotation-driven>
        ```

    2. 指定发布服务的类型(因为用cglib代理会用新的接口)

        ```java
        @Service(interfaceClass = IUserSerivce.class)
        ```

        



