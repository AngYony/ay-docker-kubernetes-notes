# Docker核心知识与常用命令

登录上Docker所在的主机之后，第一件事就是检查Docker是否正在运行。

```shell
$ docker version
Client: Docker Engine - Community
 Version:           19.03.12
...

Server: Docker Engine - Community
 Engine:
  Version:          19.03.12
  API version:      1.40 (minimum version 1.12)
...
```

如果上述内容包括Client和Server的内容，则说明Docker运行正常。可以通过如下命令，检查Docker daemon的状态：

```shell
$ service docker status
或
$ systemctl is-active docker
```



## docker image（镜像）

- Docker镜像可以简单理解为未运行的容器。
- 容器的目的就是运行应用或服务，因此容器的镜像中必须包含应用/服务运行所必需的操作系统和应用文件。
- 每个镜像的内部是一个精简的仅包含必要的操作系统（OS），同时还包含应用运行所必须的文件和依赖包。Docker镜像就像停止运行的容器。通过docker命令可以从某个镜像启动一个或多个容器。
- 容器从镜像启动后，在容器未全部停止之前，镜像是无法被删除的。
- Linux Docker主机本地镜像仓库通常位于`/var/lib/docker/<storage-driver>`
- 每个Docker镜像仓库服务（Image Registry）可以包含多个镜像仓库（Image Repository），而每个镜像仓库中又可以包含多个不同标签的镜像。因此镜像是存储在镜像仓库服务中的镜像仓库中的。Docker客户端的镜像仓库服务是可配置的，默认使用Docker Hub。
- 同一个镜像可以根据用户需要设置多个标签。例如同时设置标签为latest和edge。在拉取镜像时，如果没有显式指定标签，默认拉取标签为latest的镜像。注意：latest是一个非强制标签，不保证指向仓库中最新的镜像。
- 一个Docker镜像由多个松耦合的只读镜像层组成。
- 当构建Docker镜像时，可以通过嵌入指令来列出希望容器运行时启动的默认应用。

### 镜像与分层

Docker镜像由一些松耦合的只读镜像层组成，Docker负责堆叠这些镜像层，并且将它们表示为单个统一的对象。

所有的Docker镜像都起始于一个基础镜像层，当进行修改或增加新的内容时，就会在当前镜像层之上，创建新的镜像层。

在添加额外的镜像层的同时，镜像始终保持是当前所有镜像层的组合。如果要添加的上层镜像层中的文件，覆盖了底层镜像层中的文件，那么这个要添加的上层镜像层就是一个更新版本，它将会作为一个新镜像层添加到镜像当中。Docker通过存储引擎（新版本采用快照机制）的方式来实现镜像层堆栈，并保证多镜像层对外展示为统一的文件系统。（所有镜像层堆叠并合并，对外提供统一的视图）。

当多个镜像层堆叠在一起，就构成了一个完整镜像。

在拉取镜像的时候，Docker可以识别出要拉取的镜像中，哪几层已经在本地存在，多个镜像之间会共享镜像层，这样可以有效节省空间并提升性能。

可以使用以下方式查看和检查构成某个镜像的分层。

方式一：使用docker image pull命令。

```shell
$ docker image pull mongo
Using default tag: latest
latest: Pulling from library/mongo
a1125296b23d: Pull complete
3c742a4a0f38: Pull complete
4c5ea3b32996: Pull complete
1b4be91ead68: Pull complete
af8504826779: Pull complete
8faaabd5f8b2: Pull complete
...
```

上述一Pull complete结尾的每一行都代表了镜像中某个被拉取的镜像层，最前面的为镜像层ID。

方式二：通过`docker image inspect`命令查看。

```shell
$ docker image inspect mongo
...
 "Layers": [
 "sha256:88cc1a200eb9be206c4261e79d642c392d97980236a066f23434f4452f8212a7",
 "sha256:bf509d6bc5ecd3b1a3660fb5f167ce2e320f5cc532574217cd8ba179ca06bae9",             "sha256:2af0e1f1e531af9c95e30c3f908f6431d635a173b866e39b5f480e9ff3a180b9",
  ...
  ]
...
```

上述输出的内容中使用了镜像的SHA256散列值来标识镜像层。

### 虚悬镜像

```shell
$ docker image ls -f dangling=true
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              00285df0df87        5 days ago          342 MB
```

