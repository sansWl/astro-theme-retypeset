---
 title: MVCC 版本并发控制
 published: 2025-03-11
 tags:
  - BlogPost
 lang: zh
 abbrlink:  b90d6666-487a 
---

# 一 MVCC前提
> 必须是支持`ACID`的存储引擎，譬如Mysql的`InnoDB`  <br>
> Mysql中mvcc的实现是利用`undolog`日志和 `事务id` 以及 `ReadView`实现； <br>
> `ReadView（视图）`主要实现的完成事务间的数据可见性 <br>
> Mysql维护着redolog 、binlog、undolog三种日志：
> <a href="https://blog.csdn.net/Weixiaohuai/article/details/117896523" name="redolog">redolog</a>：用于数据库异常宕机的恢复工作；innodb存储引擎特有的；物理层；循环覆写 <br>
> <a href="https://www.cnblogs.com/Presley-lpc/p/9619571.html" name="redolog">binlog</a>: server共有、逻辑层；备份；追加写入append (需要配置文件开启默认关闭，每次重启都会创建新的binlog，且目录会创建mysql-binlog.index文件，内容为binlog文件名，会先从第一行读取，若第一行空会导致报错)<br>
> <a href="https://zhuanlan.zhihu.com/p/383824552" name="redolog">undolog</a> : 用于事务回滚，意为撤销或取消，以撤销操作为目的，将数据返回到某个状态的操作，实现事务的原子性

# 二 操作逻辑
> 利用Undolog文件中的回滚指针进行数据的回退,利用事务id和ReadView实现事务间的可见性 <br>
> 二级索引可通过回表操作实现MVCC，另二级索引维护了自己最后操作的事务id `MAX_TRX_ID`,再通过 ReadView 中活跃事务id判断可见性；<br>
>DB_TRX_ID：记录最后一次修改该行的事务 ID。<br>
>DB_ROLL_PTR：指向 Undo Log 的指针，用于访问该行的历史版本。<br>

# 三 视图可见性
> 视图依据创建时间维护一组活跃的事务ids，和视图创建范围的事务id，包含生成此视图后下一个生成的事务id和生成视图前活跃事务的最小id，利用事务id关系完成可见性判断

![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250311160745728-1401991784.png)
