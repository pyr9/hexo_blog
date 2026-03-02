---
title: docker
date: 2023-04-18 14:20:45
tags:
categories: docker
---

# 1 docker解决了什么问题？

比如我们现在开发了一个应用程序比如淘宝App，程序员从头到尾搭建了一套环境开始写代码，比如安装JDK等，代码写完后需要交给测试同事去运行，这事测试同学也需要从头到尾搭建一套环境，但比如JDK版本和你安装的不同，导致无法在测试同学的机器上运行起来，找开发去看问题，此时开发也很无辜，“明明在我的机器上是可以跑起来的”。

在没有容器技术之前，我们可以采用搭建虚拟机，然后测试人员克隆一个同样的虚拟机。

但是虚拟机里面是有一套完整的操作系统的。单单运行起来都需要耗费很多资源。比如我们新装的系统，还没有装任何的软件，磁盘已经被占用了好几个G，导致我们无法部署更多的应用程序，此外，操作系统的启动也很缓慢。

因此出现了容器技术。容器和集装箱在概念上是很相似的。需要实现隔离，也就是多个应用程序在运行时互相不干扰。

容器技术只隔离应用程序运行时环境，但是共享同一个操作系统。这里的运行时环境指的是程序运行依赖的各种库以及配置。

因此，相比较虚拟机，容器更加的轻量且占用的资源更少。我们可以部署更多的应用程序，同时启动的速度也很快。

docker就是容器技术的一种实现。

# 2 docker是什么

docker是一个用Go语言实现的开源项目，可以让我们方便的创建和使用容器，docker将程序以及程序所有的依赖都打包到docker container，这样你的程序可以在任何环境都会有一致的表现。

这里程序运行的依赖也就是容器就好比集装箱，容器所处的操作系统环境就好比货船或港口，**程序的表现只和集装箱有关系(容器)，和集装箱放在哪个货船或者哪个港口(操作系统)没有关系**。

**docker的底层实现**

docker基于**Linux内核**提供这样几项功能实现的：

- **NameSpace**： 我们知道Linux中的PID、IPC、网络等资源是全局的，而NameSpace机制是一种资源隔离方案，在该机制下这些资源就不再是全局的了，而是属于某个特定的NameSpace，各个NameSpace下的资源互不干扰，这就使得每个NameSpace看上去就像一个独立的操作系统一样，但是只有NameSpace是不够。
- **Control groups**：虽然有了NameSpace技术可以实现资源隔离，但进程还是可以不受控的访问系统资源，比如CPU、内存、磁盘、网络等，为了控制容器中进程对资源的访问，Docker采用control groups技术(也就是cgroup)，有了cgroup就可以控制容器中进程对系统资源的消耗了，比如你可以限制某个容器使用内存的上限、可以在哪些CPU上运行等等。

# 3. **Docker容器和虚拟机的区别是什么？**

Docker容器在操作系统级别进行虚拟化，共享宿主机的内核。

虚拟机在硬件级别进行虚拟化，拥有独立的内核。

容器通常更轻量级、启动更快，资源占用更少。

# 4. **什么是Docker镜像？**

Docker镜像是一个轻量级、只读的模板，用于创建Docker容器。它包含运行容器所需的代码、库、环境变量和配置文件。

# 5. **Docker镜像和容器之间有什么区别？**

Docker镜像是静态的定义，类似于类；而容器是镜像的实例，类似于对象。

# 6. 什么是 DockerFile？

Dockerfile 是一个文本文件，其中包含我们需要运行以构建 Docker 映像的所有命令。Docker 使用 Dockerfile 中的指令自动构建镜像。我们可以`docker build`用来创建按顺序执行多个命令行指令的自动构建。

# 3 安装下载

## 1. 安装

```text
brew cask install docker
```

## 2. 修改镜像源

这里是网易的镜像源

```
 "registry-mirrors": [
    "http://hub-mirror.c.163.com"
  ]
```

![image-20230418145138450](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20230418145138450.png)



