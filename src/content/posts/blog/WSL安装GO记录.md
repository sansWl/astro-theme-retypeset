---
 title: WSL安装GO记录
 published: 2025-03-13
 tags:
  - BlogPost
 lang: zh
 abbrlink:  d72babf7-8dfb 
---

# 一：下载
> 在官网按照系统匹配的版本下载（手动下载避免`apt`等下载工具暂未更新包）

![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250313212320916-812339548.png)

# 二：解压、配置
> ` rm -rf /usr/local/go && tar -C /usr/local -xzf go1.24.1.linux-amd64.tar.gz`
> 如果安装有旧的`go`，需要删除旧版本在安装新版本；官网提供的指令可能无法运行成功，需要`sudo`权限（注意：可能导致后续依赖无法import）
> 配置env环境变量，`export PATH=$PATH:/usr/local/go/bin` 在 WSL 中修改 /etc/profile 文件内容 （官网教程到此为止）


# 三：问题
> `1.` 修改代理 `go env -w GOPROXY=https://goproxy.cn` (国内可能导入依赖失败，修改proxy处理) 

> `2.` <a href="https://stackoverflow.com/questions/74325538/golang-mkdir-usr-local-go-pkg-mod-permission-denied">golang mkdir /usr/local/go/pkg/mod permission denied</a> (这里是直接拿的Stackoverflow上的问题，具体解决方案差不多); <br>
> 主要原因是`GOPATH` 和 `GOMODCACHE`这两个参数导致（使用`go env`查看），下图显示的为本人安装GO后此参数目录，使用`go mod tidy`报permission-denied，本人尝试了stackoverflow上描述的设置权限解决方案`chmod 775` 无法起效，而且在对应目录下使用`sudo mkdir xxx`依旧不起效；<br>
![](https://img2024.cnblogs.com/blog/3426265/202503/3426265-20250313214136126-119204309.png)
>  解决：`export GOPATH=$HOME/go/lib` 在`/etc/profile`切换 `GOPATH`参数，`GOMODCACHE`可以直接使用`go env -w GOMODCACHE= `,默认保持 
> `GOPATH` 和 `GOMODCACHE`处在同一目录，`GOMODCACHE` 将创建`pkg`目录
