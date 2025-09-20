---
 title: MQTT认证【一】
 published: 2025-03-06
 tags:
  - BlogPost
 lang: zh
 abbrlink:  46918707-3b4d 
---

`1. ` 认证架构图
![](https://docs.emqx.com/assets/emqx-authn-flow.nqM4_Tys.png) 
> Auth & ACL 钩子作为验证功能的扩展
EMQX 认证器
按照认证方式和数据源来划分，EMQX 内置了以下 8 种认证器：

|认证方式	|数据源	|说明|
| -----  | -----|-----| 
|密码认证	|内置数据库|	[使用内置数据库（Mnesia）进行密码认证](https://docs.emqx.com/zh/emqx/v5.0/access-control/authn/mnesia.html)|
|密码认证	|MySQL	|[使用 MySQL 进行密码认证](https://docs.emqx.com/zh/emqx/v5.0/access-control/authn/mysql.html)|
|密码认证	|PostgreSQL	|[使用 PostgreSQL 进行密码认证](https://docs.emqx.com/zh/emqx/v5.0/access-control/authn/postgresql.html)|
|密码认证	|MongoDB	|[使用 MongoDB 进行密码认证](https://docs.emqx.com/zh/emqx/v5.0/access-control/authn/mongodb.html)|
|密码认证	|Redis	|[使用 Redis 进行密码认证](https://docs.emqx.com/zh/emqx/v5.0/access-control/authn/redis.html)|
|密码认证	|HTTP Server	|[使用 HTTP 服务进行密码认证](https://docs.emqx.com/zh/emqx/v5.0/access-control/authn/http.html)|
|JWT		|--|[JWT 认证](https://docs.emqx.com/zh/emqx/v5.0/access-control/authn/jwt.html)|
|增强认证	|内置数据库	|[MQTT 5.0 增强认证(SCRAM 认证)](https://docs.emqx.com/zh/emqx/v5.0/access-control/authn/scram.html)|

`2. `认证接入
> 官网提供DashBoard认证管理页，如果期望自定义Hook实现，需要使用插件扩展开发，以gRpc为例；
[官方给出的代码示例](https://github.com/emqx/emqx-extension-examples)