# Docker 快速安装 Mysql



### 重启 docker

```shell
systemctl restart docker
```



### 运行并安装 MySql

```
docker run \
--detach \
--name=mysql02 \
--env="MYSQL_ROOT_PASSWORD=root123456@" \
--publish 7306:3306 \
--volume=/root/docker/mysql02/conf.d:/etc/mysql/conf.d \
--volume=/root/docker/mysql02/data:/var/lib/mysql \
mysql/mysql-server:latest \
--character-set-server=utf8 \
--collation-server=utf8_general_ci
```

上述命令已经包含了拉取镜像、设置密码、挂载目录、设置编码格式等基本设置。



### 修改 Docker中的 MySql 外部连接策略（允许外部连接）：

~~方式一：修改全局配置文件/etc/mysql/my.conf，~~

~~执行下述命令：docker run -it mysql /bin/bash~~

找到bind-address = 127.0.0.1这一行

改为bind-address = 0.0.0.0即可，如果是容器挂载文件，需要重启容器才会生效。

方式二（推荐）：进入容器修改

```
docker exec -it mysql02 mysql -uroot -p
```

输入密码后，执行下述Mysql查询语句：

```
use mysql;
update user set host='%' where user='root'
flush privileges;
```



### 测试连接

如果是云服务器，需要看是否开启了端口。

