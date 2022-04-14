---
title: docker 端口的 expose 与 publish
date: 2018-08-24 10:36:48
tags: 技术
---

在 `Dockerfile` 里通过 `Expose` 导出一个 `80` 的端口:

```bash
EXPOSE 80
```

启动容器后,发现还是无法本机的 `80` 端口访问容器内服务。只有在启动时通过 `-p` 参数publish 端口才可以，像下面这样:

```bash
docker run -p 8080:80 demo
```

这是为啥呢?
都已经在 `Dockerfile` 里面 `Expose` 为啥还不能直接访问?

## 实验
使用下面4中不同的方法启动容器:

- 不在 `Dockerfile` 里 `EXPOSE`，也不通过 `-p` 参数指定
- 在 `Dockerfile` 里 `EXPOSE`，但不使用 `-p` 参数
- 在 `Dockerfile` 里 `EXPOSE`，也使用 `-p` 参数
- 只使用 `-p` 参数

## 结果

- 第一中情况: 不能在外网访问，也不能被 link 的 container 访问
- 第二种情况: 不能被外网访问，但是能被 link 的 container 访问
- 第三种情况: 能被外网访问，也能被 link 容器访问
- 第四中情况: 和第三种情况一样

## 结论
`EXPOSE` 只是导出端口。只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。
在 `Dockerfile` 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 `docker run -P` 时，会自动随机映射 `EXPOSE` 的端口。

通过 `docker ps` 命令，可以看到，端口的映射关系。