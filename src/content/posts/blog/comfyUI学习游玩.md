---
title: comfyUi学习游玩
published: 2025-09-18
tags:
  - 软件
  - AI工具
  - 推荐
lang: zh
abbrlink:  comfyui-studypreview
---

### 1. 软件介绍
comfyUI 是一个利用 AI 大模型来进行文创的工作流软件，包括文生图、文生视频、生成3D模型等。

### 2. 软件安装
[安装指导](https://comfyui-wiki.com/zh/install/install-comfyui/comfyui-desktop-installation-guide)
> 国内，以及新手用户推荐安装秋叶整合包

### 3. 软件使用
#### 3.1 模型
> comfyUI 是利用大模型依据提示词(Prompt)和设定好的工作流，进行流水作业，输出图、视频、3D模型等 。
> 1. 安装模型： 
> 模型分类，如下图： checkPoint、vae 、lora、ControlNet等，checkPoint是理解提示词的模型（最基本最重要的模型），lora是风格迁移的模型（譬如，西部、日系、动漫），vae是生成图像的模型（提高图像质量），ControlNet控制生成期望的结果的模型（比如人物姿势，蒙版填充等），embedding是提示词模型（提高提示词效果以及避免写大量提示词，减少无意义的提示词）。
![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920220242099-2018688605.png)
> 2.模型下载存放位置
![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920220915899-913863422.png)
> 3. 模型下载：
 [huggingface](https://huggingface.co/) 以及 [CivitAI](https://civitai.com/) 等，按需下载，**注意**: vae 等增强 checkpoint模型的模型，需要和对应的模型版本匹配，否则可能无法理解和解析。
#### 3.2 工作流
 > 上图展示了一个较为基本的工作流，更多工作流可查看[OpenArt](https://openart.ai/workflows/home)、[runninghub(国内)](https://www.runninghub.ai/?inviteCode=rh-v1121)等<br> 基础工作流例子可查看官方提供的工作流，工作流保存目录如下：
 ![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920222212509-801206479.png)
#### 3.3 自定义节点
> 当你下载上述网址的分享工作流后，可能会存在本地没有的节点，这时需要下载这些缺失节点，你可以选择通过官方提示进行下载，或者手动通过git下载
![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920225207649-1115064780.png)
![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920222625140-441118962.png)
#### 3.4 自定义节点结构
![简单自定义节点基本结构](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920223731563-1657179072.png)


![复杂自定义节点基本结构-阿里千问视频模型节点](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920225557374-275739624.png)


![节点信息](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920230333624-1599168561.png)

[官方文档-自定义节点](https://docs.comfy.org/custom-nodes/walkthrough)
1. `node.py`: 节点的定义和注册
2. `___init__.py`: 客户端的扩展
3. `js`: 逻辑功能的实现
