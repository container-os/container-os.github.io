---
title: Easystack container Linux
description: hello containers
---

easystack container Linux是一款基于原子项目Project Atomic的轻量级容器操作系统，
主要目标是面向云平台的虚拟机操作系统。easystack container linux是以ESCloud Linux为基础设计的，
不仅继承了ESCloud Linux的高效性和稳定性，而且相对普通虚拟机采用更精简的kernel内核和更丰富的系统工具，
使系统运行更快，更高效。完全继承了atomic的本身的特性，包括快速部署和OS版本升级回退。 easystack container
Linux专门为提高容器效率而定制，使其成为云环境的kubenetes/Docker运行环境的最佳选择。
<!--more--> 

easystack container Linux采用了immutable的系统架构，保证系统安全稳定，并提供系统级的升级和回退功能，
让用户只需关注容器本身，而容器操作系统由企业统一来管理。easystack container Linux操作系统支持版本管理，
一步实现新版本的下载与部署，大幅简化系统升级流程。升级后同时保留旧版本的分支，可以方便快速地实现系统的回滚，
减少系统维护所耗费的时间。

# easystack container Linux 系统结构图
![COS](/images/infrastructure.png)


随着各种系统组件容器化越来越成熟，包括etcd，kubernetes，运行容器依赖系统的要求越来越低，为了系统更加轻量级，
easystack container linux通过集成busybox utils来缩减系统容量以及软件包的依赖。为了解决依赖问题，busybox
自身提供对应包的依赖，包括grep，which等。



easystack container linux通过对linux的裁剪让内核更小，专门适用于虚拟机容器操作系统，并通过自建的initramfs
并集成到kernel中，简化了内核的加载initramfs流程，这样让container linux在启动的时候更快，启动时间比centos atomic
启动时间减少了30%～40%。



为了可以进一步减小系统的容量，easystack container linux支持yak组件，让easystack container linux部署的旧版本
可以不用保存在客户端。客户端只保留必须的rollback源数据，因为easystack container linux的系统是只读的，对于旧版本
只读的系统部分完全可以直接用服务器端来获取。
