---
title: Docker网络模式不完全指南
tags: 
- 技术
- HTTPS
- 网络
- Docker
date: 2018-10-10 08:00:00
---

本次分享没有：
* Docker的背景
* Docker的整体架构
* Docker的核心实现技术
* Docker的高级用法
* Docker的使用秘笈

本次分享是：
*Docker使用中的...*
*一条命令的...*
*一个参数的..*
*不完全说明.*

### Docker网络模式

一般文章中说的网络模式，其实主要是指 `docker run` 命令的 `--net` 或 `--network` 参数所支持的模式，默认包括：

1. bridge模式（划重点，最后说）
    使用 `--network=bridge` 指定，默认设置可不指定
2. host模式
    使用 `--network=host` 指定
3. container模式
    使用 `--network=container:NAME_or_ID` 指定
4. none模式
    使用 `--network=none` 指定

有时网络模式也会包括其他的模式，例如 [macvlan](https://docs.docker.com/network/macvlan/)、[overlay](https://docs.docker.com/network/overlay/) 等，这些并不仅通过上述参数指定，属于高级用法，如有需要可查阅文档。
这里只介绍常见的上述4种模式。

#### host模式
容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。
* 试验
  创建容器：
```
docker run --rm -d --network host --name my_nginx nginx
```
在宿主机检查端口监听情况：
```
sudo netstat -tulpn | grep :80
```

通过这种模式，一些命令行工具可以很方便地通过docker来使用，避免在宿主机安装大量的依赖包，也便于随时清理。例如启动一个 tcpdump 的容器抓取主机上的网络报文：
```
docker run --net=host -v $PWD:/data corfr/tcpdump -i any -w /data/dump.pcap "icmp"
```

#### container模式
在了解了 host 模式后，这个模式也就好理解了。这个模式创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。
kubernetes 的 pod 可以认为就是用这个实现的（？），同一个 pod 中的容器共享一个 network namespace。
* 试验
  我们运行两个 nginx 容器：web1 和 web2：
  web1 监听在 80 端口，使用默认的网络模型
  web2 监听在 8080 端口，使用 container 网络模型共享 web1 的网络
  先启动 web1，通过端口映射把端口绑定到主机上：
```
docker run -d --name=web1 -p 80:80 nginx
```
使用 curl 命令验证容器运行正常：
```
curl http://localhost:80
```
第二个容器和 host 模式相同，使用 --net 参数让新建的容器使用 web1 的网络：
```
docker run --name=web2 -v ${PWD}/default.conf:/etc/nginx/sites-available/default -v ${PWD}/index.html:/var/www/html/index.html -d --net=container:web1 nginx
```
其中 `default.conf` 文件就是修改了 nginx 默认配置文件的端口，把它变成 8080；`inedx.html` 可以随便修改一点，以区别于默认的内容。
在 web1 或 web2 容器里面可以验证 nginx 服务：
```
curl http://127.0.0.1:8080
```
在两个容器中，可以分别通过 `ip addr` 查看网络配置，是完全一致的（命令和执行结果从略）。

#### none模式
这个模式和前两个不同。在这种模式下，这个 Docker 容器没有网卡、IP、路由等信息。需要我们自己为 Docker 容器添加网卡、配置 IP 等。
选择这种模式，一般是用户对网络有自己特殊的需求，不希望 docker 预设置太多的东西。

#### bridge模式
在默认的bridge模式下，docker 会在宿主机上新创建一个网桥，可以把它想象成一个虚拟的交换机，所有的容器都是连到这台交换机上面的。docker 会从私有网络中选择一段地址来管理容器，比如 172.17.0.1/16（这个地址根据你之前的网络情况而有所不同）。通过网桥，让容器的子网可以访问宿主机所连接的外网，并且可以通过 **端口映射** 来实现容器对外暴露端口提供服务（即外部可以通过宿主机、网桥访问容器上的服务）。这是最常用的网络模式，又分为以下两种情况：

1. 使用默认网络

默认的网络docker将其称为 `bridge` 网络，在这种情况下，容器可以相互通信（若出于安全考虑，也可以禁止它们之间通信，方法是在 `DOCKER_OPTS` 变量中设置 `--icc=false`，这样只有使用 `--link` 选项才能使两个容器通信，关于 `--link` 后面还会说到）。
容器可以访问外部网络，但是Docker容器的IP、网络等对外是不可见的。即外部服务发现的访问客户端IP，是宿主机IP而不是容器IP。
而通过端口映射，可以让外部访问Docker容器的服务。
* 试验
  我们首先用下面命令创建一个含有 web 应用的容器，将容器的 80 端口映射到主机的 80 端口。
```
docker run --rm -d --name web -p 80:80 nginx
```

如果宿主机的IP为10.10.101.105，外界只需访问10.10.101.105:80 就可以访问到容器中的服务。
```
curl -v http://10.10.101.105:80
```
查看默认的网络
```
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "aeeabedfaa07ae4d06d0dad4ede4126a93e0efd9a2a5f0034551665aa2744976",
        "Created": "2019-01-29T03:25:10.046595598Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "1a30155cb661730fe5733a20ca8f692da0ce9edae1932b902399ed31e4b42cbd": {
                "Name": "web",
                "EndpointID": "2d2009eb67cd56b9a40bea4d43be87d45fd61994a85e27fd0546494d065cfd3b",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

2. 自行创建网络

这种情况与使用默认网络基本类似，只是通过命令创建自有的网络和网桥来实现通信，这样就可以自己规划网络拓扑。在建立开发环境时，这是很常用的一种方式。laradock 即采用这种方式。
* 试验
  创建一个名为 `web-net` 的自定义网络，使用bridge网络驱动：
```
docker network create --driver bridge web-net
```

查看一下已经创建的网络列表，可以看到除了docker自行创建的 `bridge` 网络，还有刚创建的 `web-net` ：
```
docker network ls
```

查看 `web-net` 网络的详细信息：
```
$ docker network inspect web-net
[
    {
        "Name": "web-net",
        "Id": "79be842fea854d32708498bb01bf67e3b4967ffe32405493be2f1c6424eb4752",
        "Created": "2019-01-29T06:57:23.58969899Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

创建两个容器，使用 `web-net` 网络：
```
docker run --rm -d --name web1 --network web-net nginx
docker run --rm -d --name web2 --network web-net nginx
```

再次查看 `web-net` 网络的详细信息：
```
$ docker network inspect web-net
[
    {
        "Name": "web-net",
        "Id": "79be842fea854d32708498bb01bf67e3b4967ffe32405493be2f1c6424eb4752",
        "Created": "2019-01-29T06:57:23.58969899Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "614b91e2000356945a1aba29cfa4dacad04f0bf254972c78a54aacd3663079ca": {
                "Name": "web1",
                "EndpointID": "d0254e60f67f6d5eb155731bb028c40a2b723b0ef2aacfbe641b96dd67ca5a75",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "6aa947097ca8b6189c7a057b545ff718a82b15df4212cad6ee1ce3f2031e4bb5": {
                "Name": "web2",
                "EndpointID": "9be085bf7f3d075f4224b285d7a3c4c382b516c22698a151c1049ea9298ee567",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

可见容器 `web1` 和 `web2` 已经加入了此网络，并且分配了各自的网络IP，并使用同一个网络和网关。

#### bridge模式下默认网络和自建网络的差别
1. 提供更好的网络隔离和更灵活的拓扑

这一点不言而喻，只是需了解：各容器的全部端口，对当前网络（不论是默认还是自建）内的其他容器完全开放，而对宿主机和外界都不开放，除非设置端口映射。

2. 自建网络自动提供了各容器名称的DNS解析

默认网络下，各容器只能通过IP访问，除非显式设置 [--link 选项](https://docs.docker.com/network/links/)
而在自建网络中，默认就可以在某个容器中通过容器的名称来访问其他容器。

3. 容器可以动态的（无需重启容器）加入自建网络或移除（通过 `docker network connect` 或 `docker network disconnect` ，而如果要加入或移除默认网络，需要重建容器。

4. 每个自建网络拥有自己的可配置的网桥，可以进行更灵活的网桥参数配置。

5. 在默认网络并通过 `--link` 选项连接的容器可以共享环境变量，但在自建网络中不行。

### Docker实现
Docker的网络模式实际上是基于网络驱动来实现的，要了解网络驱动，需要先了解Docker的网络模型架构。
Docker的网络架构基于称为 **容器网络模型 Container Networking Model (CNM)** 的一组接口来实现：

![图片描述](/tfl/captures/2019-02/tapd_64812569_base64_1550485228_49.png)

图中以Docker Engine为界，上半部分（高层网络设施）是下半部分（驱动）的处理实例。

CNM与网络驱动的结构和关系：

![图片描述](/tfl/captures/2019-02/tapd_64812569_base64_1550485279_69.png)

另外还需了解的是：Docker 使用了 Linux 的 [Namespaces](https://coolshell.cn/articles/17010.html) 技术来进行资源隔离，如 PID Namespace 隔离进程，Mount Namespace 隔离文件系统，Network Namespace 隔离网络等。一个 Network Namespace 提供了一份独立的网络环境，包括网卡、路由、Iptable 规则等都与其他的 Network Namespace 隔离。

1. Host驱动
  在Host驱动模式下，docker 不会为容器创建单独的网络 namespace，而是共享主机的 network namespace，也就是说：容器可以直接访问主机上所有的网络信息。

![图片描述](/tfl/captures/2019-02/tapd_64812569_base64_1550485300_42.png)

1. Bridge驱动（默认网络）
  在Bridge驱动模式下，如果不自行建立网络（ `docker network create ...` ），会直接使用docker自建的默认网络。docker会在主机上创建一个名为 docker0 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。

![图片描述](/tfl/captures/2019-02/tapd_64812569_base64_1550485329_29.png)

Docker 完成以上网络配置的过程大致是这样的：
在主机上创建一对虚拟网卡 veth pair 设备。veth 设备总是成对出现的，它们组成了一个数据的通道，数据从一个设备进入，就会从另一个设备出来。因此，veth 设备常用来连接两个网络设备。Docker 将 veth pair 设备的一端放在新创建的容器中，并命名为 eth0。另一端放在主机中，以 veth65f9 这样类似的名字命名，并将这个网络设备加入到 docker0 网桥中，可以通过 brctl show 命令查看。

![图片描述](/tfl/captures/2019-02/tapd_64812569_base64_1550485347_11.png)

从 docker0 子网中分配一个 IP 给容器使用，并设置 docker0 的 IP 地址为容器的默认网关。


3. Bridge驱动（自建网络）

![图片描述](/tfl/captures/2019-02/tapd_64812569_base64_1550485354_79.png)

与默认网络的区别在于，自行创建了网桥和一个或多个子网。

4. Overlay、MACVLAN、None从略，如需了解请查阅 [官方文档](https://success.docker.com/article/networking)

一些细节说明：
1. host/bridge模式分别基于Host/Bridge驱动实现，这很好理解。那么container模式呢？

实际上container模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，因此可以认为是使用“别人”的驱动来实现。

2. 在bridge模式下，docker通过Iptable来实现容器对外是不可见。docker是如何实现的？

通过宿主机的 iptables 的 **SNAT** 转换。
查看包含bridge模式容器的宿主机上的 iptables 规则，可以看到这么一条规则
```
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

这条规则会将源地址为 172.17.0.0/16 的包（也就是从 Docker 容器产生的包），并且不是从 docker0 网卡发出的，进行源地址转换，转换成主机网卡的地址。
举例说明：假设主机有一块网卡为 eth0，IP 地址为 10.10.101.105/24，网关为 10.10.101.254。从主机上一个 IP 为 172.17.0.1/16 的容器中 ping 百度（180.76.3.151）。IP 包首先从容器发往自己的默认网关 docker0，包到达 docker0 后，也就到达了主机上。然后会查询主机的路由表，发现包应该从主机的 eth0 发往主机的网关 10.10.105.254/24。接着包会转发给 eth0，并从 eth0 发出去（主机的 ip_forward 转发应该已经打开）。这时候，上面的 Iptable 规则就会起作用，对包做 SNAT 转换，将源地址换为 eth0 的地址。这样在外界看来，这个包就是从 10.10.101.105 上发出来的

3. docker如何实现端口映射？

通过宿主机的 iptables 的 **DNAT** 转换。
在进行端口映射之后，查看宿主机的 iptables 规则的变化，发现多了这样一条规则：
```
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.17.0.5:80
```
此条规则就是对主机 eth0 收到的目的端口为 80 的 tcp 流量进行 DNAT 转换，将流量发往 172.17.0.5:80，也就是我们上面创建的 Docker 容器。所以，外界只需访问 10.10.101.105:80 就可以访问到容器中得服务。

### 常见问题
1. 容器如何连接和使用宿主机上的服务？

从上面的内容可以看出，如果可以的话，使用 `Host` 网络模式，是最方便的，容器里随便怎么连宿主机都毫无成本。
其实，即使是 `bridge` 模式，容器本来也是连接到宿主机的，唯一的一点点障碍，仅仅是宿主机的IP可能是动态的，同时也没有天然的DNS名对应宿主机（其实在Windows、MAC环境有这样的DNS名称，有需要可以查看 host.docker.internal [for windows](https://docs.docker.com/docker-for-windows/networking/#i-cannot-ping-my-containers) [for mac](https://docs.docker.com/docker-for-mac/networking/#i-cannot-ping-my-containers)）。
通过以下方式之一可以在 `bridge` 模式 找到宿主机的IP：

* 如果使用默认网络，在宿主机上执行
```
$ ip addr show docker0 | grep inet
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
    inet6 fe80::42:62ff:fefa:e57c/64 scope link
```
其中 `172.17.0.1` 便是默认网络在宿主机上的IP

* 如果使用自建网络，在宿主机上执行
```
$ docker network inspect {你的自建网络名称} | grep Gateway
                    "Gateway": "172.19.0.1"
```
其中 `172.19.0.1` 便是默认网络在宿主机上的IP。
其实这种方式也适用于在默认网络情况，将网络名称改为 `bridge` 即可。

* 在容器中更加简单，执行 `ip route show` 或
```
$ hostip=$(ip route show | awk '/default/ {print $3}')
$ echo $hostip
```

* 通过 `ifconfig` 或 `ip addr` 获取到宿主机的eth0或外网IP，也是可以的，只是略微有一点性能损失。例如
```
hostip=`ip -4 addr show scope global dev eth0 | grep inet | awk '{print \$2}' | cut -d / -f 1`
$ echo $hostip
```

获取到IP之后，还需注意：
a. 宿主机需要允许被连接，一般情况应该是可以的，如果不行需要设置 `iptables -A INPUT -i docker0 -j ACCEPT` 。
b. 宿主机上的服务应该监听在 0.0.0.0 或 * （即 `INADDR_ANY`），可以通过 `lsof -i | grep 端口号XXXX` 查看。

另外，为了在容器中更方便的连接宿主机，可以通过传入Host的方式来自行设置一个DNS名称，例如：
```
$ HOSTIP=`ip -4 addr show scope global dev eth0 | grep inet | awk '{print \$2}' | cut -d / -f 1`
$ docker run  --add-host=docker:${HOSTIP} --rm -it debian
```
在 `docker-compose` 中则可以通过 `extra_hosts` 来达到同样的效果，laradock就是通过这样的方式（需要在 .env 中指定 `DOCKER_HOST_IP` 变量为宿主机IP。

2. 一个承载了多个服务API的容器，每个API服务对应不同域名，如果让这些服务更加友好地相互访问？
  考虑多个项目共用一个laradock场景，nginx服承载了多个服务API，并对外提供统一的服务端口。而各个服务API的域名和nginx配置设计上是不同的，并存在相互依赖关系。某个服务想使用另一个服务的API时，要求使用默认的主机名域名（nginx）可能造成API路由冲突，各自nginx配置也不方便。如何做到对各个不同域名API的访问都指向同一个nginx容器？
  有多种办法解决这个问题，但最简单的，是通过 `--net-alias` 参数，或者是 docker-compose 的  `networks:网络XXX:aliases:`，例如在 laradock 的 `docker-compose.yml` 中有：
```
    nginx:
      ...
      networks:
        frontend:
        backend:
          aliases:
            - account-system.dd01.test
            - account-base-service.dd01.test
            - member-notification-service.dd01.test
            - points-core-system.dd01.test
            ...
```
这样就可以通过这里所列出的域名来访问各个服务，由nginx配置来根据域名（ `server_name`配置 ）分别对应到具体的服务项目。

### 参考资料
<https://docs.docker.com/network/>
<https://success.docker.com/article/networking> (深入了解推荐)
<https://docs.docker.com/engine/reference/run/#network-settings>
<https://www.infoq.cn/article/docker-network-and-pipework-open-source-explanation-practice>
<http://cizixs.com/2016/06/01/docker-default-network/>
<https://stackoverflow.com/questions/31324981/how-to-access-host-port-from-docker-container>