上面列表中显示的的镜像是一类特殊的镜像，这些镜像既没有仓库名，也没有标签，均为 `<none>`。这类无标签镜像被称为 ==虚悬镜像(==dangling image) ，在列表中展示为`<none>:<none>`。

通常出现这种情况，是因为构建了一个新镜像，然后为该镜像打了一个已经存在的标签。当次情况出现，Docker会构建新的镜像，然后发现已经有镜像包含相同的标签，接着Docker会移除旧镜像上面的标签，并将该标签标在新的镜像之上。同时移除了旧镜像上面对应的标签，旧镜像就变成了悬虚镜像。

可以用下面的命令专门显示这类镜像：

```shell
$ docker image ls -f dangling=true
```

一般来说，虚悬镜像已经失去了存在的价值，是可以随意删除的，可以用下面的命令删除。

```shell
$ docker image prune
```

如果添加了`-a`参数，Docker会额外移除没有被任何容器使用的镜像。

### 中间层镜像

为了加速镜像构建、重复利用资源，Docker 会利用 **中间层镜像**。中间层镜像，是其它镜像所依赖的镜像。

默认的 `docker image ls` 列表中只会显示顶层镜像，如果希望显示包括中间层镜像在内的所有镜像的话，需要加 `-a` 参数。

```shell
$ docker image ls -a
```

### 镜像摘要

对于标签相同的多个镜像，如果需要区分正在使用的镜像版本是修复前还是修复后的，可以通过镜像摘要来区分。

镜像摘要是镜像内容的一个散列值（内容散列），镜像内容的变更一定会导致散列值的改变。因此，如果镜像没有发生变化，它的摘要也是不可变的。

可以通过`docker image ls`命令添加`--digests`参数来查看镜像摘要：

```sh
$ docker image ls --digests alpine
REPOSITORY          TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
alpine              latest              sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321   a24bb4013296        5 weeks ago         5.57MB

```

上述镜像的签名值即为摘要信息，如下：

`sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321`

目前，已经没有原生的Docker命令支持从远端镜像仓库服务（如Docker Hub）中直接获取镜像签名了，因此必须先要通过标签方式拉取镜像到本地，然后自己维护镜像的摘要列表。

通过摘要拉取镜像的命令：

```
docker image pull alpine@<sha256散列值>
```

例如：先删除alpine:latest镜像，然后通过摘要再次拉取该镜像：

```shell
$ docker image rm alpine:latest
Untagged: alpine:latest
Untagged: alpine@sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321
Deleted: sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e
...
$ docker image pull alpine@sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321
sha256:185518070891758909c9f839cf4ca393ee977ac378609f700f60a771a2dfe321: Pulling from library/alpine
df20fa9351a1: Pull complete
...
```



### docker image ls

列出已经下载下来的镜像。

#### 命令格式

```
docker image ls [options]
```

#### 命令说明

使用该命令可以列出已经下载下来的镜像列表，包含仓库名、标签、镜像ID、创建时间以及所占用的空间。

`docker image ls` 列表中的镜像体积总和并非是所有镜像实际硬盘消耗。由于 Docker 镜像是多层存储结构，并且可以继承、复用，因此不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。由于 Docker 使用 Union FS，相同的层只需要保存一份即可，因此实际镜像硬盘占用空间很可能要比这个列表镜像大小的总和要小的多。

同时，也可以使用该命令显示虚悬镜像。（具体见示例二说明）

#### 参数描述

- --filter ( -f )：过滤`docker image ls`命令返回的镜像列表内容。支持如下过滤器：
  - dangling：可以指定true或false，仅返回悬虚镜像（true），或者非悬虚镜像（false）。
  - before：返回在之前被创建的全部镜像，需要镜像名称或者ID作为参数。
  - since：返回指定镜像之后创建的全部镜像。与before用法类似。
  - label：根据标注（label）的名称或值，对镜像镜像过滤。
  - reference：用于其他过滤方式
- -a：显示全部镜像，包括顶层镜像和中间层镜像。
- --format：对输出的内容进行格式化。
- --digests：查看镜像的摘要信息（SHA256签名）。
- -q：返回镜像的ID列表。

#### 综合示例

示例一：列出当前本地Docker主机上已拉取的镜像。

```shell
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redis               latest              5f515359c7f8        5 days ago          183 MB
ubuntu              18.04               f753707788c5        4 weeks ago         127 MB
ubuntu              latest              f753707788c5        4 weeks ago         127 MB
```

