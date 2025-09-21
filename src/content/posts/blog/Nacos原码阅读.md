---
title: Nacos v2.5 源码阅读
published: 2025-09-20
author: Nacos
tags:
  - Nacos
lang: zh
abbrlink: nacos-code-reading
---

### 1. Nacos 包结构
[Nacos 官网](https://nacos.io/docs/v2.5/quickstart/quick-start/?spm=5238cd80.2ef5001f.0.0.3f613b7cHyO4x8)

> 通过查阅官网，可以看到服务注册发现、配置读取是利用的HTTP协议，因此，我们可以从Nacos的源码中找到HTTP协议的实现。

Nacos 编译包结构如下图所示：

![反编译后的 Nacos 包结构](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250922015511683-151104998.png)

> 其中，服务注册相关的包为：nacos-naming，配置读取相关的包为：nacos-config。

### 2. Nacos 服务注册发现实现

#### 2.1 服务注册
 > `curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'` <br>
 > 官方文档提到使用了 /nacos/v1/ns/instance 的Post请求来注册服务。<br>
 > 我们可以查看 Nacos-naming 模块的源码，找到注册服务的实现。<br>

 ![注册实例 controller](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250922022525102-1249828774.png)

 ```java
 @CanDistro
  @PostMapping
  @TpsControl(pointName = "NamingInstanceRegister", name = "HttpNamingInstanceRegister")
  @Secured(action = ActionTypes.WRITE)
  public String register(HttpServletRequest request) throws Exception {
    // 命名空间
    String namespaceId = WebUtils.optional(request, "namespaceId", "public");
    String serviceName = WebUtils.required(request, "serviceName");
    NamingUtils.checkServiceNameFormat(serviceName);
    // 实例信息构建, 包括ip、port、clusterName、weight、metadata等
    // （即上述curl命令所传参 ip=20.18.7.10&port=8080）
    Instance instance = HttpRequestInstanceBuilder.newBuilder().setDefaultInstanceEphemeral(this.switchDomain.isDefaultInstanceEphemeral()).setRequest(request).build();
    // 注册实例
    getInstanceOperator().registerInstance(namespaceId, serviceName, instance);
    // 发布事件
    NotifyCenter.publishEvent((Event)new RegisterInstanceTraceEvent(System.currentTimeMillis(), 
          NamingRequestUtil.getSourceIpForHttpRequest(request), false, namespaceId, 
          NamingUtils.getGroupName(serviceName), NamingUtils.getServiceName(serviceName), instance.getIp(), instance
          .getPort()));
    return "ok";
  }

  /**
   * 注册实例 , 来自 class InstanceOperatorClientImpl implements InstanceOperator
   *
   * @param namespaceId 命名空间
   * @param serviceName 服务名
   * @param instance 实例信息
   * @throws NacosException 异常
   */
    public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    NamingUtils.checkInstanceIsLegal(instance);
    boolean ephemeral = instance.isEphemeral();
    String clientId = IpPortBasedClient.getClientId(instance.toInetAddr(), ephemeral);
    createIpPortClientIfAbsent(clientId);
    Service service = getService(namespaceId, serviceName, ephemeral);
    this.clientOperationService.registerInstance(service, instance, clientId);
  }
 ```

 #### 2.2 配置读取
 > `curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.test.1&group=test&tenant=public'` <br>
 > 官方文档提到使用了 /nacos/v1/cs/configs 的Get请求来读取配置。<br>
 > 我们可以查看 Nacos-config 模块的源码，找到读取配置的实现。<br>

 ![config controller](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250922023257142-1653272495.png)

 ![配置存放路径](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250922023604704-938823942.png)

 ##### 2.2.1 Nacos实现 Spring boot 热更新

 > 1. `Nacos` 服务端 发送配置变更信息，项目依赖中的 `client 实例` 会收到配置变更通知，然后修改Spring Boot的配置对象。<br>
 > 2. 发布事件通知，`Spring Boot` 监听到配置变更事件，重新加载配置。<br>

![更新配置事件](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250922040434461-71429088.png)

 ![热更新执行流程](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250922035931400-1477783068.png)