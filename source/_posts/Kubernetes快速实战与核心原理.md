---
title: Kubernetes核心原理
date: 2025-06-29 16:51:58
tags:
Categories: Kubernetes
---

# 1.  K8s简介

Kubernetes（通常简称为K8s）是一个开源的容器编排的工具，用于**自动化部署、扩展和管理容器化应用**。它提供了一组丰富的功能和工具，用于简化容器化应用程序的部署、伸缩和运维。

**为什么出现k8s?**

这要容器技术兴起开始说，为了解决“一次打包，到处运行”的问题，大家引入了docker,  随着项目越来越大，一个微服务应用可能由几十个甚至上百个容器组成，这时候就出现了新的问题。

1. 容器的管理问题： 启动、停止、重启、升级很多容器很麻烦
2. 如何自动重启失败的容器？如果某个服务挂了，怎么让它自动恢复？
3. 如何应对流量高峰？实现自动扩容？
4. 多个容器分布在不同机器上时，怎么通信，网络怎么配？
5. 怎么更实现更新版本时，用户不能感知到停机？

这些问题加在一起， 导致运维工作变得极其复杂。Google 看到了这个问题，开发了 Kubernetes，目标就是：提供一个统一的平台，用来自动化地部署、扩展和管理容器化应用。

你可以把他想象成一个容器管家：

- 你告诉它：“我要运行一个服务，副本数是3个。”
- 它就会自动帮你启动3个容器，并确保它们一直正常运行。
- 如果其中一个容器坏了，它会自动重新启动一个新的。
- 如果访问量变多了，它可以自动增加副本数量。
- 所有这些都不需要你手动操作。

# 2. **K8s核心特性**

- 服务发现与负载均衡：无需修改你的应用程序即可使用陌生的服务发现机制。
- 自我修复：重新启动失败的容器，在节点死亡时替换并重新调度容器，杀死不响应用户定义的健康检查的容器。
- 自动化上线和回滚：Kubernetes会分步骤地将针对应用或其配置的更改上线，同时监视应用程序运行状况以确保你不会同时终止所有实例。
- 水平扩缩：使用一个简单的命令、一个UI或基于CPU使用情况自动对应用程序进行扩缩。
- 存储编排：自动挂载所选存储系统，包括本地存储。
- Secret和配置管理：部署更新Secrets和应用程序的配置时不必重新构建容器镜像，且不必将软件堆栈配置中的秘密信息暴露出来。
- 批量执行：除了服务之外，Kubernetes还可以管理你的批处理和CI工作负载，在期望时替换掉失效的容器。
- 自动装箱：根据资源需求和其他约束自动放置容器，同时避免影响可用性。

使用场景：快速部署应用、快速扩展应用、无缝对接新的应用功能、节省资源，优化硬件资源的使用；

# 3. K8s核心概念

## 1 Pod

### 1. 定义

Pod 是 Kubernetes 中最小的可部署单元。一个 Pod 可以包含一个或多个容器（通常是一个），所有的应用，服务最终都是运行在pod上。

**特点**

- 有自己独立的 IP（Pod IP）。pod里的容器可以共享网络，共享IP
- 生命周期短暂：一旦挂了，K8s 会新建一个 Pod，但 IP 会变。
- 通常不直接对外暴露，而是通过 Service 访问。

<img src="https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250713152823707.png" alt="image-20250713152823707" style="zoom:50%;" />

### 2. pod的生命周期

Pending（挂起）：Pod 已被接受，但至少有一个容器还没启动完成（镜像在拉取、调度未完成等）。

Running（运行中）：Pod 已绑定到节点，所有容器都已创建，至少一个容器处于运行状态。

Succeed（成功）：所有容器都正常退出，且不会再重启（常见于一次性任务 Job）。

Failed（失败）：至少有一个容器非正常退出（非 0 状态码或被系统终止）。

Unknown（未知）：控制平面无法获取 Pod 状态（网络分区等异常情况）。

### 3. pod的重启策略

通过命令`kubectl explain pod.spec`查看pod的重启策略。

- Always：但凡pod对象终止就重启，此为默认策略;
- Never： 不重启
- OnFailure：仅在pod对象出现错误时才重启;

<img src="https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250713174239215.png" alt="image-20250713174239215" style="zoom:50%;" />