# 4 docker容器生命周期管理

![image-20230418150102287](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20230418150102287.png)

- 从镜像仓库中拉取或者更新指定镜像

  `docker pull nginx`

- 创建一个新的容器但不启动它

  `docker create nginx`

- 列出所有在运行的容器信息。

  `docker ps`

- 显示所有的容器，包括未运行的

  `docker ps -a`
  ![image-20230418151220544](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20230418151220544.png)

-  启动一个停止的容器

  `docker start d7e105501d1f`

- 登录容器，在运行的容器中执行命令

  ` docker exec -it d7e105501d1f /bin/bash`

- 暂停容器中所有的进程

  `docker pause d7e105501d1f`

- 恢复容器中所有的进程。

  `docker unpause d7e105501d1f`

- 停止一个运行中的容器

  `docker stop d7e105501d1f`

- 杀掉一个运行中的容器。

  `docker kill d7e105501d1f`

- 删除一个或多个容器

  `docker rm -f d7e105501d1f`

- 在后台启动一个容器

  `docker run -d redis`

- 启动一个映射端口的容器

  `docker run -p 80:80 -d nginx `

# 5 docker镜像构建

1. 创建并编辑Dockerfile

   ` vim Dockerfile`

```
#Owner by Pan Yu Rou
FROM redis
MAINTAINER pyr
RUN mkdir test01
RUN touch test02
COPY test3 .
ADD test4.tar.gz .
ENTRYPOINT ["/bin/sh"]
CMD ["-c","ls -l"]
```

2. 在Dockerfile所在路径下，运行`docker build -t your-image-name .`

```
➜  docker_dir docker build -t mysh .
[+] Building 0.5s (10/10) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                                                                                                                                              0.0s
 => => transferring dockerfile: 199B                                                                                                                                                                                                                                                              0.0s
 => [internal] load .dockerignore                                                                                                                                                                                                                                                                 0.0s
 => => transferring context: 2B                                                                                                                                                                                                                                                                   0.0s
 => [internal] load metadata for docker.io/library/redis:latest                                                                                                                                                                                                                                   0.0s
 => [internal] load build context                                                                                                                                                                                                                                                                 0.0s
 => => transferring context: 176B                                                                                                                                                                                                                                                                 0.0s
 => [1/5] FROM docker.io/library/redis                                                                                                                                                                                                                                                            0.0s
 => [2/5] RUN mkdir test01                                                                                                                                                                                                                                                                        0.2s
 => [3/5] RUN touch test02                                                                                                                                                                                                                                                                        0.1s
 => [4/5] COPY test3 .                                                                                                                                                                                                                                                                            0.0s
 => [5/5] ADD test4.tar.gz .                                                                                                                                                                                                                                                                      0.0s
 => exporting to image                                                                                                                                                                                                                                                                            0.0s
 => => exporting layers                                                                                                                                                                                                                                                                           0.0s
 => => writing image sha256:679e723aa319cdb9d1aeb923534c200b6ddb60eb9a87c8e7a3814145f716b3ff                                                                                                                                                                                                      0.0s
 => => naming to docker.io/library/mysh
```

3. 通过` docker run your-image-name `运行容器

可以看到test01，test02， test4 都已经出现在了容器目录中，当不指定参数命令运行时，默认运行CMD里配置的命令，否则执行参数中的命令

```
➜  docker_dir docker run mysh
total 8
drwxr-xr-x 2 root root    4096 Apr 18 08:14 test01
-rw-r--r-- 1 root root       0 Apr 18 08:14 test02
drwxr-xr-x 2  501 dialout 4096 Apr 18 08:11 test4
```

```
➜  docker_dir docker run mysh -c date
Tue Apr 18 08:16:51 UTC 2023
```

# 4 Dockerfile常用指令

- **FROM**：定制的镜像都是基于 某个 基础镜像，这里的FROM就是指定基础镜像。因此在Dockerfile中，FORM是必备指令，并且必须是第一条指令。
- **RUN**：用于执行后面跟着的命令行命令。

