---
 title: docker 安装 oracle database 问题记录
 published: 2025-03-16
 tags:
  - BlogPost
 lang: zh
 abbrlink:  7a364d5c-6c2f 
---

#pre
> 本地docker （WSL）安装运行 Oracle
#1. 镜像处理
> 参考链接：https://www.cnblogs.com/wuchangsoft/p/18344847
> `oracle` 镜像获取：
> 1. https://container-registry.oracle.com/ords/f?p=113:10:::::: (Oracle官网,由于部分问题导致直接pull无法拉取)
> 2. 阿里云，参考链接里有个个人19c版本的
> 3. 官网下载Linux版本，自己编写image

#2. 容器创建
> 基本操作查看参考链接；
> 介绍几个运行会遇到的问题：
> `1.` docker 目录卷绑定赋权，如果不提前赋权，容器创建对应目录会发生报错，出现权限问题，单纯使用`sudo`命令无法解决
> `2.` 查看日志等待安装完毕（`docker logs -ft container_id`）,进入容器里后使用命令
```vi
 echo "DISABLE_OOB=ON" >> /opt/oracle/oradata/dbconfig/XE/sqlnet.ora
```                                        
> 这里的XE是Oracle的实例（SID），如果按照上述参考链接运行，则这里的`XE`就为`ORCLCDB`，如果不处理会导致数据库连接不了
> **参考链接：**https://stackoverflow.com/questions/19660336/how-to-approach-a-got-minus-one-from-a-read-call-error-when-connecting-to-an-a
> `3.` 注意驱动版本问题，进入容器使用 `sqlplus -v` 查看驱动版本
![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250316205925218-407436518.png)


#3. Oracle链接
> 1. 切换库，`alter session set container=${your database}`
> 2. 创建用户,`create user ${userName} identified by ${password}` (注意，这个用户只针对你创建时的库，连接时使用这个账号登录需要注意)
> 3. 驱动版本thin、oci、oci8;
> 不同点 **参考连接** :https://stackoverflow.com/questions/21711085/what-is-the-difference-between-oci-and-thin-driver-connection-with-data-source-c
![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250316210138050-748327833.png)
