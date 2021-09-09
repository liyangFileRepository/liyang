# Docker


## 1、Docker概述

DevOps

开发 --- 运维。最容易产生的问题：我在我的电脑上可以运行！

发布一个项目[ jar+（Redis MySQL JDK ES）]，

Docker在运维的工作中主要是搭建各种环境。

## 2、Docker安装

> 环境准备

```
Linux 、内核3.0以上

https://docs.docker.com
```

> 设置仓库镜像

```shell
Yum-config-manager \

--add-repo \

阿里云镜像
```

> 卸载的步骤

**1、卸载Docker**

```shell
yum remove docker-ce docker-ce-cli containers.io
```

**2、删除资源**

```shell
rm -rf /var/lib/docker
```

---

### 底层原理

**Docker是怎么工作的？**

Docker是一个Client- server结构的系统，Docker的守护进程运行在主机上，通过socket从客户端访问！

DockerServer接收到Docker- Client的指令，就会执行这个命令！

## 3、Docker命令

### 帮助命令

```shell
docker version					#显示docker的版本信息
docker info							#显示docker的系统信息，包括镜像和容器的数量
docker 命令  --help			 #查看该命令的使用信息
```

### 镜像命令

```shell
docker images
# 解释
REPOSITORY		镜像的仓库源
TAG						镜像的标签
IMAGE ID			镜像的ID
CREATERD			镜像的创建时间
SIZE					镜像的大小

# 可选项
	-a, --all							# 列出所有的镜像
			--digests					show digests
	-f, --filter filter 	Filter output based on conditions provided
			--format string 	Pretty-pr images using a Go temolate
			--no-trunc				Don't truncate output
	-q, --quiet						Only show numeric IDs		# 只显示镜像的ID
```

```shell
docker pull								# 下载镜像
docker pull 镜像名[:tag]		# 不写tag，默认下载就是latest
eg.	docker pull mysql:5.7	# 下载5.7版本的mysql
```

```shell
docker rmi														# 删除镜像
docker rmi [IMAGE ID]									# 删除指定镜像，后面可以跟多个镜像ID，中间加一个空格
docker rmi -f $(docker images -aq)		# 删除所有的镜像
```

### 容器命令

>有了镜像之后才会有容器

```shell
docker pull centos
```

新建容器并启动

```shell
docker run 命令
docker run [可选参数] image

# 参数说明
--name="Name"			容器名字		tomcat01，tomcat02,用来区分容器
-d								后台方式运行
-it								使用交互方式运行，进入容器查看内容
-p								指定容器的端口 -p  8080:8080
		-p	ip	主机端口:容器端口
		-p	主机端口:容器端口
		-p  容器端口
-P								随机指定端口

# 测试，启动并进入容器
docker run -it centos /bin/bash
# 容器很多命令都是不完善的
```

列出所有的运行的容器

```shell
# docker ps 命令			列出当前正在运行的容器
-a # 列出当前正在运行的容器+带出历史运行过的容器
-n=? # 显示最近创建的容器
-q # 只显示容器的编号
```

退出容器

```shell
exit							# 直接容器停止并退出
Ctrl + P + Q			# 退出容器不停止
```

删除容器

```shell
docker rm 容器id									   # 删除指定的容器，但是不能删除正在运行的容器
docker rm -f $(docker ps -aq)				# 删除所有的容器
docker ps -a -q | xargs docker rm 	# 删除所有的容器
```

启动和停止容器的操作

```shell
docker start 容器id				# 开启容器
docker restart 容器id			# 重启容器
docker stop 容器id				# 停止当前正在运行的容器
docker kill 容器id				# 强制停止当前容器
```

### 常用其他命令

后台启动容器

```shell
# 命令 docker run -d 镜像名称
docker run -d centos

# 问题 docker ps，发现 centos 停止了

# 常见的坑，docker容器使用后台运行，就必须要有一个前台进程，docker发现没有应用，就会自动停止。
# nginx，容器启动后，发现自己没有提供服务，就会立刻停止，就是没有程序了。
```

查看日志

```shell
docker logs -tf --tail 10 容器ID

# 编写一段shell脚本，产生日志
docker run -d centos /bin/sh -c "while true;do echo xylink;sleep 1;done"



# 查看日志
-tf								# 显示日志，显示时间戳
--tail						# 显示日志条数
docker logs -tf --tail 10 容器ID
```

查看容器中的进程信息

```shell
# 命令 top
docker top 容器ID
```

查看容器内部元数据

```shell
# 命令
docker inspect 容器ID
```

进入当前正在运行的容器

```shell
# 我们通常容器都是使用后台方式运行的，需要进入容器，修改一些配置

# 命令
docker exec -it 容器ID bashshell

# 方式二
docker attach 容器ID


docker exec 			进入容器后进入一个新的终端，可以在里面操作
docker attach			进入容器正在执行的终端，不会启动新的进程
```

从容器内拷贝文件到主机上

```shell
docker cp 容器ID:容器内的路径 目的主机路径
```

### 小结

![image-20210707174846693](/Users/tile/Library/Application Support/typora-user-images/image-20210707174846693.png)



## Docker

```shell
# STATUS: 容器状态。
状态有7种：
created（已创建）
restarting（重启中）
running（运行中）
removing（迁移中）
paused（暂停）
exited（停止）
dead（死亡）


ip1-->ip2  ip11->ip2    ip11->ip2 ip11->ip2 


```