上述列出的镜像列表说明如下：

列表包含了 `仓库名`、`标签`、`镜像 ID`、`创建时间` 以及 `所占用的空间`。

**镜像 ID** 则是镜像的唯一标识，一个镜像可以对应多个 **标签**。例如上面的`ubuntu:18.04` 和 `ubuntu:latest` 拥有相同的 ID，因为它们对应的是同一个镜像。

示例二：根据仓库名列出镜像

```shell
$ docker image ls ubuntu
```

示例三：列出特定的某个镜像，也就是说指定仓库名和标签

```shell
$ docker image ls ubuntu:18.04
```

示例四：查看在 `mongo:3.2` 之后建立的镜像：

```shell
$ docker image ls -f since=mongo:3.2
```

想查看某个位置之前的镜像也可以，只需要把 `since` 换成 `before` 即可。

示例五：如果镜像构建时，定义了 `LABEL`，还可以通过 `LABEL` 来过滤。

```shell
$ docker image ls -f label=com.example.version=0.1
```

示例六：使用reference完成过滤并且仅显示标签为latest的镜像。

```shell
$ docker image ls --filter=reference="*:latest"
```

示例七：显示本地拉取的全部镜像的ID列表。

```shell
$ docker image ls -q
5f515359c7f8
05a60462f8ba
```

示例八：只返回Docker主机上镜像的大小属性。

```shell
$ docker image ls --format "{{.Size}}"
73.9MB
5.57MB
13.3kB
425MB
```

示例九：按照指定的格式显示镜像ID、仓库名称、标签信息。

```shell
$ docker image ls --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}"
IMAGE ID            REPOSITORY          TAG
5f515359c7f8        redis               latest
05a60462f8ba        nginx               latest
```

### docker image pull

将镜像取到Docker主机本地的操作是拉取。该命令用于将镜像拉取到本地。只需要给出镜像的仓库名字和标签，就能在官方仓库中定位一个镜像（采用 “:” 分隔）。

docker image pull 可以简写为 docker pull，这两个命令的作用相同，都用于从 Docker 镜像仓库获取镜像。

注意：标签为latest的镜像只是一种普通的镜像，并不保证标有latest标签的镜像是仓库中最新的镜像。（有些仓库中最新的镜像通常标签是edge）

#### 命令格式

```
docker image pull [options] [DockerRegistry地址[:端口号]/]repository[:tag]
```

#### 命令说明

- DockerRegistry地址：地址的格式一般是 <域名/IP>[:端口号]。默认地址是 Docker Hub。
- repository：这里的仓库名是两段式名称，即 <用户名>/<软件名>。对于 Docker Hub，如果不给出用户名，则默认为 library，也就是官方镜像。
- 如果没有在仓库名称后指定具体的镜像标签，则Docker默认拉取标签为latest的镜像。
- 如果是从第三方镜像仓库服务（非Docker Hub）获取镜像，需要在镜像仓库名称前加上第三方镜像仓库服务的DNS名称。

#### 选项描述

- -a：拉取仓库中的所有标签的全部镜像。

#### 综合示例

示例一：从Ubuntu仓库中拉取标签为“latest” 的镜像。

```shell
$ docker image pull ubuntu:latest
```

示例二：从官方Mongo库拉取标签为3.3.11的镜像。

```shell
$ docker image pull mongo:3.3.11
```

示例三：从官方Alpine库拉取标签为latest的镜像。

```shell
$ docker image pull alpine
```

示例四：从Microsoft/powershell仓库中拉取标签为nanoserver的镜像

```shell
$ docker image pull microsoft/powershell:nanoserver
```

示例五：从第三方镜像仓库服务GCR获取镜像。

```shell
$ docker image pull gcr.io/nigelpoulton/tu-demo:v2
$ docker pull gcr.azk8s.cn/google_containers/hyperkube-amd64:v1.9.2
```

### docker image build

根据Dockerfile中的指令来创建新的镜像。

#### 命令格式

```
docker image build [选项] <镜像> <path>
```

#### 命令说明

需要进入到Dockerfile文件所在的目录后，执行该命令。

#### 参数描述

- -t：为镜像打标签
- -f：指定Dockerfile的路径和名称，使用-f参数可以指定位于任意路径下的任意名称的Dockerfile。构建上下文是指应用文件存放的位置，可能是本地Docker主机上的一个目录或一个远程Git库。
- --nocache=true：可以强制忽略对缓存的使用。
- --squash：创建一个合并的镜像。

