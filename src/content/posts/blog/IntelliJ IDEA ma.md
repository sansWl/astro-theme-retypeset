---
 title: IntelliJ IDEA maven plugin build 失败 Process finished with exit code -1073741819 (0xC0000005)
 published: 2024-12-06
 tags:
  - BlogPost
 lang: zh
 abbrlink:  f6f5e263-ddf6 
---

1. `Help | Find Action` or `Idea全局搜索:双shift `
![](https://img2024.cnblogs.com/blog/3426265/202412/3426265-20241206190713167-1373741840.png)

2. 输入registry打开，寻找注册表中`debugger.attch.to.process.action`，默认处于开启状态，将其关闭即可。
![](https://img2024.cnblogs.com/blog/3426265/202412/3426265-20241206191050756-938598207.png)

3. 清除缓存，重启即可























>参考链接：https://youtrack.jetbrains.com/issue/IDEA-203172.