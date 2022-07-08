# Docker 从入门到实践-高级操作

## 第七章 数据管理

在容器中管理数据主要有两种方式：

- 数据卷（Volumes）
- 挂载主机目录 (Bind mounts)

![img](https://vuepress.mirror.docker-practice.com/assets/img/types-of-mounts.cd09b2d7.png)

### 数据卷

`数据卷` 是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：

- `数据卷` 可以在容器之间共享和重用
- 对 `数据卷` 的修改会立马生效
- 对 `数据卷` 的更新，不会影响镜像
- `数据卷` 默认会一直存在，即使容器被删除

> 注意：`数据卷` 的使用，类似于 Linux 下对目录或文件进行 mount，镜像中的被指定为挂载点的目录中的文件会复制到数据卷中（仅数据卷为空时会复制）。

#### 创建一个数据卷

```bash
# 创建一个数据卷
$ docker volume create my-vol
my-vol

# 查看所有的数据卷
$ docker volume ls
DRIVER    VOLUME NAME
local     my-vol

# 在主机里使用以下命令可以查看指定数据卷的信息
$ docker volume inspect my-vol
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```

#### 启动一个挂载数据卷的容器

在用 `docker run` 命令的时候，使用 `--mount` 标记来将 `数据卷` 挂载到容器里。在一次 `docker run` 中可以挂载多个 `数据卷`。

下面创建一个名为 `web` 的容器，并加载一个 `数据卷` 到容器的 `/usr/share/nginx/html` 目录。

```bash
$ docker run -d -P \
    --name web \
    --mount source=my-vol,target=/usr/share/nginx/html \
    nginx:alpine
da90d8ed97a2b21b48042e09ec3f1d78d1d05300c4976a31083e63f09c92a69e
```

#### 查看数据卷的具体信息

在主机里使用以下命令可以查看 `web` 容器的信息

```bash
$ docker inspect web
```

`数据卷` 信息在 "Mounts" Key 下面：

```bash
        "Mounts": [
            {
                "Type": "volume",
                "Name": "my-vol",
                "Source": "/var/lib/docker/volumes/my-vol/_data",
                "Destination": "/usr/share/nginx/html",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
            }
        ],
```

#### 删除数据卷

```bash
$ docker volume rm my-vol
```

`数据卷` 是被设计用来持久化数据的，它的生命周期独立于容器，Docker 不会在容器被删除后自动删除 `数据卷`，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的 `数据卷`。如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 `docker rm -v` 这个命令。

无主的数据卷可能会占据很多空间，要清理请使用以下命令。

```bash
$ docker volume prune
```

### 挂载主机目录

#### 挂载一个主机目录作为数据卷

使用 `--mount` 标记可以指定挂载一个本地主机的目录到容器中去。

```bash
$ docker run -d -P \
    --name web2 \
    --mount type=bind,source=/Users/uu/webapp,target=/usr/share/nginx/html \
    nginx:alpine
a7cfd2a0eedb1fe52d1afa24e50530ef6d78918a30512a957f716c3ef35f0405
```

上面的命令加载主机的 `/src/webapp` 目录到容器的 `/usr/share/nginx/html`目录。这个功能在进行测试的时候十分方便，比如用户可以放置一些程序到本地目录中，来查看容器是否正常工作。

Docker 挂载主机目录的默认权限是 `读写`，用户也可以通过增加 `readonly` 指定为 `只读`。

```bash
$ docker run -d -P \
    --name web \
    --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html,readonly \
    nginx:alpine
a7cfd2a0eedb1fe52d1afa24e50530ef6d78918a30512a957f716c3ef35f0405
```

#### 查看数据卷的具体信息

在主机里使用以下命令可以查看 `web` 容器的信息

```bash
$ docker inspect web2
```

`数据卷` 信息在 "Mounts" Key 下面：

```bash
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/Users/uu/webapp",
                "Destination": "/usr/share/nginx/html",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
```

#### 挂载一个本地主机文件作为数据卷

`--mount` 标记也可以从主机挂载单个文件到容器中。

## 第八章 使用网络

Docker 允许通过外部访问容器或容器互联的方式来提供网络服务。

### 外部访问容器

容器中可以运行一些网络应用，要让外部也可以访问这些应用，可以通过 `-P` 或 `-p` 参数来指定端口映射。

当使用 `-P` 标记时，Docker 会随机映射一个端口到内部容器开放的网络端口。

`-p` 则可以指定要映射的端口，并且，在一个指定端口上只可以绑定一个容器。支持的格式有 `ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort`。

端口映射例子：

```bash
# 映射所有接口地址
# 使用 hostPort:containerPort 格式本地的 80 端口映射到容器的 80 端口
$ docker run -d -p 80:80 nginx:alpine

# 映射到指定地址的指定端口
# 可以使用 ip:hostPort:containerPort 格式指定映射使用一个特定地址，比如 localhost 地址 127.0.0.1
$ docker run -d -p 127.0.0.1:80:80 nginx:alpine

# 映射到指定地址的任意端口
# 使用 ip::containerPort 绑定 localhost 的任意端口到容器的 80 端口，本地主机会自动分配一个端口。
$ docker run -d -p 127.0.0.1::80 nginx:alpine
# 还可以使用 udp 标记来指定 udp 端口
$ docker run -d -p 127.0.0.1:80:80/udp nginx:alpine
```

查看映射端口配置：

```bash
# 查看映射端口配置
# 使用 docker port 来查看当前映射的端口配置，也可以查看到绑定的地址
$ docker port web
80/tcp -> 0.0.0.0:55000
$ docker port web 80
0.0.0.0:55000
```

注意：

- 容器有自己的内部网络和 ip 地址（使用 `docker inspect` 查看，Docker 还可以有一个可变的网络配置。）
- `-p` 标记可以多次使用来绑定多个端口

```bash
$ docker run -d \
    -p 80:80 \
    -p 443:443 \
    nginx:alpine
```

### 容器互联

#### 新建网络

先创建一个新的 Docker 网络。

```bash
$ docker network create -d bridge my-net
93a79c1e86974394938743ffb20b9e99deb838ad2ff6c76fb415eac8a90dfc0e
```

`-d` 参数指定 Docker 网络类型，有 `bridge` `overlay`。其中 `overlay` 网络类型用于 [Swarm mode](https://vuepress.mirror.docker-practice.com/swarm_mode/)。

#### 连接容器

运行一个容器并连接到新建的 `my-net` 网络

```bash
$ docker run -it --rm --name busybox1 --network my-net busybox sh
/ # 
```

打开新的终端，再运行一个容器并加入到 `my-net` 网络

```bash
$ docker run -it --rm --name busybox2 --network my-net busybox sh
/ # 
```

再打开一个新的终端查看容器信息

```bash
$ docker container ls
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                   NAMES
9b350af0cee2   busybox        "sh"                     28 seconds ago   Up 27 seconds                           busybox2
691f8204f789   busybox        "sh"                     42 seconds ago   Up 42 seconds                           busybox1
```

通过 `ping` 来证明 `busybox1` 容器和 `busybox2` 容器建立了互联关系。

在 `busybox1` 容器输入以下命令

```bash
/ # ping busybox2
PING busybox2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.138 ms
```

用 ping 来测试连接 `busybox2` 容器，它会解析成 `172.18.0.3`。

#### Docker Compose

如果有多个容器之间需要互相连接，推荐使用 Docker Compose。

### 配置 DNS

在容器中使用 `mount` 命令可以看到挂载信息：

```bash
$ mount
/dev/disk/by-uuid/1fec...ebdf on /etc/hostname type ext4 ...
/dev/disk/by-uuid/1fec...ebdf on /etc/hosts type ext4 ...
tmpfs on /etc/resolv.conf type tmpfs ...
```

这种机制可以让宿主主机 DNS 信息发生更新后，所有 Docker 容器的 DNS 配置通过 `/etc/resolv.conf` 文件立刻得到更新。

配置全部容器的 DNS ，也可以在 `/etc/docker/daemon.json` 文件中增加以下内容来设置。

```json
{
  "dns" : [
    "114.114.114.114",
    "8.8.8.8"
  ]
}
```

这样每次启动的容器 DNS 自动配置为 `114.114.114.114` 和 `8.8.8.8`。使用以下命令来证明其已经生效。

```bash
$ docker run -it --rm ubuntu:18.04  cat etc/resolv.conf
nameserver 114.114.114.114
nameserver 8.8.8.8
```

如果用户想要手动指定容器的配置，可以在使用 `docker run` 命令启动容器时加入如下参数：

`-h HOSTNAME` 或者 `--hostname=HOSTNAME` 设定容器的主机名，它会被写到容器内的 `/etc/hostname` 和 `/etc/hosts`。但它在容器外部看不到，既不会在 `docker container ls` 中显示，也不会在其他的容器的 `/etc/hosts` 看到。

`--dns=IP_ADDRESS` 添加 DNS 服务器到容器的 `/etc/resolv.conf` 中，让容器用这个服务器来解析所有不在 `/etc/hosts` 中的主机名。

`--dns-search=DOMAIN` 设定容器的搜索域，当设定搜索域为 `.example.com` 时，在搜索一个名为 host 的主机时，DNS 不仅搜索 host，还会搜索 `host.example.com`。

> 注意：如果在容器启动时没有指定最后两个参数，Docker 会默认用主机上的 `/etc/resolv.conf` 来配置容器。

## 第九章 高级网络配置

当 Docker 启动时，会自动在主机上创建一个 `docker0` 虚拟网桥，实际上是 Linux 的一个 bridge，可以理解为一个软件交换机。它会在挂载到它的网口之间进行转发。

同时，Docker 随机分配一个本地未占用的私有网段（在 [RFC1918](https://datatracker.ietf.org/doc/html/rfc1918) 中定义）中的一个地址给 `docker0` 接口。比如典型的 `172.17.42.1`，掩码为 `255.255.0.0`。此后启动的容器内的网口也会自动分配一个同一网段（`172.17.0.0/16`）的地址。

当创建一个 Docker 容器的时候，同时会创建了一对 `veth pair` 接口（当数据包发送到一个接口时，另外一个接口也可以收到相同的数据包）。这对接口一端在容器内，即 `eth0`；另一端在本地并被挂载到 `docker0` 网桥，名称以 `veth` 开头（例如 `vethAQI2QT`）。通过这种方式，主机可以跟容器通信，容器之间也可以相互通信。Docker 就创建了在主机和所有容器之间一个虚拟共享网络。

![Docker 网络](https://vuepress.mirror.docker-practice.com/assets/img/network.6ad909f2.png)

### 快速配置指南

下面是一个跟 Docker 网络相关的命令列表。

其中有些命令选项只有在 Docker 服务启动的时候才能配置，而且不能马上生效。

- `-b BRIDGE` 或 `--bridge=BRIDGE` 指定容器挂载的网桥
- `--bip=CIDR` 定制 docker0 的掩码
- `-H SOCKET...` 或 `--host=SOCKET...` Docker 服务端接收命令的通道
- `--icc=true|false` 是否支持容器之间进行通信
- `--ip-forward=true|false` 请看下文容器之间的通信
- `--iptables=true|false` 是否允许 Docker 添加 iptables 规则
- `--mtu=BYTES` 容器网络中的 MTU

下面2个命令选项既可以在启动服务时指定，也可以在启动容器时指定。在 Docker 服务启动的时候指定则会成为默认值，后面执行 `docker run` 时可以覆盖设置的默认值。

- `--dns=IP_ADDRESS...` 使用指定的DNS服务器
- `--dns-search=DOMAIN...` 指定DNS搜索域

最后这些选项只有在 `docker run` 执行时使用，因为它是针对容器的特性内容。

- `-h HOSTNAME` 或 `--hostname=HOSTNAME` 配置容器主机名
- `--link=CONTAINER_NAME:ALIAS` 添加到另一个容器的连接
- `--net=bridge|none|container:NAME_or_ID|host` 配置容器的桥接模式
- `-p SPEC` 或 `--publish=SPEC` 映射容器端口到宿主主机
- `-P or --publish-all=true|false` 映射容器所有端口到宿主主机

### 容器访问控制

容器的访问控制，主要通过 Linux 上的 `iptables` 防火墙来进行管理和实现。`iptables` 是 Linux 上默认的防火墙软件，在大部分发行版中都自带。

#### 容器访问外部网络

容器要想访问外部网络，需要本地系统的转发支持。在Linux 系统中，检查转发是否打开。

```bash
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0
```

如果为 0，说明没有开启转发，则需要手动打开。

```bash
$ sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
```

#### 容器之间访问

容器之间相互访问，需要两方面的支持。

- 容器的网络拓扑是否已经互联。默认情况下，所有容器都会被连接到 `docker0` 网桥上。
- 本地系统的防火墙软件 -- `iptables` 是否允许通过。

#### 访问所有端口

当启动 Docker 服务（即 dockerd）的时候，默认会添加一条转发策略到本地主机 iptables 的 FORWARD 链上。策略为通过（`ACCEPT`）还是禁止（`DROP`）取决于配置`--icc=true`（缺省值）还是 `--icc=false`。当然，如果手动指定 `--iptables=false` 则不会添加 `iptables` 规则。

可见，默认情况下，不同容器之间是允许网络互通的。如果为了安全考虑，可以在 `/etc/docker/daemon.json` 文件中配置 `{"icc": false}` 来禁止它。

#### 访问指定端口

在通过 `-icc=false` 关闭网络访问后，还可以通过 `--link=CONTAINER_NAME:ALIAS` 选项来访问容器的开放端口。

例如，在启动 Docker 服务时，可以同时使用 `icc=false --iptables=true` 参数来关闭允许相互的网络访问，并让 Docker 可以修改系统中的 `iptables` 规则。此时，系统中的 `iptables` 规则可能是类似。

```bash
$ sudo iptables -nL
...
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0
...
```

之后，启动容器（`docker run`）时使用 `--link=CONTAINER_NAME:ALIAS` 选项。Docker 会在 `iptable` 中为 两个容器分别添加一条 `ACCEPT` 规则，允许相互访问开放的端口（取决于 `Dockerfile` 中的 `EXPOSE` 指令）。当添加了 `--link=CONTAINER_NAME:ALIAS` 选项后，添加了 `iptables` 规则。

```bash
$ sudo iptables -nL
...
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  172.17.0.2           172.17.0.3           tcp spt:80
ACCEPT     tcp  --  172.17.0.3           172.17.0.2           tcp dpt:80
DROP       all  --  0.0.0.0/0            0.0.0.0/0
```

注意：`--link=CONTAINER_NAME:ALIAS` 中的 `CONTAINER_NAME` 目前必须是 Docker 分配的名字，或使用 `--name` 参数指定的名字。主机名则不会被识别。

### 映射容器端口到宿主主机的实现

默认情况下，容器可以主动访问到外部网络的连接，但是外部网络无法访问到容器。

#### 容器访问外部实现

容器所有到外部网络的连接，源地址都会被 NAT 成本地系统的 IP 地址。这是使用 `iptables` 的源地址伪装操作实现的。查看主机的 NAT 规则。

```bash
$ sudo iptables -t nat -nL
...
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16       !172.17.0.0/16
...
```

其中，上述规则将所有源地址在 `172.17.0.0/16` 网段，目标地址为其他网段（外部网络）的流量动态伪装为从系统网卡发出。MASQUERADE 跟传统 SNAT 的好处是它能动态从网卡获取地址。

#### 外部访问容器实现

容器允许外部访问，可以在 `docker run` 时候通过 `-p` 或 `-P` 参数来启用。

不管用那种办法，其实也是在本地的 `iptable` 的 nat 表中添加相应的规则。

使用 `-P` 时：

```bash
$ iptables -t nat -nL
...
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:49153 to:172.17.0.2:80
```

使用 `-p 80:80` 时：

```bash
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.2:80
```

注意：

- 这里的规则映射了 `0.0.0.0`，意味着将接受主机来自所有接口的流量。用户可以通过 `-p IP:host_port:container_port` 或 `-p IP::port` 来指定允许访问容器的主机上的 IP、接口等，以制定更严格的规则。
- 如果希望永久绑定到某个固定的 IP 地址，可以在 Docker 配置文件 `/etc/docker/daemon.json` 中添加如下内容。

```json
{
  "ip": "0.0.0.0"
}
```

### 自定义网桥

除了默认的 `docker0` 网桥，用户也可以指定网桥来连接各个容器。

在启动 Docker 服务的时候，使用 `-b BRIDGE`或`--bridge=BRIDGE` 来指定使用的网桥。

如果服务已经运行，那需要先停止服务，并删除旧的网桥。

```bash
$ sudo systemctl stop docker
$ sudo ip link set dev docker0 down
$ sudo brctl delbr docker0
```

然后创建一个网桥 `bridge0`。

```bash
$ sudo brctl addbr bridge0
$ sudo ip addr add 192.168.5.1/24 dev bridge0
$ sudo ip link set dev bridge0 up
```

查看确认网桥创建并启动。

```bash
$ ip addr show bridge0
4: bridge0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state UP group default
    link/ether 66:38:d0:0d:76:18 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.1/24 scope global bridge0
       valid_lft forever preferred_lft forever
```

在 Docker 配置文件 `/etc/docker/daemon.json` 中添加如下内容，即可将 Docker 默认桥接到创建的网桥上。

```json
{
  "bridge": "bridge0",
}
```

启动 Docker 服务。

新建一个容器，可以看到它已经桥接到了 `bridge0` 上。

可以继续用 `brctl show` 命令查看桥接的信息。另外，在容器中可以使用 `ip addr` 和 `ip route` 命令来查看 IP 地址配置和路由信息。

### 工具和示例

#### pipework

Jérôme Petazzoni 编写了一个叫 [pipework](https://github.com/jpetazzo/pipework) 的 shell 脚本，可以帮助用户在比较复杂的场景中完成容器的连接。

#### playground

Brandon Rhodes 创建了一个提供完整的 Docker 容器网络拓扑管理的 [Python库](https://github.com/brandon-rhodes/fopnp/tree/m/playground)，包括路由、NAT 防火墙；以及一些提供 `HTTP` `SMTP` `POP` `IMAP` `Telnet` `SSH` `FTP` 的服务器。

### 编辑网络配置文件

Docker 1.2.0 开始支持在运行中的容器里编辑 `/etc/hosts`, `/etc/hostname` 和 `/etc/resolv.conf` 文件。

但是这些修改是临时的，只在运行的容器中保留，容器终止或重启后并不会被保存下来，也不会被 `docker commit` 提交。
