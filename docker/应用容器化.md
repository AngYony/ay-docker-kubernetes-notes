# 应用的容器化

容器化（Containerizing）：将应用整合到容器中并且运行起来的过程称为“容器化”。也称为Docker化。



## Dockerfile

在Docker当中，包含应用文件的目录通常被称为构建上下文，通常将Dockerfile放到构建上下文的根目录下。

注意文件名严格区分大小写，必须是大写D开头的，名称为Dockerfile。

Dockerfile文件的作用：

- 包含了对当前应用的描述
- 指导Docker完成应用的容器化（创建一个包含当前应用的镜像）

Dockerfile文件中的细节部分：

- Dockerfile中的注释行，都是以#开头的。
- Dockerfile中的每一行都是一条指令（Instruction），注释除外。
- Dockerfile中的指令是不区分大小写的，但通常都采用大写的方式。
- `Docker image build`命令会按行来解析Dockerfile中的指令，并顺序执行。
- 部分指令会在镜像中创建新的镜像层，其他指令只会增加或修改镜像的元数据信息。

新增镜像层的指令有：

- FROM
- RUN
- COPY

新增元数据的指令有：

- EXPOSE
- WORKDIR
- ENV
- ENTERPOINT

如果指令的作用是向镜像中新增新的文件或者程序，那么这条指令就会新建镜像层；如果只是告诉Docker如何完成构建或者如何运行程序，那么就只会增加镜像的元数据。

简单的Dockerfile文件内容示例：

```dockerfile
# Test web-app to use with Pluralsight courses and Docker Deep Dive book
# Linux x64
FROM alpine

LABEL maintainer="nigelpoulton@hotmail.com"

# Install Node and NPM
RUN apk add --update nodejs nodejs-npm

# Copy app to /src
COPY . /src

WORKDIR /src

# Install dependencies
RUN  npm install

EXPOSE 8080

ENTRYPOINT ["node", "./app.js"]
```

### FROM

FROM指令位于Dockerfile文件中的第一行。

FROM指令指定的镜像，会作为当前镜像的一个基础镜像层，当前应用的剩余内容，将会作为新增镜像层添加到基础镜像层之上。

通常使用FROM指令引用官方基础镜像，因为官方的镜像通常会遵循一些最佳实践，并且可以避免一些已知的问题。

### LABEL

LABEL标签主要用来进行额外的说明描述，它其实是一个键值对，在一个镜像当中可以通过增加标签的方式来为镜像添加自定义元数据。

### RUN

RUN指令会在FROM指定的alpine基础镜像之上，新建一个镜像层来存储这些安装内容。