- **COPY**: 复制指令，从上下文目录中复制文件或者目录到容器里指定路径。

- **ADD**: ADD 指令和 COPY 的使用格类似（同样需求下，官方推荐使用 COPY）。功能也类似，不同之处如下：

  - ADD 的优点：在执行 <源文件> 为 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，会自动复制并解压到 <目标路径>。

  - ADD 的缺点：在不解压的前提下，无法复制 tar 压缩文件。会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。具体是否使用，可以根据是否需要自动解压来决定。

- **CMD**: 
  - 类似于 RUN 指令，用于运行程序，但二者运行的时间点不同: CMD 在docker run 时运行，RUN 是在 docker build。
  - CMD 指令指定的程序可被 docker run 命令行参数中指定要运行的程序所覆盖。
  - 如果 Dockerfile 中如果存在多个 CMD 指令，仅最后一个生效。

- **ENTRYPOINT**
  - 类似于 CMD 指令，但其不会被 docker run 的命令行参数指定的指令所覆盖，而且这些命令行参数会被当作参数送给 ENTRYPOINT 指令指定的程序。
  - 如果运行 docker run 时使用了 --entrypoint 选项，将覆盖 ENTRYPOINT 指令指定的程序。
  - 如果 Dockerfile 中如果存在多个 ENTRYPOINT 指令，仅最后一个生效。

  > **模式：ENTRYPOINT 固定主程序，CMD 提供默认参数**
  >
  > ```
  > FROM openjdk:8-jre
  > COPY app.jar /app.jar
  > ENTRYPOINT ["java", "-jar", "/app.jar"]
  > CMD ["--spring.profiles.active=prod"]  # 默认参数
  > ```
  
- **ENV**

  - 设置环境变量，定义了环境变量，那么在后续的指令中，就可以使用这个环境变量。

  - 格式：

    ```
    ENV <key> <value>
    ENV <key1>=<value1> <key2>=<value2>...
    ```

- **ARG**
  - 构建参数，与 ENV 作用一致。不过作用域不一样。ARG 设置的环境变量仅对 Dockerfile 内有效，也就是说只有 docker build 的过程中有效，构建好的镜像内不存在此环境变量。
  - 构建命令 docker build 中可以用 --build-arg <参数名>=<值> 来覆盖。
  - 格式：`ARG <参数名>[=<默认值>]`

- **EXPOSE**
  
  - 仅仅只是声明端口。

# 6 docker数据持久化

摆放数据的持久化卷有两种常见的使用方法：Bing mounts 和 Volume

## 1. Bind Mounts（狩猎模式）

Bind mounts 是由用户在主机上进行选择目录进行存储，方便指定特定文件或者目录

```
docker run -v <host_path:><container_path> -d 
```

```
➜  docker docker run -v $PWD/data:/data -d redis:3.2 redis-server
d7a6e3dccf100ea42bb1a8168dbf9ea27ed71b0b2837d19239f3eff8497fedd5
➜  docker docker ps
CONTAINER ID   IMAGE       COMMAND                  CREATED         STATUS         PORTS      NAMES
d7a6e3dccf10   redis:3.2   "docker-entrypoint.s…"   5 seconds ago   Up 4 seconds   6379/tcp   quizzical_swartz
2e3ce4212a25   redis       "docker-entrypoint.s…"   2 hours ago     Up 2 hours     6379/tcp   charming_ritchie
➜  docker docker exec -it d7a6e3dccf10 /bin/bash
root@d7a6e3dccf10:/data# touch redis-logs
root@d7a6e3dccf10:/data# exit
exit
➜  docker ls
data         dockerfile01 dockerfile02
➜  docker cd data
➜  data ls
redis-logs
```



## 2. Volumes（靶场模式）

而Volume则统一将数据放到主机的、var/lib/docker/volumes下，只能对应特定目录下的子目录，不支持文件，但是不容易乱，适合小白进行使用。两种主流volumn的创建方式：