#### 综合示例

示例：创建镜像名为test:latest的Docker镜像。

```shell
$ docker image build -t test:latest .
```

上述命令会构建并生成一个名为test:latest的镜像，注意：命令最后的点（.）表示Docker在进行构建的时候，使用当前目录作为构建上下文。

### docker image inspect

用于查看镜像的详细信息，包括镜像层数据和元数据。

示例：查看mongo镜像的详细信息。

```shell
$ docker image inspect mongo
```

该命令同时可以查看容器启动时将要运行的应用列表。

```shell
$ docker image inspect nigelpoulton/pluralsight-docker-ci:latest
...
 "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"/bin/sh\" \"-c\" \"cd /src && node ./app.js\"]"
            ],
...
```

上述结果中的Cmd一项展示了容器启动后将会执行的命令或应用。除此之外，也可以在启动的时候，人为手动的指定其他的应用。

在构建镜像时指定默认命令是一种很普遍的做法，这样可以简化容器的启动，也为镜像指定了默认的行为，从侧面阐述了镜像的用途。

### docker image rm

用于删除镜像，删除操作会在当前主机上删除该镜像以及相关的镜像层。但是如果某个镜像层被多个镜像共享，那么只有当全部依赖该镜像层的镜像都被删除后，该镜像层才会被删除。

当镜像存在关联的容器，并且容器处于运行（Up）或者停止（Exited）状态时，不允许删除该镜像。

#### 命令格式

```
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```

#### 命令说明

命令中的<镜像>可以是`镜像短 ID`、`镜像长 ID`、`镜像名` 或者 `镜像摘要`。

可以通过`docker image ls`查看镜像的ID，然后摘取ID的前几位可以区分不同镜像的字符即可。

#### 综合示例

```shell
$ docker image ls
REPOSITORY             TAG           IMAGE ID            CREATED             SIZE
centos                 latest        0584b3d2cf6d        3 weeks ago         196.5 MB
redis                  alpine        501ad78535f0        3 weeks ago         21.03 MB
```

示例一，通过ID删除 `redis:alpine` 镜像，可以执行：

```shell
$ docker image rm 501
Untagged: redis:alpine
...
```

示例二，使用`镜像名`，也就是 `<仓库名>:<标签>`，来删除镜像。

```shell
$ docker image rm centos
Untagged: centos:latest
```

示例三，更精确的是使用 `镜像摘要` 删除镜像。

```shell
$ docker image ls --digests
REPOSITORY                  TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
node                        slim                sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228   6e0c4c8e3913        3 weeks ago         214 MB

$ docker image rm node@sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228
Untagged: node@sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228
```

**示例四，删除Docker主机上的全部镜像**：

```shell
$ docker image ls -q
340bd30ab40hf30f
ed03f530fh0q202j
$ docker image rm $(docker image ls -q) -f
```

上述操作，先通过`docker image ls -q`命令获取全部镜像ID，然后将其传给`docker image rm`命令执行删除镜像的操作。

### docker image tag

用于为指定的镜像添加一个额外的标签。

#### 命令格式

```
docker image tag <current-tag> <new-tag>
```

#### 命令说明

current-tag：使用`docker image build`命令构建镜像时指定的标签。

该命令不需要覆盖已经存在的标签。

#### 综合示例

```shell
$ docker image tag web:latest smallz/web:latest
```

### docker image push

用于将镜像推送到Docker Hub中。

```shell
$ docker image push smallz/web:latest
```

### docker image history

显示在构建镜像的过程中，都执行了哪些指令。

```shell
$ docker image history web:latest
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
dab74052193a        15 hours ago        /bin/sh -c #(nop)  ENTRYPOINT ["node" "./app…   0B
224a3531a754        15 hours ago        /bin/sh -c #(nop)  EXPOSE 8080                  0B
c3ec0a945100        15 hours ago        /bin/sh -c npm install                          20.9MB
97bbd9fb9a28        15 hours ago        /bin/sh -c #(nop) WORKDIR /src                  0B
27359f55024b        15 hours ago        /bin/sh -c #(nop) COPY dir:8abfedb4af4f697c8…   2.29kB
0e580c769e09        15 hours ago        /bin/sh -c apk add --update nodejs nodejs-npm   51MB
0fb275ef5c63        15 hours ago        /bin/sh -c #(nop)  LABEL maintainer=nigelpou…   0B
a24bb4013296        7 weeks ago         /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>           7 weeks ago         /bin/sh -c #(nop) ADD file:c92c248239f8c7b9b…   5.57MB
```

