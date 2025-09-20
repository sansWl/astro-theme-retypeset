---
 title: CNB【腾讯的云原生开发平台】部署 RAGFlow
 published: 2025-03-04
 tags:
  - BlogPost
 lang: zh
 abbrlink:  c3ff95f8-2fac 
---

**为什么使用CNB** 
> 具有不错的免费额度使用，每月刷新，平台硬件配置够用8核 16G内存，完全满足常见的AI模型开发环境
练习docker、linux，不再局限于WSL、vm硬件配置不够用，具有WebIDE（Vscode）
https://docs.cnb.cool/zh/saas/pricing.html 【计费规则】
https://docs.cnb.cool/zh/vscode/quick-start.html

**开始部署**
> 1. 创建一个仓库,开启云平台初始化，使用cnb-init-from ~~https://your-git.com/your-repo.git~~
 ~~git clone~~ https://github.com/infiniflow/ragflow.git [RAGFlow的GitHub仓库]
![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250304023728808-769939825.png)
**初始化后，使其文件依赖结构如图所示，否则ragflow-server启动将失败**
**OSError: [Errno loading yaml file config from /ragflow/conf/service_conf.yaml failed:] [Errno 2] No such file or directory: '/ragflow/conf/service_conf.yaml'**

>2. Ragflow的部署，待下载完Ragflow（大约9G），进入到上图的ragflow/docker目录下，启动 `docker compose -f docker-compose.yml up -d`
使用 `docker logs -f ragflow-server` 查看是否启动完成

>3. https://ragflow.io/docs/dev/ 开始学习