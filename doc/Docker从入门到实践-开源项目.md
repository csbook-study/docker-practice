# Docker 从入门到实践-开源项目

## etcd

`etcd` 是 `CoreOS` 团队发起的一个管理配置信息和服务发现（`Service Discovery`）的项目。

`etcd` 是 `CoreOS` 团队于 2013 年 6 月发起的开源项目，它的目标是构建一个高可用的分布式键值（`key-value`）数据库，基于 `Go` 语言实现。我们知道，在分布式系统中，各种服务的配置信息的管理分享，服务的发现是一个很基本同时也是很重要的问题。`CoreOS` 项目就希望基于 `etcd` 来解决这一问题。

`etcd` 目前在 [github.com/etcd-io/etcd](https://github.com/etcd-io/etcd) 进行维护。

受到 [Apache ZooKeeper](https://zookeeper.apache.org/) 项目和 [doozer](https://github.com/ha/doozerd) 项目的启发，`etcd` 在设计的时候重点考虑了下面四个要素：

- 简单：具有定义良好、面向用户的 `API` ([gRPC](https://github.com/grpc/grpc))
- 安全：支持 `HTTPS` 方式的访问
- 快速：支持并发 `10 k/s` 的写操作
- 可靠：支持分布式结构，基于 `Raft` 的一致性算法

*Apache ZooKeeper 是一套知名的分布式系统中进行同步和一致性管理的工具。*

*doozer 是一个一致性分布式数据库。*

*[Raft](https://raft.github.io/) 是一套通过选举主节点来实现分布式系统一致性的算法，相比于大名鼎鼎的 Paxos 算法，它的过程更容易被人理解，由 Stanford 大学的 Diego Ongaro 和 John Ousterhout 提出。更多细节可以参考 [raftconsensus.github.io](http://raftconsensus.github.io/)。*

一般情况下，用户使用 `etcd` 可以在多个节点上启动多个实例，并添加它们为一个集群。同一个集群中的 `etcd` 实例将会保持彼此信息的一致性。

### etcd 集群

使用 `docker compose up` 启动集群之后使用 `docker exec` 命令登录到任一节点测试 `etcd` 集群。

```bash
$ docker exec -it etcd-node1-1 sh
# etcdctl member list
daf3fd52e3583ff, started, node3, http://172.16.238.102:2380, http://172.16.238.102:2379, false
422a74f03b622fef, started, node1, http://172.16.238.100:2380, http://172.16.238.100:2379, false
ed635d2a2dbef43d, started, node2, http://172.16.238.101:2380, http://172.16.238.101:2379, false
```

### edcdctl 使用

`etcdctl` 是一个命令行客户端，它能提供一些简洁的命令，供用户直接跟 `etcd` 服务打交道，而无需基于 `HTTP API` 方式。这在某些情况下将很方便，例如用户对服务进行测试或者手动修改数据库内容。我们也推荐在刚接触 `etcd` 时通过 `etcdctl` 命令来熟悉相关的操作，这些操作跟 `HTTP API` 实际上是对应的。

`etcd` 项目二进制发行包中已经包含了 `etcdctl` 工具，没有的话，可以从 [github.com/etcd-io/etcd/releases](https://github.com/etcd-io/etcd/releases) 下载。

#### 数据库操作

数据库操作围绕对键值和目录的 CRUD （符合 REST 风格的一套操作：Create）完整生命周期的管理。

etcd 在键的组织上采用了层次化的空间结构（类似于文件系统中目录的概念），用户指定的键可以为单独的名字，如 `testkey`，此时实际上放在根目录 `/` 下面，也可以为指定目录结构，如 `cluster1/node2/testkey`，则将创建相应的目录结构。

> 注：CRUD 即 Create, Read, Update, Delete，是符合 REST 风格的一套 API 操作。

put

```bash
$ etcdctl put /testdir/testkey "Hello world"
OK
```

get

获取指定键的值。例如

```bash
$ etcdctl put testkey hello
OK
$ etcdctl get testkey
testkey
hello
```

支持的选项为：

`--sort` 对结果进行排序

`--consistent` 将请求发给主节点，保证获取内容的一致性

del

删除某个键值。例如

```bash
$ etcdctl del testkey
1
```

#### 非数据库操作

watch

监测一个键值的变化，一旦键值发生更新，就会输出最新的值。

例如，用户更新 `testkey` 键值为 `Hello world`。

```bash
$ etcdctl watch testkey
PUT
testkey
2
```

member

通过 `list`、`add`、`update`、`remove` 命令列出、添加、更新、删除 etcd 实例到 etcd 集群中。

例如本地启动一个 `etcd` 服务实例后，可以用如下命令进行查看。

```bash
$ etcdctl member list
422a74f03b622fef, started, node1, http://172.16.238.100:2380, http://172.16.238.100:23
```

## Fedora CoreOS

`CoreOS` 是一个专门为安全和大规模运行容器化工作负载而构建的新 Fedora 版本，它继承了 Fedora Atomic Host 和 CoreOS Container Linux 的优势。

`CoreOS` 的安装文件和运行依赖非常小，它提供了精简的 Linux 系统。它使用 Linux 容器在更高的抽象层来管理你的服务，而不是通过常规的包管理工具 `yum` 或 `apt` 来安装包。

同时，`CoreOS` 几乎可以运行在任何平台：`VirtualBox` `Amazon EC2` `QEMU/KVM` `VMware` `Bare Metal` 和 `OpenStack` 等 。

### Fedora CoreOS 介绍

[Fedora CoreOS](https://getfedora.org/coreos/) 是一个自动更新的，最小的，整体的，以容器为中心的操作系统，不仅适用于集群，而且可独立运行，并针对运行 Kubernetes 进行了优化。它旨在结合 CoreOS Container Linux 和 Fedora Atomic Host 的优点，将 Container Linux 中的 [Ignition](https://github.com/coreos/ignition) 与 [rpm-ostree](https://github.com/coreos/rpm-ostree) 和 Project Atomic 中的 SELinux 强化等技术相集成。其目标是提供最佳的容器主机，以安全，大规模地运行容器化的工作负载。

#### FCOS 特性

##### 一个最小化操作系统

FCOS 被设计成一个基于容器的最小化的现代操作系统。它比现有的 Linux 安装平均节省 40% 的 RAM（大约 114M ）并允许从 PXE 或 iPXE 非常快速的启动。

##### 系统初始化

Ignition 是一种配置实用程序，可读取配置文件（JSON 格式）并根据该配置配置 FCOS 系统。可配置的组件包括存储，文件系统，systemd 和用户。

Ignition 在系统首次启动期间（在 initramfs 中）仅运行一次。由于 Ignition 在启动过程中的早期运行，因此它可以在用户空间开始启动之前重新对磁盘分区，格式化文件系统，创建用户并写入文件。当 systemd 启动时，systemd 服务已被写入磁盘，从而加快了启动时间。

##### 自动更新

FCOS 使用 rpm-ostree 系统进行事务性升级。无需像 yum 升级那样升级单个软件包，而是 rpm-ostree 将 OS 升级作为一个原子单元进行。新的 OS 部署在升级期间进行，并在下次重新引导时生效。如果升级出现问题，则一次回滚和重新启动会使系统返回到先前的状态。确保了系统升级对群集容量的影响降到最小。

##### 容器工具

对于诸如构建，复制和其他管理容器的任务，FCOS 用一组容器工具代替了 **Docker CLI**。**podman CLI** 工具支持许多容器运行时功能，例如运行，启动，停止，列出和删除容器和镜像。**skopeo CLI** 工具可以复制，认证和签名镜像。您还可以使用 **crictl CLI** 工具来处理 CRI-O 容器引擎中的容器和镜像。

## podman

[`podman`](https://github.com/containers/podman) 是一个无守护程序与 docker 命令兼容的下一代 Linux 容器工具。

```bash
# 安装
$ sudo yum -y install podman

# 使用
# podman 与 docker 命令完全兼容，只需将 docker 替换为 podman 即可，例如运行一个容器：
# $ docker run -d -p 80:80 nginx:alpine
$ podman run -d -p 80:80 nginx:alpine
```

## Kubernetes

`Kubernetes` 是 Google 团队发起并维护的基于 Docker 的开源容器集群管理系统，它不仅支持常见的云平台，而且支持内部数据中心。

建于 Docker 之上的 `Kubernetes` 可以构建一个容器的调度服务，其目的是让用户透过 `Kubernetes` 集群来进行云端容器集群的管理，而无需用户进行复杂的设置工作。系统会自动选取合适的工作节点来执行具体的容器集群调度处理工作。其核心概念是 `Container Pod`。一个 `Pod` 由一组工作于同一物理工作节点的容器构成。这些组容器拥有相同的网络命名空间、IP以及存储配额，也可以根据实际情况对每一个 `Pod` 进行端口映射。此外，`Kubernetes` 工作节点会由主系统进行管理，节点包含了能够运行 Docker 容器所用到的服务。

Kubernetes 的目标是管理跨多个主机的容器，提供基本的部署，维护以及应用伸缩，主要实现语言为 Go 语言。Kubernetes 是：

- 易学：轻量级，简单，容易理解
- 便携：支持公有云，私有云，混合云，以及多种云平台
- 可拓展：模块化，可插拔，支持钩子，可任意组合
- 自修复：自动重调度，自动重启，自动复制

在分布式系统中，部署，调度，伸缩一直是最为重要的也最为基础的功能。Kubernetes 就是希望解决这一序列问题的。

### 基本概念

![img](https://vuepress.mirror.docker-practice.com/assets/img/kubernetes_design.94f04f39.jpg)

- 节点（`Node`）：一个节点是一个运行 Kubernetes 中的主机。
- 容器组（`Pod`）：一个 Pod 对应于由若干容器组成的一个容器组，同个组内的容器共享一个存储卷(volume)。
- 容器组生命周期（`pos-states`）：包含所有容器状态集合，包括容器组状态类型，容器组生命周期，事件，重启策略，以及 replication controllers。
- Replication Controllers：主要负责指定数量的 pod 在同一时间一起运行。
- 服务（`services`）：一个 Kubernetes 服务是容器组逻辑的高级抽象，同时也对外提供访问容器组的策略。
- 卷（`volumes`）：一个卷就是一个目录，容器对其有访问权限。
- 标签（`labels`）：标签是用来连接一组对象的，比如容器组。标签可以被用来组织和选择子对象。
- 接口权限（`accessing_the_api`）：端口，IP 地址和代理的防火墙规则。
- web 界面（`ux`）：用户可以通过 web 界面操作 Kubernetes。
- 命令行操作（`cli`）：`kubectl`命令。



### 基本架构

- 分布式架构，保证扩展性；
- 逻辑集中式的控制平面 + 物理分布式的运行平面；
- 一套资源调度系统，管理哪个容器该分配到哪个节点上；
- 一套对容器内服务进行抽象和 HA 的系统。

#### 运行原理

![Kubernetes 架构](https://vuepress.mirror.docker-practice.com/assets/img/k8s_architecture.1cde0882.png)

可见，Kubernetes 首先是一套分布式系统，由多个节点组成，节点分为两类：一类是属于管理平面的主节点/控制节点（Master Node）；一类是属于运行平面的工作节点（Worker Node）。

显然，复杂的工作肯定都交给控制节点去做了，工作节点负责提供稳定的操作接口和能力抽象即可。

从这张图上，我们没有能发现 Kubernetes 中对于控制平面的分布式实现，但是由于数据后端自身就是一套分布式的数据库 Etcd，因此可以很容易扩展到分布式实现。

#### 控制平面

##### 主节点服务

主节点上需要提供如下的管理服务：

- `apiserver` 是整个系统的对外接口，提供一套 RESTful 的 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/)，供客户端和其它组件调用；
- `scheduler` 负责对资源进行调度，分配某个 pod 到某个节点上。是 pluggable 的，意味着很容易选择其它实现方式；
- `controller-manager` 负责管理控制器，包括 endpoint-controller（刷新服务和 pod 的关联信息）和 replication-controller（维护某个 pod 的复制为配置的数值）。

##### etcd

这里 etcd 即作为数据后端，又作为消息中间件。

通过 etcd 来存储所有的主节点上的状态信息，很容易实现主节点的分布式扩展。

组件可以自动的去侦测 etcd 中的数值变化来获得通知，并且获得更新后的数据来执行相应的操作。

#### 工作节点

- kubelet 是工作节点执行操作的 agent，负责具体的容器生命周期管理，根据从数据库中获取的信息来管理容器，并上报 pod 运行状态等；
- kube-proxy 是一个简单的网络访问代理，同时也是一个 Load Balancer。它负责将访问到某个服务的请求具体分配给工作节点上的 Pod（同一类标签）。

![Proxy 代理对服务的请求](https://vuepress.mirror.docker-practice.com/assets/img/kube-proxy.e356ec8f.png)

### kubectl 使用

[kubectl](https://github.com/kubernetes/kubernetes) 是 Kubernetes 自带的客户端，可以用它来直接操作 Kubernetes。

使用格式有两种：

```bash
kubectl [flags]
kubectl [command]
```

get：显示一个或多个资源

describe：显示资源详情

create：从文件或标准输入创建资源

update：从文件或标准输入更新资源

delete：通过文件名、标准输入、资源名或者 label selector 删除资源

log：输出 pod 中一个容器的日志

rolling-update：对指定的 replication controller 执行滚动升级

exec：在容器内部执行命令

port-forward：将本地端口转发到Pod

proxy：为 Kubernetes API server 启动代理服务器

run：在集群中使用指定镜像启动容器

expose：将 replication controller service 或 pod 暴露为新的 kubernetes service

label：更新资源的 label

config：修改 kubernetes 配置文件

cluster-info：显示集群信息

api-versions：以 "组/版本" 的格式输出服务端支持的 API 版本

version：输出服务端和客户端的版本信息

help：显示各个命令的帮助信息

## 容器与云计算

Docker 目前已经得到了众多公有云平台的支持，并成为除虚拟机之外的核心云业务。

除了 AWS、Google、Azure 等，国内的各大公有云厂商，基本上都同时支持了虚拟机服务和基于 Kubernetes 的容器云业务。

目前与容器相关的云计算主要分为两种类型：

一种是传统的 IaaS 服务商提供对容器相关的服务，包括镜像下载、容器托管等。

另一种是直接基于容器技术对外提供容器云服务，所谓 Container as a Service（CaaS）。

### 腾讯云：

![img](https://mc.qcloudimg.com/static/img/0581dbeb97c869bbe6e62025dbc592d7/image.png)

### 阿里云

![img](https://img.alicdn.com/tps/TB10yjtPpXXXXacXXXXXXXXXXXX-1531-1140.png)

### 亚马逊云

![AWS 容器服务](https://vuepress.mirror.docker-practice.com/assets/img/ECS.1106e042.jpg)
