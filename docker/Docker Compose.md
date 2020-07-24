# Docker Compose

Docker Compose是一个需要在Docker主机上进行安装的外部Python工具。使用它时，首先编写定义多容器（多服务）应用的YAML文件，然后将其交由docker-compose命令处理，Docker Compose就会基于Docker引擎API完成应用的部署。

Docker Compose通过一个声明式的配置文件（YAML文件）描述整个应用，从而使用一条命令完成部署。

安装Docker Compose，可参考官方文档：https://docs.docker.com/compose/install/



## Compose文件

Docker Compose使用YAML文件来定义多服务的应用，文件名默认为docker-compose.yml，也可以使用`-f`参数指定具体文件。YAML是JSON的一个子集，因此也可以使用JSON。

docker-compose.yml文件示例：

```yaml
version: "3.5"
services:
  web-fe:
    build: .
    command: python app.py
    ports:
      - target: 5000
        published: 5000
    networks:
      - wy-net
    volumes:
      - type: volume
        source: wy-vol
        target: /code
  redis:
    image: "redis:alpine"
    networks:
      wy-net:

networks:
  wy-net:

volumes:
  wy-vol:
```

### YAML文件中的一级key

YAML文件中常见的一级key有：

- version
- services
- networks
- volumes

#### version

version是必须指定的，而且总是位于文件的第一行，它定义了Compose文件格式的版本，建议使用最新版本。