> docker 的重启策略是：
>
> - **no**：默认策略，容器退出后不会自动重启。适用于一次性任务或测试场景。
>
> - **always**：无论退出原因，容器都会自动重启。即使 Docker 守护进程重启，容器也会随之启动。
>
> - **on-failure[:n]**：仅在容器异常退出（非 0 状态码）时重启。可选参数 *n* 指定最大重启次数。
> - **unless-stopped**：与 *always* 类似，但如果容器被手动停止（如 *docker stop*），即使 Docker 守护进程重启，容器也不会自动启动。

## 2 Deployment

Deployment 是一种控制器对象，用来声明“我要多少个 Pod、用什么镜像、怎么升级”。

**主要职责：**

1. 创建 Pod：按模板生成 Pod。

2. 维持副本数：比如声明 3 个副本，挂了一个会自动补一个新 Pod。

3. 滚动升级 / 回滚：镜像更新时，逐步替换旧 Pod，支持回滚到上一版本。
4. 扩缩容：改一下副本数，自动增加或减少 Pod。

## 3 service

Service为一组 Pod 提供**稳定的网络访问方式**（ClusterIP / NodePort / LoadBalancer 等）。

**解决的问题**：

- Pod 会挂、会重建，IP 会变。
- Service 通过一个固定的 DNS 名称和 ClusterIP，把流量转发到当前这批 Pod 上。

**常见类型**：

- ClusterIP（默认）：集群内部访问（默认）。、
- NodePort：在每个节点开一个端口，外部可以访问。
- LoadBalancer：云厂商提供的负载均衡入口。
- ExternalName：通过返回带有该名称的CNAME记录，使用任意名称（由spec中的externalName指定）公开Service。不使用代理。


> 为什么Pod已经有了IP还需要service再包一层呢？
>
> - 当Pod出现意外挂掉时，这个服务无法访问，他肯定会去其他位置新启动一个服务，通过service的IP依旧可以找到这个新建的服务

> Deployment 管 Pod 的“生老病死和版本”，Pod 是真正干活的“小盒子”，Service 是给这些小盒子挂一个“固定门牌号”，让别人能稳定找到它们。

## 4 **Volume**

- Volume（卷）用来给 **Pod 里的容器提供持久化存储**。

- 容器本身是临时的：容器删了，里面的文件就没了；Volume 可以让数据**独立于容器生命周期**保留下来。

## 5 **Namespace**

Namespace 是 Kubernetes 的**逻辑隔离机制**，把集群资源（Pod、Service、Deployment、Volume 等）分成不同的“虚拟集群”。

**作用：**

- **多环境隔离** 
  - `dev`命名空间：开发环境资源
  -  `test`命名空间：测试环境资源 
  - `prod`命名空间：生产环境资源

- **多团队 / 多租户隔离**
  -  A 团队用 `team-a`命名空间 
  -  彼此看得到但不直接干扰，RBAC 可以按 Namespace 授权。

- **资源配额管理** 可以给每个 Namespace 设置 CPU、内存、Pod 数量上限，防止某团队把集群资源吃光。

**默认 Namespace**

- default：不指定 Namespace 时默认放这里。
- kube-system：K8s 系统组件自己用的。
- kube-public：公开可读的一些配置。

# 4. **K8s的组件有哪些？作用是什么？**

K8s需要一个集群来运行和管理容器化应用程序。Kubernetes 集群由多个计算节点组成，其中包括主节点（Master Node）和从节点（Worker Node）。

- 主节点负责管理整个集群的，安装了K8s的核心组件。

- 从节点是容器应用真正运行的地方。
  - 从节点（Worker Node）上的容器化环境一般是通过 Docker 运行的。
  - 每个 Worker Node 上可以运行多个容器， 其中每个容器都可以托管一个或多个 Pod。

- master节点包含的组件有：kube-api-server、kube-controller-manager、kube-scheduler、etcd
- node节点包含的组件有：kubelet、kube-proxy、container-runtime

![image-20250701214938688](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250701214938688.png)

## 1  **Master Node**的组件

### 1. **kube API Server**

- **K8S 的请求入口服务**。

- 它是k8s集群管理的统一访问入口，接收 K8S 所有请求，实现了认证、授权和准入控制等安全功能；
- API Server 根据用户的具体请求，去通知其他组件干活。
- api-server是其他组件之间的数据交互和通信的枢纽，其他组件彼此之间并不会直接通信，其他组件对资源对象的增、删、改、查和监听操作都是交由api-server处理后，api-server再提交给etcd数据库做持久化存储，只有api-server才能直接操作etcd数据库，其他组件都不能直接操作etcd数据库，其他组件都是通过api-server间接的读取，写入数据到etcd。

### 2. **kube Controller Manager**