通过该命令输出的内容，每一行都对应了Dockerfile中的一条指令，顺序是从下到上。

其中SIZE列对应的数值不为零的指令，都会新建镜像层。

### docker image命令的配合使用

示例一，删除所有仓库名为 `redis` 的镜像：

```shell
$ docker image rm $(docker image ls -q redis)
```

示例二，删除所有在 `mongo:3.2` 之前的镜像：

```shell
$ docker image rm $(docker image ls -q -f before=mongo:3.2)
```



## docker container（容器）

容器是镜像的运行时实例，容器会共享其所在主机的操作系统的内核。

容器会随着其中运行应用的退出而终止。

容器如果不运行任何进程则无法存在，如果容器内的进程或者主进程都被杀死，那么这个容器也将被杀死。

按下【Ctrl+P+Q】组合键会退出当前容器，但不会终止容器运行，将会切回到Docker主机的Shell，并保持容器在后台运行。

容器中的全部配置和内容不会随着容器的停止而消失，他们仍然保存在Docker主机的文件系统之中，只要不删除容器，仍然可以随时重新启动并恢复。停止容器运行并不会损毁容器或者其中的数据。虽然如此，但必须指出的是：卷（volume）才是在容器中存储持久化数据的首选方式，如果将容器数据存储在卷中，数据也会被保存下来。

### 容器的重启策略

- always：使用该策略运行的容器，除非显式使用`docker container stop`命令将容器明确停止，否则该策略会一直尝试重启处于停止状态的容器。示例：

  ```shell
  $ docker container run --name neversaydie -it --restart always alpine sh
  / # exit
  $ docker container ls -a
  CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
  5036699a89d3        alpine              "sh"                33 seconds ago      Up 9 seconds                            neversaydie
  ```

  上述示例使用restart always策略启动了一个新的交互式容器，同时在命令中指定运行Shell进程，当间隔一段时间输入“exit”退出Shell时，该容器会自动被杀死。但是由于指定了--restart为always策略，该容器会在被杀死后又再次自动被重启，因此通过`docker container ls -a`命令，可以看到容器的启动时间远小于创建时间。

  只有在执行`docker container stop`命令显式的停止该容器时，容器才处于Stopped（Exited）状态。虽然如此，但是当整个Docker daemon重启时，该策略的容器仍会重新启动。

- unless-stopped：与always的最大区别是，指定了--restart为unless-stopped策略的并处于Stopped（Exited）状态的容器，不会在Docker daemon重启的时候被重启。

  下面分别创建两个容器，其中一个策略指定为always，另一个指定为unless-stopped，将这两个容器通过`docker container stop`命令同时停止，接着重启Docker。

  创建两个容器，并在启动时指定不同的策略：

  ```shell
  $ docker container run -d --name myalways --restart always alpine sleep ld
  16c666d1d33da3f5defbdfa015c9ff063cbdb53482e8e723b84ae8974a5e3e56
  $ docker container run -d --name myunlessstopped \
  --restart unless-stopped alpine sleep ld
  3c8da19b9a8354a9fa3318c504c2509f401e38f78ec48ab7675647e669be6982
  ```

  同时停止两个容器：

  ```shell
  $ docker container stop myalways myunlessstopped
  ```

  重启Docker：

  ```shell
  $ sudo systemctl restart docker
  ```

  检测状态：

  ```shell
  $ docker container ls -a
  STATUS                        PORTS               NAMES
  Exited (1) 2 minutes ago                          myunlessstopped
  Restarting (1) 1 second ago                       myalways
  ```

  结果：启动时指定了--restart always策略的容器已经重启，但unless-stopped容器没有重启。

- on-failed：指定了该策略的容器，会在退出容器并且返回值不是0的时候，重启容器。就算容器处于Stopped状态，在Docker daemon重启的时候，容器也会被重启。



### docker container ls

列出所有在运行（UP）状态的容器。

```shell
$ docker container ls
CONTAINER ID  IMAGE         COMMAND         CREATED        STATUS       PORTS NAMES
77b2dc01fe0f  ubuntu:18.04  /bin/sh -c 'while tr  2 minutes ago  Up 1 minute        agitated_wright
```

