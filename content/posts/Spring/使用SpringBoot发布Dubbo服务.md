---
title: 使用SpringBoot发布Dubbo服务
tags:
  - Spring
date: 2018-03-22
categories:
 - Spring
---

前言
==
通常我们部署dubbo服务时都是使用spring容器，这样一来配置就会很多，使用springboot部署dubbo服务，让你的操作变得如此简单。官方地址:[incubator-dubbo-spring-boot-project][1]，本文源码地址:[springboot-dubbo][2]
准备
==
使用Dubbo和SpringBoot之前，首先安装并配置好[Dubbo][3],[ZooKeeper][4],[Maven][5],并了解[SpringBoot][6]。

开发环境
====

 - 开发工具:intellij IDEA 2017.1.3
 - JDK:1.8.0_101
 - spring boot 版本:1.5.5.RELEASE
 - maven:3.3.9

开始
==

 - 首先，new->project，选择maven项目，取一个喜欢的名称，下一步即可![此处输入图片的描述][7]
 - 在新建的项目上右键，新建三个module，名称分别为:springboot-dubbo-rpc-api(服务接口),springboot-dubbo-rpc-service(服务实现，同时向zookeeper注册服务)，springboot-dubbo-web（restful接口端），操作与第一步相同，只不过是new ->module。![此处输入图片的描述][8]
 - springboot-dubbo的pom.xml中加入以下配置:

   
   ​     
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
      <modelVersion>4.0.0</modelVersion>
      <groupId>com.test</groupId>
      <artifactId>springboot-dubbo</artifactId>
      <packaging>pom</packaging>
      <version>1.0-SNAPSHOT</version>
      <modules>
        <module>springboot-dubbo-web</module>
        <module>springboot-dubbo-rpc-api</module>
        <module>springboot-dubbo-rpc-service</module>
      </modules>
      <!--加入此配置，子工程不需继承springboot-parent-->
      <dependencyManagement>
        <dependencies>
          <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-dependencies -->
          <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>1.5.5.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
          </dependency>
    
        </dependencies>
      </dependencyManagement>
      <!--指定编译级别-->
      <build>
        <plugins>
          <plugin>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>2.3.2</version>
            <configuration>
              <source>1.8</source>
              <target>1.8</target>
              <encoding>UTF-8</encoding>
            </configuration>
          </plugin>
        </plugins>
      </build>
   </project>
    ```
   


 生产者
===

 - 创建服务生产者，由两部分组成，分别是springboot-dubbo-rpc-api(服务接口),springboot-dubbo-rpc-service(服务实现，同时向zookeeper注册服务)，首先在springboot-dubbo-rpc-api项目的src下新建包，名称随意，并且新建接口类DemoService:

        ```java
     package springboot.dubbo.rpc.api;
        /**
        ** Created by zhangyq on 2018/3/22.
        */
        public interface DemoService {
             public String sayHello();
        }
        ```

     


 - springboot-dubbo-rpc-service:修改pom.xml文件,加入以下依赖:

   ​    

      ```xml
      <?xml version="1.0" encoding="UTF-8"?>
      <project xmlns="http://maven.apache.org/POM/4.0.0"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
          <parent>
              <artifactId>springboot-dubbo</artifactId>
              <groupId>com.test</groupId>
              <version>1.0-SNAPSHOT</version>
          </parent>
          <modelVersion>4.0.0</modelVersion>
          <artifactId>springboot-dubbo-rpc-service</artifactId>
         
         <dependencies>
         <!--接口程序-->
             <dependency>
                 <artifactId>springboot-dubbo-rpc-api</artifactId>
                 <groupId>com.test</groupId>
                 <version>1.0-SNAPSHOT</version>
             </dependency>
             <!--springboot-->
             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter-web</artifactId>
             </dependency>
             <dependency>
                 <groupId>com.alibaba.boot</groupId>
                 <artifactId>dubbo-spring-boot-starter</artifactId>
                 <version>0.1.0</version>
             </dependency>
             <!--如果不加此依赖启动时会报错找不到zkclient-->
             <dependency>
                 <groupId>com.101tec</groupId>
                 <artifactId>zkclient</artifactId>
                 <version>0.10</version>
             </dependency>
         </dependencies>
      </project>
      ```
   
 - springboot-dubbo-rpc-service:在src下新建包,名称随意，并且新建实现类DemoServiceImpl:

      ```java
       package springboot.dubbo.rpc.service.impl;
         
        import com.alibaba.dubbo.config.annotation.Service;
        import springboot.dubbo.rpc.api.DemoService;
          
        /**
         * Created by zhangyq on 2018/3/22.
         */
        @Service(
                version = "1.0.0",
                application = "${dubbo.application.id}",
        protocol = "${dubbo.protocol.id}",
                registry = "${dubbo.registry.id}"
        )
        public class DemoServiceImpl implements DemoService {
            @Override
            public String sayHello() {
                return "hello";
            }
        }
      ```

​     

 - springboot-dubbo-rpc-service:在resource下建立配置文件application.yml:

        ```yml
     spring:
            application:
             name: springboot-dubbo-demo
        dubbo:
          #扫描包路径
          scan:
            base-packages: springboot.dubbo.rpc.service.impl
          #应用id,name
          application:
            id: springboot-dubbo-demoService
            name: springboot-dubbo-demoService
          #服务提供者协议配置
          protocol:
            id: dubbo
            name: dubbo
            port: 20881
          #注册信息
          registry:
            id: demoService
            #zookeeper地址
            address: zookeeper://192.168.1.134:2181
        #使用springboot已集成的日志管理,如需要log4j等可自行配置
        logging:
          file: ./logs/zhangyq-shop-springboot-service.log
        ```

     

 

 - springboot-dubbo-rpc-service:建立springboot启动类DemoServiceApplication:

       ```java
      package springboot.dubbo.rpc.service;
        import org.springframework.boot.SpringApplication;
        import org.springframework.boot.autoconfigure.SpringBootApplication;
     
        /**
         * Created by zhangyq on 2018/3/22.
         */
        @SpringBootApplication
        public class DemoServiceApplication {
            public static void main(String[] args) {
                SpringApplication.run(DemoServiceApplication.class,args);
            }
        }
       ```

     

 - 运行此类，可在控制台看到日志输出，此时我们访问zookeeper的监控程序dubbo-admin，可以看到服务提供者已经注册到了zookeeper中。


消费者
===

 - springboot-dubbo-web:在src下新建包，并建立一个controller类DemoController:

       ```java
      package springboot.dubbo.web.controller;
          import com.alibaba.dubbo.config.annotation.Reference;
          import org.springframework.web.bind.annotation.GetMapping;
          import org.springframework.web.bind.annotation.RestController;
          import springboot.dubbo.rpc.api.DemoService;
     
          /**
           * Created by zhangyq on 2018/3/22.
           */
          @RestController
          public class DemoController {
              @Reference(
                      version = "1.0.0",
                      application = "${dubbo.application.id}",
                      interfaceClass = DemoService.class
              )
              private DemoService demoService;
              @GetMapping("/index")
              public String index(){
                  return demoService.sayHello();
              }
          }
     
      
       ```

     

 - springboot-dubbo-web:在resource下建立配置文件application.yml:

      ```yml
       server:
          port: 9090
        spring:
          application:
            name: web
        dubbo:
          application:
            id: springboot-dubbo-web
            name: springboot-dubbo-web
          registry:
            address: zookeeper://192.168.1.134:2181
        #日志
        logging:
          file: ./logs/zhangyq-shop-springboot-web.log
      ```

​     


 - springboot-dubbo-web:建立springboot启动类DemoWebApplication:

      ```java
       package springboot.dubbo.web;
         
        import org.springframework.boot.SpringApplication;
        import org.springframework.boot.autoconfigure.SpringBootApplication;
          
        /**
         * Created by zhangyq on 2018/3/22.
         */
        @SpringBootApplication(scanBasePackages = "springboot.dubbo.web.controller")
        public class DemoWebApplication {
            public static void main(String[] args) {
                SpringApplication.run(DemoWebApplication.class,args);
            }
        }
      ```

​     

 

 - 运行此类，可在控制台看到日志输出，此时我们访问zookeeper的监控程序dubbo-admin，可以看到服务消费者已经注册到了zookeeper中，在浏览器中输入http://localhost:9090/index可查看运行结果:

![此处输入图片的描述][9]


部署linux
==

 - 应用做好了，但是只是在IDEA里面启动，怎么能部署到linux上呢，在springboot-dubbo-rpc-service和springboot-dubbo-web项目的pom.xml中加入maven插件:

       ```xml
      <build>
                <plugins>
                    <plugin>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-maven-plugin</artifactId>
                        <version>1.5.5.RELEASE</version>
                        <!--<executions>
                            <execution>
                                <goals>
                                    <goal>repackage</goal>
                                </goals>
                            </execution>
                        </executions>-->
                        <!--打包为可执行jar包-->
                        <configuration>
                            <executable>true</executable>
                        </configuration>
                    </plugin>
                </plugins>
        </build>
       ```

     

 - 添加maven build参数:mvn clean package spring-boot:repackage并运行

 - 将生成的jar包上传到Linux服务器中，这里使用systemd service的方式运行:

      ```shell
       //新建服务
        $ cd /etc/systemd/system/
        $ vim demo.service
        //加入以下内容
        [Unit]
        Description=demo
        After=syslog.target
          
        [Service]
        User=用户
        ExecStart=jar包的路径
        SuccessExitStatus=143
          
        [Install]
        WantedBy=multi-user.target        
      ```

​     


 - 创建成功后，我们就可以用systemctl start demo.service ，systemctl stop demo.service来开启和关闭服务了，是不是很方便呢。

   [1]: https://github.com/apache/incubator-dubbo-spring-boot-project
[2]: https://github.com/suiyueranzly/springboot-dubbo
[3]: http://dubbo.io/
[4]: http://zookeeper.apache.org/
[5]: https://maven.apache.org/
[6]: https://projects.spring.io/spring-boot/
[7]: http://suiyueranzly.oss-cn-beijing.aliyuncs.com/18-7-13/62405057.jpg
[8]: http://suiyueranzly.oss-cn-beijing.aliyuncs.com/18-7-13/78733576.jpg
[9]: http://suiyueranzly.oss-cn-beijing.aliyuncs.com/18-7-13/39356878.jpg