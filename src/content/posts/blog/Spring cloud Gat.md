---
 title: Spring cloud GateWay
 published: 2025-03-11
 tags:
  - BlogPost
 lang: zh
 abbrlink:  47690543-517e
---

# 一：配置
> GateWay本身既有webflux不能使用SpringMvc依赖，否则无法启动，spring-cloud-config依赖会导致配置文件错误，需要移除依赖
```yml
spring:
  application:
    name: Gateway
  cloud:
    gateway:
      routes:
        - id: service1  #网关服务id
          uri: http://localhost:8081   #目标服务器地址
          predicates:    #接受一组规则，返回布尔值
            - Path=/service1/**   #路由断言工厂名称 = 键值对  断言必须配置
            - Weight= group{String: } ，weight{int: }
          filters:
          #自定义过滤工厂名称
          - name: DemoGateWayFilterFactory
            args:
              name: foo,bar
              value: 1,100
          #uri跳过字段，例如   /service1/hello ==》/hello
          - StripPrefix=1
```

![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250311155047870-147292185.png)
