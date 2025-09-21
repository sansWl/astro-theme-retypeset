---
 title: Bat脚本启动本地Kafka
 published: 2025-03-11
 tags:
  - BlogPost
 lang: zh
 abbrlink:  bc002c30-f7c2 
---

# 一：功能介绍
> 本地启动Kafka，当需要测试多个Kafka Broker时使用脚本启动多个实例
# 二：使用介绍 
> 编写 `bat` 文件，将下述代码填入保存，***注意路径配置***  

![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250311152627310-894820133.png)

![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250311152735206-581968248.png)

```bat
@echo off
@REM 声明 UTF-8 编码，避免乱码问题
chcp 65001
setlocal enabledelayedexpansion  
@REM 代表脚本启动的当前路径，如上图，脚本处于kafka\kafka_2.13-xxx
set cds=%~dp0 
set zookeeper_config=%cds%config\zookeeper.properties
set zookeeper_Server=zookeeper-server-start.bat
set kafka_server=%cds%bin\windows\kafka-server-start.bat
set brokernum=0
@REM %1 表示第一个参数，没有参数即server.properties缺失
if "%1" == "" ( echo 请输入Kafka的Broker的配置文件路径 
) else (

   @REM 启动zookeeper
   start "zookeeper" cmd /k call %cds%bin\windows\%zookeeper_Server%  %zookeeper_config%
   timeout /t 2

   @REM 按参数顺序启动 Kafka broker实例 打开的窗口按照 kafka_1、kafka_2命名格式
   for %%i in (%*) do (
    set /a brokernum += 1
    @Rem  "kafka_"!brokernum!  窗口命名
    start "kafka_"!brokernum! cmd /k call %kafka_server% %cds%%%i
   )

)



pause >nul
```