可以指定`-a`参数，让Docker列出所有容器，包括那些处于停止（Exited）状态的。

```shell
$ docker container ls -a
```

### docker container run

docker container run命令用于启动新的指定镜像的容器，它告诉Docker daemon启动新的容器。

当执行这个命令时，Docker daemon会先搜索Docker本地缓存，观察是否有命令所请求的镜像，如果本地没有该镜像，Docker会在Docker Hub中检查是否存在对应镜像，有的话就拉取到本地，并存储在本地缓存中，然后创建容器，并在其中运行指定的应用。

#### 命令格式

```
docker container run [options] <iamge> <app>
```

#### 命令说明

- image：表示启动所需的镜像，指定的格式和镜像拉取时相同，通常是仓库名[:标签]的形式。
- app：启动容器后将要运行的应用。

#### 选项描述

- -it：这是两个参数，一个是 `-i`：交互式操作，一个是 `-t` 终端。其中，`-t` 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上， `-i` 则让容器的标准输入保持打开。-it表示将当前终端连接到容器的Shell终端之上。
- `--rm`：这个参数是说容器退出后随之将其删除。默认情况下，为了排障需求，退出的容器并不会立即删除，除非手动 `docker rm`。
- --name：指定新建的容器的名称。
- -d：表示后台模式，告知容器在后台运行。即：指定启动的容器在后台运行，这种后台启动的方式不会将当前终端连接到容器当中。
- --restart：指定容器启动时采用哪种重启策略。
- -P：将主机的端口与容器内的端口进行映射。

#### 综合示例

示例一，启动ubuntu:18.04镜像容器，并通过bash进行交互式操作，由于使用了--rm，因此退出容器后将会立即删除该容器（这里只是随便执行个命令，看看结果，不需要排障和保留结果，因此使用 `--rm` 可以避免浪费空间）：

```shell
$ docker run -it --rm \
    ubuntu:18.04 \
    bash
```

示例二，直接从镜像来启动容器，并使用-it参数将shell切换到容器终端，该操作会直接进入到容器内部。其中-it参数告诉Docker开启容器的交互模式并将用户当前的Shell连接到容器终端。

```shell
$ docker container run -it ubuntu:latest /bin/bash
```

如果要退出容器终端，可以使用快捷键：【Ctrl+P,Q】，该组合键可以在退出容器的同时还保持容器运行（在容器内部使用该操作可以退出当前容器，但不会杀死容器进程）。这样Shell就会返回到Docker主机终端。

示例三，创建一个名称为percy的容器：

```shell
$ docker container run --name percy -it ubuntu:latest /bin/bash
```

示例四，创建一个Web服务器镜像容器，并将Docker主机的端口映射到容器内：

```shell
$ docker container run -d --name webserver -p 80:8080 \
nigelpoulton/pluralsight-docker-ci
```

上述示例中的-p参数将Docker主机的80端口映射到容器内的8080端口，这意味着当有流量访问主机80端口的时候，流量会直接映射到容器内的8080端口。（原因是该镜像的容器在启动的时候会运行一个Web服务，监听的是容器内的8080端口，因此可以通过Docker主机的浏览器来访问该容器，只需要在浏览器中指定Docker主机的IP地址和默认的80端口即可）

### docker container exec

该命令允许用户在运行状态的容器中，启动一个新进程，通常用于将Shell连接到一个运行中的容器终端。该命令会创建新的Bash或者PowerShell进程并且连接到容器，这意味着在当前Shell输入exit并不会导致容器终止，因为原Bash或者PowerShell进程还在运行当中。

#### 命令格式

```
docker container exec <options> <container-name or container-id> <command/app>
```

#### 命令说明

options：选项参数

container-name和container-id：这两个值都可以通过`docker container ls`命令，得到。

command/app：指定进入到容器后需要运行的终端程序。

#### 综合示例

示例一，将Shell连接到当前运行的容器名词为gracious_faraday的容器：

```shell
$ docker container ls
CONTAINER ID  IMAGE	        COMMAND		CREATED		   STATUS		NAMES
9721c0028dca  ubuntu:latest "/bin/bash" 8 minutes ago  Up 8 minutes gracious_faraday
angyony@ubuntu:~$ docker container exec -it gracious_faraday bash
```

在示例中，将本地Shell连接到容器是通过-it参数实现的。若要退出容器，使用组合键：【Ctrl+P,Q】

示例二，连接到指定名称的容器：

