### 1、概述

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中,然后发布到任何流行的Linux或Windows操作系统的机器上,也可以实现虚拟化,容器是完全使用沙箱机制,相互之间不会有任何接口。

一个完整的Docker有以下几个部分组成：

1. DockerClient客户端
2. Docker Daemon守护进程
3. Docker Image镜像
4. DockerContainer容器 

![img](pics\u=1293437859,864068487&fm=26&fmt=auto)

### 2、常用操作

常用操作访问：https://www.runoob.com/docker/docker-tutorial.html



### Zookeeper集群搭建

因为一个一个地启动 ZK 太麻烦了, 所以为了方便起见, 直接使用 docker-compose 来启动 ZK 集群.
首先创建一个名为 **docker-compose.yml** 的文件, 其内容如下:

```yml
version: '3.7'

services:
  zoo1:
    image: zookeeper
    hostname: zoo1
    container_name: zoo1
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo2:
    image: zookeeper
    hostname: zoo2
    container_name: zoo2
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo3:
    image: zookeeper
    hostname: zoo3
    container_name: zoo3
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
```

这个配置文件会告诉 Docker 分别运行三个 zookeeper 镜像, 并分别将本地的 2181, 2182, 2183 端口绑定到对应的容器的2181端口上.
**ZOO_MY_ID** 和 **ZOO_SERVERS** 是搭建 ZK 集群需要设置的两个环境变量, 其中 **ZOO_MY_ID** 表示 ZK 服务的 id, 它是1-255 之间的整数, 必须在集群中唯一. **ZOO_SERVERS** 是ZK 集群的主机列表.

接着我们在 docker-compose.yml 当前目录下运行:

```shell
docker-compose up -d
```

此时Zookeeper集群即搭建完成且启动，可以查看容器角色或者客户端连接。

![image-20211110160936220](pics\image-20211110160936220.png)