- 隐式创建卷：`docker run -v <container_path> -d `

  ```
  ➜  docker docker run -v data2 -d redis:3.2 redis-server
  829ea54fbaeef20c7ea1f3f1e3a82d5dd7f7a73d099c74b758926a14b2c4f5ca
  ➜  docker docker ps
  CONTAINER ID   IMAGE       COMMAND                  CREATED              STATUS              PORTS      NAMES
  829ea54fbaee   redis:3.2   "docker-entrypoint.s…"   4 seconds ago        Up 3 seconds        6379/tcp   awesome_euclid
  d7a6e3dccf10   redis:3.2   "docker-entrypoint.s…"   About a minute ago   Up About a minute   6379/tcp   quizzical_swartz
  2e3ce4212a25   redis       "docker-entrypoint.s…"   2 hours ago          Up 2 hours          6379/tcp   charming_ritchie
  ➜  docker docker exec -it 829ea54fbaee /bin/bash
  root@829ea54fbaee:/data# touch redis-log2
  root@829ea54fbaee:/data# exit
  exit
  ```

  ```
  ➜  docker docker run -it --privileged --pid=host justincormack/nsenter1
  
  / # cd var/lib/docker/volumes/b4778654451afef91ac2dbf8e6ce7881c065404ebc749d1bc7ea50fa93122e3d/_data
  /var/lib/docker/volumes/b4778654451afef91ac2dbf8e6ce7881c065404ebc749d1bc7ea50fa93122e3d/_data # ls
  redis-log2
  ```

  

- 显式创建卷：(先创建卷，再创建容器绑定卷)        `docker volume create data3`

```
 ➜  docker docker volume create data3
 ➜  docker docker run -v data3:/data3 -d redis:3.2 redis-server
da981af699805bf18e87ed7d986e2e9c6c08220c67f111c136597c62d15f144c
```

```
➜  docker docker run -it --privileged --pid=host justincormack/nsenter1
/ #  ls -ld /var/lib/docker/volumes/data3
drwx-----x    3 root     root          4096 Apr 18 09:31 /var/lib/docker/volumes/data3
```



![image-20230418170422909](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20230418170422909.png)

# 7 Docker仓库

仓库（Repository）是集中存放镜像的地方。 总体来看Docker 仓库可以分为两大类：

- 公共仓库：最经典的公共仓库就是Docker Hub。当Docker装好后，其默认的镜像源头就是这个功能仓库。当运行Docker run等命令时系统会自动从Docker Hub下载现有镜像到本地。除了Docker Hub之外还有一些第三方的公共仓库，对外提供服务，比如Quay、阿里云等。
- 私有仓库：私有仓库是互联网企业的生产中心的实际操作中的主流选择。不采用公共仓库通常出于以下两个原因：
  - 公共仓库通常建于海外，下载速度受限，对于大量节点频繁更新的互联网应用场景不太友好。
  - 公共仓库位于外网，直接将生产中心的应用和外网相连，存在明显安全隐患。黑客等容易通过网络漏洞渗透到企业内网；并且公共仓库内的容器镜像未通过企业内部的静态文件扫描，无法保证镜像中所采用的运行时和依赖库的安全性。

# 8 修改Docker默认镜像和容器存储位置

[Docker](https://so.csdn.net/so/search?q=Docker&spm=1001.2101.3001.7020) 默认安装的情况下，会使用 `/var/lib/docker/` 目录作为存储目录，用以存放拉取的镜像和创建的容器等。不过由于此目录一般都位于系统盘，遇到系统盘比较小，而镜像和容器多了后就容易磁盘空间不足

- 查看当前[docker](https://so.csdn.net/so/search?q=docker&spm=1001.2101.3001.7020)的默认存储目录 `docker info` ,默认为`/var/lib/docker/` 

- 编辑 `/etc/docker/daemon.json` 文件

  `sudo vim /etc/docker/daemon.json `

  ```
  {
    "data-root": "/home/docker"
  }
  ```

- 重启docker, 并查看 `docker info`