```shell
$ docker container exec -it percy bash
```

### docker container start

该命令会重启处于停止（Exited）状态的容器。（用于容器的重写启动，将一个已经停止的容器重新启动。）

#### 命令格式

```
docker container start <container-name or container-id> 
```

#### 综合示例

重新启动名称为percy的容器：

```shell
$ docker container start percy
```

### docker container logs

获取容器的输出信息。

```shell
$ docker container logs [container ID or NAMES]
hello world
hello world
```

### docker container stop

该命令会停止运行中的容器，并将状态置为Exited（0）。

#### 命令格式

```
docker container stop <container-name or container-id> 
```

#### 综合示例

终止一个运行中的容器：

```shell
$ docker container stop gracious_faraday
或
$ docker container stop 302e2f
```

### docker container rm

杀死并删除容器。强烈建议在使用该命令前，先使用`docker container stop`停止容器。而不是使用`docker container rm <container> -f`的方式暴力的销毁容器。

#### 命令格式

```
docker container rm <container-name or container-id> [options]
```

#### 选项描述

- -f：表示强制执行，即使处于运行状态的容器也会被删除，通常不建议这么做，而是采用先停止再删除。这样可以给容器中运行的应用或进程一个停止运行并清理残留数据的机会。

#### 综合示例

示例一，删除指定名称的容器：

```shell
$ docker container rm gracious_faraday
```

可以通过运行`docker container ls -a`命令来确认容器是否已经被成功删除。

**示例二，在Docker主机上删除全部容器，无论容器是否运行或停止，都将被删除并从系统中移除：**

```shell
$ docker container rm $(docker container ls -aq) -f
```

将`$(docker container ls -aq)`作为参数传递给`docker container rm`命令，等价于将系统中每个容器的ID传给该命令。

注意：虽然该命令高效，但是一定不能在生产环境系统或者运行着重要容器的系统上执行。

### docker container inspect

该命令会显示容器的配置细节和运行时信息。该命令接收容器名称和容器ID作为主要参数。

```shell
$ docker container inspect 7ab70
或
$ docker container inspect wy
```



## docker system

### docker system df 

用于查看镜像、容器、数据卷所占用的空间。

#### 命令格式

```shell
docker system df
```

#### 示例

```shell
$ docker system df

TYPE			TOTAL		ACTIVE		SIZE			RECLAIMABLE
Images      	24      	0           1.992GB     	1.992GB (100%)
Containers      1           0           62.82MB         62.82MB (100%)
Local Volumes   9           0           652.2MB         652.2MB (100%)
Build Cache                             0B              0B
```

上面列表中显示的最后一条镜像为虚悬镜像（见上文说明）。



## docker ps 

docker ps：列出container



## docker service 

### docker service create



## docker search

### docker search

docker search命令允许通过CLI的方式搜索Docker Hub。

#### 命令格式

```
docker search <searchvalue> [options]
```

#### 命令说明

searchvalue：表示将要搜索的内容，该命令会搜索所有“NAME”字段中包含特定字符串的仓库。“NAME”字段是仓库名称，包含了DockerID，或者非官方仓库的组织名称。

##### 参数描述

- --filter
  - is-official：是否只显示官方镜像。
  - is-automated：是否只显示自动创建的仓库。
- --limit：默认情况下，Docker只返回25行结果。可以指定--limit参数来增加返回内容行数，最多为100行。

#### 综合示例

示例一：在默认的镜像仓库服务Docker Hub中，搜索NAME”字段中包含“nigelpoulton”的所有仓库。

```shell
$ docker search nigelpoulton
NAME                                 DESCRIPTION     STARS    OFFICIAL    AUTOMATED
nigelpoulton/pluralsight-docker-ci   Simple …   	 23                   [OK]
nigelpoulton/tu-demo                 Voting…   		 12
nigelpoulton/ctr-demo                Web service…    3
...
```

注意：上面返回的镜像中既有官方的也有非官方的。如果需要只显示官方镜像，需要使用`--filter`选项。

示例二：列出所有仓库名称中包含“alpine”的官方镜像：

```shell
$ docker search alpine --filter "is-official=true"
```

示例三：只显示自动创建的仓库，并且仓库名称中包含“nigelpoulton”的镜像：

```shell
$ docker search nigelpoulton --filter "is-automated=true"
```



## docker network



### docker network ls

查看网络列表。

