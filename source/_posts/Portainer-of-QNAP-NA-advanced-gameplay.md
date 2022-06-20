---
title: 威联通NAS进阶玩法之 Portainer
tags: 
- NAS
date: 2022-06-19 08:00:00
---

Portainer 是一个可视化的容器镜像的图形管理工具，利用Portainer 可以轻松构建，管理和维护Docker环境。 而且完全免费，基于容器化的安装方式，方便高效部署。

官网地址：https://www.portainer.io/

### 登录NAS
首先在 NAS 控制面板开启 SSH 登录功能。接着使用 `admin` 账号密码登录（只能使用 admin)。

### 搜索镜像
docker search portainer

### 拉取镜像
docker pull portainer/portainer

### 运行镜像

    docker run -d -p 9001:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /portainer_data:/data/portainer/portainer portainer/portainer

参数介绍：
- "-d"代表"后台运行容器，并返回容器ID"
- "-p"代表"容器内部端口随机映射到主机的高端口"，前面的9000是容器默认端口，后面的9000是安装后映射的端口（冒号前后）
- "--name" 代表容器的名字
- "--restart always"代表总是Docker启动后容器自动启动
- "-v"表示路径映射，portainer的路径映射用默认就行，如果为了方便迁移可以映射到Nas的实体路径

### 访问Portainer容器

    http://<你的NAS IP地址>:9001