注意：version并非定义Docker Compose 或 Docker 引擎的版本号，而是[Compose file format](https://docs.docker.com/compose/compose-file/compose-versioning/#compatibility-matrix) 的版本。

#### services

services用于定义不同的应用服务。上述示例中，分别定义了名为web-fe和redis的两个应用程序服务，同时它们也是两个二级key。

Docker Compose 会将 services 定义的每个应用程序服务部署在各自的容器中。并且会使用它们各自的 Key 作为容器名字的一部分。例如上述示例中定义了两个 key：web-fe 和 redis，因此 Docker Compose 会部署两个容器，一个容器的名字中会包含web-fe，另一个容器的名字中会包含redis。

使用 services 定义应用服务时，常见使用的指令：

- build：上述示例中设为了`.`，指定 Docker 基于当前目录（`.`）下的Dockerfile中定义的指令来构建一个新镜像。该镜像会被用于启动该服务的容器。

- command：上述示例中设为了`python app.py`，表示指定Docker在容器中执行名为app.py 的Python脚本作为主程序。因此镜像中必须包含app.py文件以及Python。【其实本例并不需要配置 command: python app.py，因为镜像的Dockerfile已经将python app.py定义为了默认的启动程序，这里只是为了说明如何执行而显式指出的，它将覆盖Dockerfile中配置的CMD指令】

- ports：在上述示例中，指定Docker将容器内（-target）的5000端口映射到主机（published）的5000端口。

- networks：使得Docker可以将服务连接到指定的网络上，这个网络应该是已经存在的，或者是在networks一级key中定义的网络。对于Overlay网络来说，它还需要定义一个attachable标志，这样独立的容器才可以连接上它（这时 Docker Compose 会部署独立的容器而不是 Docker 服务）。

  并且在第二个redis应用服务中，也指定了相同的networks，由于两个服务都连接到wy-net网络，因此它们可以通过名称解析到对方的地址。

- volumes：上述示例中，指定 Docker 将 wy-vol 卷（source:）挂载到容器内的 /code（target:）。wy-vol 卷应该是已存在的，或者是在文件下方的 volumes 一级 key 中定义的。

- image：指定应用服务启动容器时，基于的基本镜像。上述示例中，Docker基于 redis:alpine 镜像启动一个独立的名为redis的容器。

#### networks

networks用于指引Docker创建新的网络。上述示例中，定义了一名为 wy-net 的网络。

默认情况下，Docker Compose会创建bridge网络，这是一种单主机网络，只能够实现同一主机上容器的连接。当然也可以使用driver属性来指定不同的网络类型。

示例，创建一个名为over-net的Overlay网络，它可以允许独立的容器连接到该网络上：

```yaml
networks:
	over-net:
	driver: overlay
	attachable: true
```

#### volumes

volumes用于指引Docker来创建新的卷。上述示例中，定义了一名为 wy-vol 的卷。

在Docker主机对卷中文件的修改，会立刻反应到应用中。



## 使用 Docker Compose 部署应用

Docker Compose会使用 YAML文件（docker-compose.yml）所在的目录名作为项目名称，并将该目录作为构建上下文。示例中，YAML文件所在的目录名为wy-app，因此Docker Compose会在所有的资源名称中加上前缀wy-app_。

进入到wy-app目录，执行下述命令：

```shell
$ docker-compose up &
```

构建完应用之后，可以使用`docker image ls`命令，查看Docker Compose创建的镜像、容器、网络和卷。

```shell
$ docker image ls
REPOSITORY      TAG                 IMAGE ID            CREATED             SIZE
wyapp_web-fe    latest              88c3524cfe02        26 minutes ago      84.5MB
...
```

注意：Docker Compose会将项目名称（wyapp）和Compose文件中定义的资源名称（web-fe）连起来，作为新构建的镜像的名称。Docker Compose部署的所有资源的名称都会遵循这一规范。

通过`docker container ls`命令查看生成后的容器：

```shell
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
d314e66dda45        redis:alpine        "docker-entrypoint.s…"   31 minutes ago      Up 31 minutes       6379/tcp                 wyapp_redis_1
32c6f976c782        wyapp_web-fe        "python app.py"          31 minutes ago      Up 31 minutes       0.0.0.0:5000->5000/tcp   wyapp_web-fe_1

```

可以看到每个容器的名称都以项目名称（所在目录名称wyapp）为前缀。此外，它们还都以一个数字为后缀用于标识容器实例序号——因为Docker Compose允许扩缩容。



## 使用 Docker Compose 管理应用

### 停止 Docker Compose

使用`docker-compose down`命令停止Docker Compose应用：

```
docker-compose down
```

执行该命令后，wy-net网络将被删除，对应的docker-compose up 进程也将退出。

注意：wy-vol卷不会被删除，因为卷是用于数据的长期持久化存储的，卷的生命周期与相应的容器是完全解耦的，执行`docker volume ls`命令依然可见该卷的信息，并且写在卷上的所有数据都会保存下来。

同样，执行`docker-compose up`过程中拉取或构建的镜像也会保留在系统中。

### 启动 Docker Compose

在后台启动Docker Compose，执行下述命令：

```
docker-compose up -d
```

### 查看 Docker Compose

查看应用的状态，执行下述命令：

```
docker-compose ps
```

输出中会显示容器名称、其中运行的Command、当前状态以及其监听的网络端口。

### 查看各个服务（容器）内运行的进程

查看各个服务（容器）内运行的进程，执行下述命令：

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

### 停止 Docker Compose

停止Docker Compose应用，执行下述命令：

```shell
$ docker-compose stop
Stopping wyapp_web-fe_1 ... done
Stopping wyapp_redis_1  ... done
```

该命令只会停止应用，并不会删除资源。

再次运行`docker-compose ps`查看状态：

```shell
$ docker-compose ps
     Name                   Command               State    Ports
----------------------------------------------------------------
wyapp_redis_1    docker-entrypoint.sh redis ...   Exit 0
wyapp_web-fe_1   python app.py                    Exit 0
```

通过结果可以看到：停止Compose应用，并不会在系统中删除对应用的定义，而仅将应用的容器停止。（可以通过`docker container ls -a`命令进行验证）

### 删除已停止的Compose应用

对于已停止的Compose应用，可以使用docker-compose rm 命令来删除。

```shell
$ docker-compose rm
```

执行该命令后，只会删除应用相关的容器和网络，不会删除卷和镜像，也不会删除应用源码（YAML文件所在的目录中的源码）。

### 重启 Docker Compose

执行 docker-compose restart 命令重启应用。

```shell
$ docker-compose restart
Restarting wyapp_web-fe_1 ... done
Restarting wyapp_redis_1  ... done
```

再次运行`docker-compose ps`查看状态：

```shell
$ docker-compose ps
     Name                   Command               State           Ports
--------------------------------------------------------------------------------
wyapp_redis_1    docker-entrypoint.sh redis ...   Up      6379/tcp
wyapp_web-fe_1   python app.py                    Up      0.0.0.0:5000->5000/tcp
```

注意：如果用户在停止该应用后对其进行了变更，那么变更的内容不会反映在重启后的应用中，这时需要重新部署应用使变更生效。