```shell
$ docker network ls
NETWORK ID          NAME                      DRIVER              SCOPE
cc246b1076cd        bridge                    bridge              local
953f3bd3dcb4        wyapp_wy-net   			  bridge              local
...
```



## docker volume



### docker volume ls

查看卷列表。

```shell
$ docker volume ls
DRIVER              VOLUME NAME
local               7f26bd74922581b42b224b44a81fb642dfd2684106a7909b148fee4bebee3875
local               wyapp_wy-vol
```



## docker-compose

执行docker-compose相关的命令时，必须先进入到YAML文件（docker-compose.yml）所在的目录中。

### docker-compose ps

查看Docker Compose应用的状态，该命令会列出Compose应用中的各个容器。

```shell
$ docker-compose ps
     Name                   Command               State           Ports
--------------------------------------------------------------------------------
wyapp_redis_1    docker-entrypoint.sh redis ...   Up      6379/tcp
wyapp_web-fe_1   python app.py                    Up      0.0.0.0:5000->5000/tcp
```

输出那个会显示容器名称、其中运行的Command、当前状态以及其监听的网络端口。

### docker-compose up

使用`docker-compose up`命令用于部署一个Compose应用，它会启动一个Compose应用，该命令会构建所需的镜像，并创建网络和卷，并启动容器。

默认情况下，该命令会读取名为 docker-compose.yml或docker-compose.yaml的文件。

#### 命令格式

```
docker-compose [选项][YAML文件名] up [选项]
```

#### 命令说明

通常在YAML文件所在的目录中执行该命令，默认情况下，该命令会查找所在目录中名为docker-compose.yml或docker-compose.yaml的Compose文件。如果Compose文件是其他文件名，则需要通过`-f`参数来指定。

#### 选项描述

- -d：指定在后台启动应用。
- -f：指定其他Compose文件名。
- &：使用 & 将终端窗口返回，因此它运行在前台。

#### 综合示例

示例一，基于名为prod-equus-bass.yml的Compose文件部署应用：

```shell
$ docker-compose -f prod-equus-bass.yml up
```

示例二，在后台启动Compose应用：

```shell
$ docker-compose up -d
$ docker-compose -f prod-equus-bass.yml up -d
```

### docker-compose down

停止和关闭Docker Compose应用。该命令会停止并且也会删除运行中的Compose应用，它会删除容器和网络，但是不会删除卷和镜像。

```shell
$ docker-compose down
```

注意：使用`docker-compose down`停止应用时，vol卷并没有被删除。写到卷上的所有数据都会保存下来。

### docker-compose top

该命令会列出各个服务（容器）内运行的进程。

```shell
$ docker-compose top
wyapp_redis_1
  UID      PID    PPID   C   STIME   TTY     TIME         CMD
------------------------------------------------------------------
systemd+   4165   4131   0   04:29   ?     00:00:00   redis-server

wyapp_web-fe_1
UID    PID    PPID   C   STIME   TTY     TIME                    CMD
------------------------------------------------------------------------------------
root   4179   4141   0   04:29   ?     00:00:00   python app.py
root   4284   4179   0   04:29   ?     00:00:01   /usr/local/bin/python /code/app.py
```

上述结果中，PID编号是在Docker主机上（而不是容器内）的进程ID。

### docker-compose stop

该命令会停止Docker Compose应用相关的所有容器，但不会删除它们及资源。被停止的应用可以通过 docker-compose restart 命令重新启动。

```shell
$ docker-compose stop
Stopping wyapp_web-fe_1 ... done
Stopping wyapp_redis_1  ... done
```

注意：如果用户在停止该应用后对其进行了变更，那么变更的内容不会反映在重启后的应用中，这时需要重新部署应用使变更生效。

### docker-compose rm

该命令用于删除已停止的Compose应用。它会删除容器和网络，但是不会删除卷和镜像。

```shell
$ docker-compose rm
Going to remove wyapp_web-fe_1, wyapp_redis_1
```

注意：该命令只会删除应用相关的容器和网络，不会删除卷和镜像，也不会删除应用源码（YAML文件所在的目录中的源码）。

### docker-compose restart

docker-compose restart 命令会重启已停止的Compose应用。

```shell
$ docker-compose restart
Restarting wyapp_web-fe_1 ... done
Restarting wyapp_redis_1  ... done
```

注意：如果用户在停止该应用后对其进行了变更，那么变更的内容不会反映在重启后的应用中，这时需要重新部署应用使变更生效。