每一个RUN指令都会新增一个镜像层，因此，通常的做法是将多个命令包含在同一个RUN指令中。（通过使用`&&`连接多个命令以及使用反斜杠（`\`）换行的方法，将多个命令包含在一个RUN指令中）

### COPY

COPY指令将应用相关文件从构建上下文复制到当前镜像中，并且新建一个镜像层来存储。

COPY --from：将从之前阶段构建的镜像中**仅复制生产环境相关的应用代码**，而不会复制生产环境不需要的构件。

通常使用COPY指令将应用代码赋值到镜像中。

### WORKDIR

WORKDIR指令为Dockerfile中尚未执行的指令设置工作目录。该目录与镜像相关，并且会作为元数据记录到镜像配置中，但不会创建新的镜像层。

### ENTRYPOINT

ENTRYPOINT指令指定当前镜像的入口程序，ENTRYPOINT指定的配置信息也是通过镜像元数据的形式保存下来，而不是新增镜像层。

ENTRYPOINT指令用于指定镜像以容器方式启动后默认运行的程序。

### EXPOSE

EXPOSE指令用于记录应用所使用的网络端口。

### ENV

ENV：设定环境变量

### ONBUILD

### HEALTHCHECK

### CMD

CMD：执行命令

### ADD

ADD：添加文件

### MAINTAINER

MAINTAINER：维护者

### USER

USER：指定用户

### VOLUME

VOLUME：mount point







## 构建镜像

通常在Dockerfile文件所在的目录中，执行`docker image build`命令来构建镜像。

```shell
$ docker image build -t web:latest .
```

上述命令会构建并生成一个名为web:latest的镜像，注意：命令最后的点（.）表示Docker在进行构建的时候，使用当前目录作为构建上下文。

命令执行结束后，可以通过`docker image ls`命令来查看本地Docker镜像库是否包含了刚才构建的镜像。

也可以通过`docker image inspect`命令来查看构建的镜像的配置是否正确。

还可以通过`docker image history`命令来查看在构建镜像的过程中都执行了哪些指令。

通过`docker image build`命令可以看到基本的构建过程：

1. 运行临时容器 
2. 在该容器中运行Dockerfile中的指令
3. 将指令运行结果保存为一个新的镜像层
4. 删除临时容器

### 多阶段构建

使用RUN执行一个命令时，可能会拉取一些构建工具，这些工具可能会留在镜像中移交至生产环境，很显然这是不合适的。通常不会在构建完成后才进行清理，而是在构建的过程中规避或解决这些问题。

改善这一问题有多种方式：

- 建造者模式（可行，但较复杂）
- 多阶段构建（在不增加复杂性的情况下优化构建过程）

多阶段构建方式使用一个Dockfile，其中包含多个FROM指令。

每一个FROM指令都是一个新的单独的构建阶段（Build Stage），并且可以方便的复制之前阶段的构件。

各个阶段在内部从0开始编号，可以通过FROM指令为每个阶段指定一个便于理解的名字。

```dockerfile
FROM node:latest AS storefront
....
FROM maven:latest AS appserver
...
FROM java:8-jdk-alpine AS production
...
```

上述有3个FROM指令，阶段0、阶段1、阶段2，分别叫作storefront、appserver、production。

### 最佳实践

#### 利用构建缓存

Docker的构建过程利用了缓存机制。

docker image build命令会从顶层开始解析Dockerfile中的指令并逐行执行，而对每一条指令，Docker都会检查缓存中是否已经有与该指令对应的镜像层。如果有，即为缓存命中，并且会使用这个镜像层；如果没有，则缓存未命中，Docker会基于该指令构建新的镜像层。缓存命中能够显著加快构建过程。

一旦有指令在缓存中未命中（没有该指令对应的镜像层），则后续的整个构建过程将不再使用缓存，因此，在编写Dockerfile 时需要特别注意这一点，尽量将易于发生变化的指令置于Dockerfile文件的后方执行。

通过对docker image build命令加入`--nocache=true`参数，可以强制忽略对缓存的使用。

COPY和ADD指令会检查复制到镜像中的内容自上一次构建之后是否发生了变化，如果发生了变化就认为缓存无效并构建新的镜像层。

#### 合并镜像

合并镜像利弊参半，非最佳实践。

优点：将多个镜像层合并为一个。

缺点：合并的镜像将无法共享镜像层，导致存储空间低效利用。push和pull操作时，镜像体积更大。

执行`docker image build`命令时，可以通过增加`--squash`参数来创建一个合并的镜像。

#### 使用 no-install-recommends

在构建Linux镜像时，若使用的是APT包管理器，则应该在执行apt-get install命令时增加no-install-recommends参数。这能够确保APT仅安装核心依赖（Depends中定义）包，而不是推荐和建议的包。这样能够减少不必要包的下载数量。

#### 不要安装MSI（Windows）

在构建Windows镜像时，尽量避免使用MSI包管理器。因其对空间的利用率不高，会大幅增加镜像的体积。



## 推送镜像

`docker image push`命令默认的推送的公共镜像仓库服务地址是Docker Hub。

### 第一步：登录Docker Hub

执行docker login命令，输入用户名和密码。

```shell
$ docker login
```

### 第二步：为镜像打标签

Docker在镜像推送的过程中需要如下信息，因此需要先设定再执行推送。需要的信息有：

- Registry（镜像仓库服务）：默认为docker.io。
- Repository（镜像仓库）：需要显式指定，从被推送镜像中的REPOSITORY属性值获取。
- Tag（镜像标签）：默认为latest。

为镜像打标签命令格式：

```
docker image tag <current-tag> <new-tag>
```

该命令的作用是为指定的镜像添加一个额外的标签，并且不需要覆盖已经存在的标签。

示例：

```shell
$ docker image tag web:latest smallz/web:latest
```

### 第三步：推送到Docker Hub

```shell
$ docker image push smallz/web:latest
```

确定镜像所要推送的目的仓库：<span style="color:blue">docker.io</span>/<span style="color:red;">smallz/web</span>:<span style="color:blue">latest</span>，其中蓝色字体为默认值，而红色字体的内容可以通过`docker image ls`命令进行查看。



## 运行镜像和容器

使用`docker container run`命令启动容器。