- **k8s集群内部的管理控制中心**， 他是Kubernetes的大脑，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态。
- kube-controller-manager由一系列的控制器组成，像Replication Controller控制副本，Node Controller节点控制，Deployment Controller管理deployment等等
- Controller 负责监控和调整在 Worker Node 上部署的服务的状态，比如用户要求 A 服务部署 2 个副本，那么当其中一个服务挂了的时候，Controller 会马上调整，让 Scheduler 再选择一个 Worker Node 重新部署服务。

### 3. **kube Scheduler**

- **K8S 负责集群资源调度的调度器**。
- 其作用是将待调度的pod通过一系列复杂的调度算法计算出最合适的node节点，然后将pod绑定到目标节点上。当用户要部署服务时，Scheduler 会选择最合适的Worker Node（服务器）来部署。

### 4. **etcd**

- **K8S 的存储服务， 是一个分布式的键值对存储数据库**。
- etcd 存储了 K8S 的关键配置和用户配置，像k8s本身的节点信息，组件信息，还有通过kubernetes运行的pod，deployment，service等等, 都需要持久化到etc。
- K8S 中仅 API Server 才具备读写权限，其他组件必须通过 API Server 的接口才能读写数据。
- 生产环境中为了保证数据中心的高可用和数据的一致性，一般会部署最少三个节点。

## 2  **Worker Node**的组件

### 1.  **Kubelet**

- **Worker Node 的监视器，以及与 Master Node 的通讯器**。
- 每个工作节点上都运行一个kubelet服务进程，它会定期向 Master Node 汇报自己 Node 上运行的服务的状态，并接收并执行master发来的指令，管理Pod及Pod中的容器。比如创建、更新、终止pod等任务，kubelet 即通过控制docker来创建、更新、销毁容器。

### 2. **Kube-Proxy**

- **K8S 的网络代理**。Kube-Proxy 负责 Node 在 K8S 的网络通讯、以及对外部网络流量的负载均衡。
- 核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上

### 3. Container Runtime

- **Worker Node 的运行环境**。即安装了容器化所需的软件环境确保容器化程序能够跑起来，比如 Docker Engine运行环境。

## 3 各组件协同工作的过程

以用K8S部署Nginx的过程为例, **我们在master节点执行一条命令要master部署一个nginx应用**

```shell
kubectl create deployment nginx --image=nginx
```

1. 这条命令首先发到master节点的网关api server，这是matser的唯一入口
2. kube api server将命令请求交给kube controller mannager进行控制
3. kube controller mannager 进行应用部署解析，会生成一次部署信息，并通过api server将信息存入etcd存储中
4. kube scheduler调度器通过api server从etcd存储中，拿到要部署的应用，开始调度看哪个节点有资源适合部署,   scheduler把计算出来的调度信息通过api server再放到etcd中
5. 每一个node节点的监控组件kubelet，随时和master保持联系（给api-server发送请求不断获取最新数据），拿到master节点存储在etcd中的部署信息
6. 假设node2的kubelet拿到部署信息，显示他自己节点要部署某某应用. kubelet就自己run一个应用在当前机器上，并随时给master汇报当前应用的状态信息,  node和master也是通过master的api-server组件联系的

![image-20250713122653667](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250713122653667.png)

# 5. kubelet的功能、作用是什么？

kubelet是部署在每个node节点上的，它主要有4个功能：

## 1. 节点管理

kubelet启动时会向api-server进行注册，然后会定时的向api-server汇报本节点信息状态，资源使用状态等，这样master就能够知道node节点的资源剩余，节点是否失联等等相关的信息了。master知道了整个集群所有节点的资源情况，这对于 pod 的调度和正常运行至关重要。
## 2. pod管理

kubelet负责维护node节点上pod的生命周期，当kubelet监听到master的下发到自己节点的任务时，比如要创建、更新、删除一个pod，kubelet 就会通过CRI（容器运行时接口）插件来调用不同的容器运行时来创建、更新、删除容器；

## 3. 容器健康检查

pod中可以定义启动探针、存活探针、就绪探针等3种，我们最常用的就是存活探针、就绪探针，kubelet 会定期调用容器中的探针来检测容器是否存活，是否就绪，如果是存活探针，则会根据探测结果对检查失败的容器进行相应的重启策略。

## 4. 资源监控

在node节点上部署Metrics Server用于监控node节点、pod的CPU、内存、文件系统、网络使用等资源使用情况，而kubelet则通过Metrics Server获取所在节点及容器的上的数据。

