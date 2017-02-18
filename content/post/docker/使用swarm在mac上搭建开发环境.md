+++
author = "asdfsx"
date = "2017-02-18T10:12:39+08:00"
title = "使用swarm在mac上搭建开发环境"
keywords = [
  "docker",
  "swarm",
  "golang",
  "kafka",
]
description = "使用swarm在mac上搭建开发环境"
type = "post"
tags = [
  "docker",
  "swarm",
  "golang",
  "kafka",
]
topics = [
  "topic 1",
]
draft = true

+++

# 源起

docker并不是新技术。作为 golang 社区内的旗舰级的项目，从诞生之初就吸引了很多人的瞩目，甚至可以说是获得了所有 golang 社区的人多瞩目吧。同时对推广 golang 也起了很大的作用。作为从一直关注着 golang 的人，大概2年前就开始尝试使用它了。不过由于其依赖 linux 的多项特性，导致在 mac 上使用必须要借助虚拟机，体验总是要差一点。最近经人提醒发现新的 docker for mac 可以不借助虚拟机就可以在 mac 上提供与 linux 上同样的体验，大喜！遂卸载虚拟机，打算用新软件搭个测试环境玩耍一下。

# 安装

超级简单，照着说明来就好。顺便装上了 kitematic。这个确实也非常好用。

# 目标

将一套以前用 python 实现的 kafka producer程序，移植到 golang 上。测试新程序的异常情况处理（特指连接失败的处理）。
然后就被坑掉了。

# 坑1 网络的问题

最初只是想通过 docker 启动一个单点的 kafka，程序通过本地网络直接访问就可以了。通过之前安装的 kitematic 从 hub.docker.com 上下载了 spotify/kafka 镜像，并直接启动容器。通过kitematic连接到容器内，各项命令执行正常。但是执行程序的时候却发现，总是第一次连接成功后，后边的所有连接都是失败的。通过日志发现程序在第一次成功连接 kafka 之后，连接地址发生了变更，变成了容器内部的地址（直观的现象就是，端口从 kitematic 随机生成的 12345 变成了 9092）。然后又尝试了手动用 --net=host 方式启动，结果彻底连不上了。

在对 github.com/shopify/sarama 简单的分析以后，发现它的连接过程是这样的：

* 首次连接，根据配置信息连接服务器  
* 连接成功后，从服务器获取broker信息
* 根据获得到的broker信息，连接其余的broker

问题就出现在最后一步上，服务器返回的broker信息中，网络地址都是容器内部的网络，没有办法从容器网络外部直接连接的。

通过在github上查 issue， google 上查资料，有了以下不负责任的猜想：

> 目前的 docker for mac 使用了自己搞的 github.com/docker/hyperkit 来实现轻量级的虚拟化。从项目的说明可以看到，其基于 github.com/mist64/xhyve，借助了 mac os 自身的 hypervisor.framework 来实现虚拟化。在 developer.apple.com 上可以看到相关的说明和 api refrence。它是 OS X v10.10 新增的一个功能，从 api 上来看，可以实现对 cpu 和 mem 的简单控制
，并没有提供对网络的控制。网络这块是 xhyve 自己通过 virto-net 实现的。这可能也是导致网络这块体验比较差的原因吧。完善网络还需要花不少时间。

测试还是要继续，从外部连接容器网络有问题，那就把测试程序也放到容器环境中去。

__上SWARM吧，骚年！__

# 用SWARM解决网络问题
业界已有很多容器集群方案：kubernetes、mesos、swarm等。各有优势，选择swarm，主要原因还是方便，直接与docker集成。顺便吐槽，容器的江湖也是乱啊！

* 下载测试用到的镜像

  ```
  docker pull nginx
  docker pull spotify/kafka:latest
  docker pull prom/busybox:latest
  ```

* 在本地创建swarm

  ```
  docker swarm init
  ```

* 创建 overlay 网络

  ```
  docker network  create \
    --driver overlay \
    --subnet 10.0.9.0/24 \
    --opt encrypted \
    my-network
  ```

* 简单测试一下 overlay 网络，以及swarm中的服务发现

  * 启动一个 nginx 服务，并挂到 overlay 网络上

    ```
    docker service create \
      --replicas 1 \
      --name my-web \
      --network my-network \
      nginx
    ```

  * 检查服务

    ```
    docker service ps my-web
    ```

  * 检查网络

    ```
    docker network inspect my-network
    ```

  * 使用busybox检查服务发现，其实就是网络中的dns

    ```
    docker service create \
      --name dnstest \
      --network my-network \
      busybox \
      sleep 3000
  
    docker service ps dnstest

    docker exec -it dnstest.1.k8pjka8c9ikrpeje6spm6z8b5 sh

    nslookup my-web
    ```

* 启动 kafka 服务

  ```
  docker service create \
    --name kafka \
    --network my-network \
    spotify/kafka:latest
  ```
  
* 测试 kafka
  用测试 my-web 的方法测试 kafka

* 关闭服务

  ```
  docker service rm my-web
  docker service rm kafka
  ```

# 测试程序打包
golang 支持交叉编译，也就是说我在 mac 上就可以直接编译出 linux 上可以使用的可执行程序。这真是极大的方便了开发人员。只需要`GOOS=linux GOARCH=amd64 go build ${LDFLAGS} ${PACKAGE}`就可以获得需要的可执行程序。然后简单的打包到镜像中就可以了。然后如愿以偿的掉入...

# 坑2 时区的问题
鉴于 golang 的可执行文件只是依赖 glibc，打包时选择了官方busybox作为基础镜像，好处就是小！小到连时区的支持都缩减到只有 UTC 一个，然后程序执行的时候就悲剧了。找到了最简单的一个解决办法是，设置环境变量`TZ`，设为`CST-8`表示中国标准时间，设为`JST-9`表示日本标准时间。这个对于 date 命令很好使，但是对于编译好的 golang 程序顶个球用。只能选择另外一个办法，换一个带时区支持的 `prom/busybox` 镜像，然后在Dockerfile中增加一个命令

```
RUN mv /etc/localtime /etc/localtime.bak && \
    ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

时区问题搞定，测试顺利进行