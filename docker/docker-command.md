# Docker命令记录



## 常用命令

### docker pull

从 Docker 镜像仓库获取镜像。

#### 命令格式

```
docker pull [选项] [Docker Registry地址[:端口号]/]仓库名[:标签]
```

#### 命令说明

- Docker Registry地址：地址的格式一般是 <域名/IP>[:端口号]。默认地址是 Docker Hub。
- 仓库名：这里的仓库名是两段式名称，即 <用户名>/<软件名>。对于 Docker Hub，如果不给出用户名，则默认为 library，也就是官方镜像。

#### 示例

```shell
$ docker pull ubuntu:18.04
```

```shell
$ docker pull gcr.azk8s.cn/google_containers/hyperkube-amd64:v1.9.2
```



### docker run

用于运行指定镜像的容器。

#### 命令格式

```shell

```

#### 命令说明

#### 示例

示例一：启动ubuntu:18.04镜像容器，并通过bash进行交互式操作。

```shell
$ docker run -it --rm \
    ubuntu:18.04 \
    bash
```

上述命令中的选项和参数说明如下：

- `-it`：这是两个参数，一个是 `-i`：交互式操作，一个是 `-t` 终端。我们这里打算进入 `bash` 执行一些命令并查看返回结果，因此我们需要交互式终端。
- `--rm`：这个参数是说容器退出后随之将其删除。默认情况下，为了排障需求，退出的容器并不会立即删除，除非手动 `docker rm`。我们这里只是随便执行个命令，看看结果，不需要排障和保留结果，因此使用 `--rm` 可以避免浪费空间。
- `ubuntu:18.04`：这是指用 `ubuntu:18.04` 镜像为基础来启动容器。
- `bash`：放在镜像名后的是 **命令**，这里我们希望有个交互式 Shell，因此用的是 `bash`。



### docker image ls

列出已经下载下来的镜像。

#### 命令格式

```
docker image ls
```

#### 命令说明

使用该命令可以列出已经下载下来的镜像列表，包含仓库名、标签、镜像ID、创建时间以及所占用的空间。

`docker image ls` 列表中的镜像体积总和并非是所有镜像实际硬盘消耗。由于 Docker 镜像是多层存储结构，并且可以继承、复用，因此不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。由于 Docker 使用 Union FS，相同的层只需要保存一份即可，因此实际镜像硬盘占用空间很可能要比这个列表镜像大小的总和要小的多。

同时，也可以使用该命令显示虚悬镜像。（具体见示例二说明）

#### 示例

示例一：

```
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redis               latest              5f515359c7f8        5 days ago          183 MB
ubuntu              18.04               f753707788c5        4 weeks ago         127 MB
ubuntu              latest              f753707788c5        4 weeks ago         127 MB
```

上述列出的镜像列表说明如下：

列表包含了 `仓库名`、`标签`、`镜像 ID`、`创建时间` 以及 `所占用的空间`。

**镜像 ID** 则是镜像的唯一标识，一个镜像可以对应多个 **标签**。例如上面的`ubuntu:18.04` 和 `ubuntu:latest` 拥有相同的 ID，因为它们对应的是同一个镜像。

#### **显示虚悬镜像**

```shell
$ docker image ls -f dangling=true
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              00285df0df87        5 days ago          342 MB
```

上面列表中显示的的镜像是一类特殊的镜像，这些镜像既没有仓库名，也没有标签，均为 `<none>`。原因是随着官方镜像维护，发布了新版本后，原本的镜像名被转移到了新下载的镜像身上，而旧的镜像上的这个名称则被取消，从而成为了 `<none>`。

由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 <none> 的镜像。这类无标签镜像也被称为 ==虚悬镜像(==dangling image) 。

可以用下面的命令专门显示这类镜像：

```
docker image ls -f dangling=true
```

一般来说，虚悬镜像已经失去了存在的价值，是可以随意删除的，可以用下面的命令删除。

```shell
$ docker image prune
```

#### 显示中间层镜像

为了加速镜像构建、重复利用资源，Docker 会利用 **中间层镜像**。中间层镜像，是其它镜像所依赖的镜像。

默认的 `docker image ls` 列表中只会显示顶层镜像，如果希望显示包括中间层镜像在内的所有镜像的话，需要加 `-a` 参数。

```
$ docker image ls -a
```

#### 列出部分镜像

方式一：根据仓库名列出镜像

```
$ docker image ls ubuntu
```

方式二：列出特定的某个镜像，也就是说指定仓库名和标签

```
$ docker image ls ubuntu:18.04
```

方式三：使用过滤器参数--filter，或者简写-f。

例如，查看在 `mongo:3.2` 之后建立的镜像：

```
$ docker image ls -f since=mongo:3.2
```

想查看某个位置之前的镜像也可以，只需要把 `since` 换成 `before` 即可。

此外，如果镜像构建时，定义了 `LABEL`，还可以通过 `LABEL` 来过滤。

```
$ docker image ls -f label=com.example.version=0.1
```

#### 以特定格式显示镜像

```
$ docker image ls -q
5f515359c7f8
05a60462f8ba
```

```
$ docker image ls --format "{{.ID}}: {{.Repository}}"
5f515359c7f8: redis
05a60462f8ba: nginx
fe9198c04d62: mongo
```

```
$ docker image ls --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}"
IMAGE ID            REPOSITORY          TAG
5f515359c7f8        redis               latest
05a60462f8ba        nginx               latest
```



### docker system df 

用于查看镜像、容器、数据卷所占用的空间。

#### 命令格式

```
docker system df
```

#### 示例

```
$ docker system df

TYPE			TOTAL		ACTIVE		SIZE			RECLAIMABLE
Images      	24      	0           1.992GB     	1.992GB (100%)
Containers      1           0           62.82MB         62.82MB (100%)
Local Volumes   9           0           652.2MB         652.2MB (100%)
Build Cache                             0B              0B
```

上面列表中显示的最后一条镜像为虚悬镜像（见上文说明）。





### docker image rm

删除本地镜像。

#### 命令格式

```
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```

#### 命令说明

命令中的<镜像>可以是`镜像短 ID`、`镜像长 ID`、`镜像名` 或者 `镜像摘要`。

可以通过`docker image ls`查看镜像的ID，然后摘取ID的前几位可以区分不同镜像的字符即可。

例如：

```
$ docker image ls
REPOSITORY             TAG           IMAGE ID            CREATED             SIZE
centos                 latest        0584b3d2cf6d        3 weeks ago         196.5 MB
redis                  alpine        501ad78535f0        3 weeks ago         21.03 MB
```

示例一，通过ID删除 `redis:alpine` 镜像，可以执行：

```
$ docker image rm 501
Untagged: redis:alpine
...
```

示例二，使用`镜像名`，也就是 `<仓库名>:<标签>`，来删除镜像。

```
$ docker image rm centos
Untagged: centos:latest
```

示例三，更精确的是使用 `镜像摘要` 删除镜像。

```
$ docker image ls --digests
REPOSITORY                  TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
node                        slim                sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228   6e0c4c8e3913        3 weeks ago         214 MB

$ docker image rm node@sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228
Untagged: node@sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228
```

#### 与`docker image ls`命令的配合使用

示例一，删除所有仓库名为 `redis` 的镜像：

```
$ docker image rm $(docker image ls -q redis)
```

示例二，删除所有在 `mongo:3.2` 之前的镜像：

```
$ docker image rm $(docker image ls -q -f before=mongo:3.2)
```









