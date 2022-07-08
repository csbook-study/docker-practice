# Docker 从入门到实践-核心项目

## 第十章 Docker Buildx

Docker Buildx 是一个 docker CLI 插件，其扩展了 docker 命令，支持 Moby BuildKit 提供的功能。提供了与 docker build 相同的用户体验，并增加了许多新功能。

> 该功能仅适用于 Docker v19.03+ 版本

### 使用 BuildKit 构建镜像

**BuildKit** 是下一代的镜像构建组件，在 <https://github.com/moby/buildkit> 开源。

目前，Docker Hub 自动构建已经支持 buildkit，具体请参考 <https://github.com/docker-practice/docker-hub-buildx>。

**由于 `BuildKit` 为实验特性，每个 `Dockerfile` 文件开头都必须加上如下指令：**

```dockerfile
# syntax = docker/dockerfile:experimental
```

#### Dockerfile 新增指令详解

启用 `BuildKit` 之后，我们可以使用下面几个新的 `Dockerfile` 指令来加快镜像构建。

##### RUN --mount=type=cache

目前，几乎所有的程序都会使用依赖管理工具，例如 `Go` 中的 `go mod`、`Node.js` 中的 `npm` 等等，当我们构建一个镜像时，往往会重复的从互联网中获取依赖包，难以缓存，大大降低了镜像的构建效率。

`Dockerfile` 中 `--mount=type=cache,...` 中指令作用如下：

|      Option       | Description                                                  |
| :---------------: | :----------------------------------------------------------- |
|       `id`        | `id` 设置一个标志，以便区分缓存。                            |
| `target` (必填项) | 缓存的挂载目标文件夹。                                       |
|  `ro`,`readonly`  | 只读，缓存文件夹不能被写入。                                 |
|     `sharing`     | 有 `shared` `private` `locked` 值可供选择。`sharing` 设置当一个缓存被多次使用时的表现，由于 `BuildKit` 支持并行构建，当多个步骤使用同一缓存时（同一 `id`）会发生冲突。`shared` 表示多个步骤可以同时读写，`private` 表示当多个步骤使用同一缓存时，每个步骤使用不同的缓存，`locked` 表示当一个步骤完成释放缓存后，后一个步骤才能继续使用该缓存。 |
|      `from`       | 缓存来源（构建阶段），不填写时为空文件夹。                   |
|     `source`      | 来源的文件夹路径。                                           |

##### RUN --mount=type=bind

该指令可以将一个镜像（或上一构建阶段）的文件挂载到指定位置。

```dockerfile
# syntax = docker/dockerfile:experimental
RUN --mount=type=bind,from=php:alpine,source=/usr/local/bin/docker-php-entrypoint,target=/docker-php-entrypoint \
        cat /docker-php-entrypoint
```

##### RUN --mount=type=tmpfs

该指令可以将一个 `tmpfs` 文件系统挂载到指定位置。

```dockerfile
# syntax = docker/dockerfile:experimental
RUN --mount=type=tmpfs,target=/temp \
        mount | grep /temp
```

##### RUN --mount=type=secret

该指令可以将一个文件(例如密钥)挂载到指定位置。

```dockerfile
# syntax = docker/dockerfile:experimental
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
        cat /root/.aws/credentials
```

```bash
$ docker build -t test --secret id=aws,src=$HOME/.aws/credentials .
```

##### RUN --mount=type=ssh

该指令可以挂载 `ssh` 密钥。

```dockerfile
# syntax = docker/dockerfile:experimental
FROM alpine
RUN apk add --no-cache openssh-client
RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
RUN --mount=type=ssh ssh git@gitlab.com | tee /hello
```

```bash
$ eval $(ssh-agent)
Agent pid 11449
$ ssh-add ~/.ssh/id_rsa
(Input your passphrase here)
$ docker build -t test --ssh default=$SSH_AUTH_SOCK .
```

### 使用 Buildx 构建镜像

你可以直接使用 `docker buildx build` 命令构建镜像。

```bash
$ docker buildx build .
[+] Building 8.4s (23/32)
 => ...
```

Buildx 使用 BuildKit 引擎 进行构建，支持许多新的功能。

### 使用 buildx 构建多种系统架构支持的 Docker 镜像

在之前的版本中构建多种系统架构支持的 Docker 镜像，要想使用统一的名字必须使用 [`$ docker manifest`](https://vuepress.mirror.docker-practice.com/image/manifest.html) 命令。

在 Docker 19.03+ 版本中可以使用 `$ docker buildx build` 命令使用 `BuildKit` 构建镜像。该命令支持 `--platform` 参数可以同时构建支持多种系统架构的 Docker 镜像，大大简化了构建步骤。

#### 构建镜像

新建 Dockerfile 文件。

```dockerfile
FROM --platform=$TARGETPLATFORM alpine

RUN uname -a > /os.txt

CMD cat /os.txt
```

使用 `$ docker buildx build` 命令构建镜像，注意将 `myusername` 替换为自己的 Docker Hub 用户名。

`--push` 参数表示将构建好的镜像推送到 Docker 仓库。

```bash
$ DOCKER_BUILDKIT=1 docker buildx build --platform linux/arm,linux/arm64,linux/amd64 -t huxiangyuxy/hello . --push
```

在不同架构运行该镜像，可以得到该架构的信息。

```bash
# arm
$ docker run -it --rm myusername/hello
Linux buildkitsandbox 4.9.125-linuxkit #1 SMP Fri Sep 7 08:20:28 UTC 2018 armv7l Linux

# arm64
$ docker run -it --rm myusername/hello
Linux buildkitsandbox 4.9.125-linuxkit #1 SMP Fri Sep 7 08:20:28 UTC 2018 aarch64 Linux

# amd64
$ docker run -it --rm myusername/hello
Linux buildkitsandbox 4.9.125-linuxkit #1 SMP Fri Sep 7 08:20:28 UTC 2018 x86_64 Linux
```

#### 架构相关变量

`Dockerfile` 支持如下架构相关的变量

**TARGETPLATFORM**：构建镜像的目标平台，例如 `linux/amd64`, `linux/arm/v7`, `windows/amd64`。

**TARGETOS**：`TARGETPLATFORM` 的 OS 类型，例如 `linux`, `windows`

**TARGETARCH**：`TARGETPLATFORM` 的架构类型，例如 `amd64`, `arm`

**TARGETVARIANT**：`TARGETPLATFORM` 的变种，该变量可能为空，例如 `v7`

**BUILDPLATFORM**：构建镜像主机平台，例如 `linux/amd64`

**BUILDOS**：`BUILDPLATFORM` 的 OS 类型，例如 `linux`

**BUILDARCH**：`BUILDPLATFORM` 的架构类型，例如 `amd64`

**BUILDVARIANT**：`BUILDPLATFORM` 的变种，该变量可能为空，例如 `v7`
