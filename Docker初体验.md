

# Docker初体验

1、Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

```shell
由于未开启docker服务
```

2、docker的使用

```shell
docker pull 																# 拉取镜像

docker run  <==>  docker create + docker start								# 容器的运行，先创建后启动
```

3、docker状态信息

```shell
[root@txdev-ops-test1 ~]# docker ps -a
CONTAINER ID   IMAGE         COMMAND       CREATED          STATUS                      PORTS     NAMES
7f92e2c5c0f4   centos        "/bin/bash"   13 minutes ago   Exited (0) 13 minutes ago             upbeat_chatelet
1dcc41ad5f00   centos        "/bin/bash"   18 minutes ago   Up 18 minutes                         busy_neumann
e9ce6e8167c0   hello-world   "/hello"      28 minutes ago   Exited (0) 28 minutes ago             sweet_hertz
```

```shell
[root@txdev-ops-test1 ~]# docker images -a
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    d1165f221234   4 months ago   13.3kB
centos        latest    300e315adb2f   7 months ago   209MB
```

```shell
[root@txdev-ops-test1 ~]# docker stop 4f				# 停止容器4f
4f
[root@txdev-ops-test1 ~]# docker ps -a					# 查看状态
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS                        PORTS     NAMES
4ff05698af5f   centos        "/bin/sh -c 'while t…"   4 minutes ago   Exited (137) 14 seconds ago             infallible_visvesvaraya
eca0b6339c55   mysql:5.7     "docker-entrypoint.s…"   5 minutes ago   Exited (1) 5 minutes ago                charming_hertz
ed8cbdeb6f5a   hello-world   "/hello"                 8 minutes ago   Exited (0) 8 minutes ago                elegant_lewin
1e8a614f67ad   centos        "/bin/bash"              8 minutes ago   Up 3 minutes                            laughing_cerf
[root@txdev-ops-test1 ~]# docker start 4f
4f
[root@txdev-ops-test1 ~]# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS                     PORTS     NAMES
4ff05698af5f   centos        "/bin/sh -c 'while t…"   5 minutes ago   Up 1 second                          infallible_visvesvaraya
eca0b6339c55   mysql:5.7     "docker-entrypoint.s…"   6 minutes ago   Exited (1) 6 minutes ago             charming_hertz
ed8cbdeb6f5a   hello-world   "/hello"                 9 minutes ago   Exited (0) 9 minutes ago             elegant_lewin
1e8a614f67ad   centos        "/bin/bash"              9 minutes ago   Up 4 minutes                         laughing_cerf
```

> 当容器使用交互模式启动后，退出之后，再一次启动的时候，发现状态一直未改变！
>
> 
>
> 这是由于运行时发现前台没有应用，就会自动停止，所以状态一直为关闭状态，通过看它的状态时间，会发现启动的痕迹。

```shell
# 查看日志
[root@txdev-ops-test1 ~]# docker run -d centos /bin/sh -c "while true;do echo xylink;sleep 1;done"
4ff05698af5f923ce8c495f2f1adf7567b66b00e35c7c5240ce80ffd65d9bb03

[root@txdev-ops-test1 ~]# docker logs --tail 10 4f
xylink
xylink
xylink
xylink
xylink
xylink
xylink
xylink
xylink
xylink
```

```shell
# 容器命名
docker run -d -P --name [容器名称] [镜像名称]

容器多少种状态
```

## 环境搭建

> Docker 安装 Nginx 

1、搜索

```shell
docker search [镜像]
https://hub.docker.com/				# 用于检索镜像版本
```

2、拉取镜像

```shell
docker pull nginx							# 拉取最新的nginx
```

3、启动测试

```shell
docker run -d --name nginx01 -p 3344:80 nginx				# 创建一个命名为nginx01的容器，端口号为3344:80

# 测试
[root@txdev-ops-test1 ~]# curl localhost:3344
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# 进入并查看
[root@txdev-ops-test1 ~]# docker exec -it  nginx01 /bin/bash
root@d4c5fcc1108f:/# whereis nginx
nginx: /usr/sbin/nginx /usr/lib/nginx /etc/nginx /usr/share/nginx
root@d4c5fcc1108f:/# ls
bin  boot  dev	docker-entrypoint.d  docker-entrypoint.sh  etc	home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@d4c5fcc1108f:/# cd /etc/nginx
root@d4c5fcc1108f:/etc/nginx# ls
conf.d	fastcgi_params	mime.types  modules  nginx.conf  scgi_params  uwsgi_params
root@d4c5fcc1108f:/etc/nginx#
```

> Docker  安装 Tomcat 9.0

```shell
# 官方使用
docker run -it -rm tomcat									# 用完即删


```

## 思考问题

每次改动nginx配置文件，都需要进入容器内部？十分的麻烦，我要是在容器外部提供一个映射路径，达到在容器修改文件名，容器内部就可以自动修改？-v 数据卷！

+++

我们以后部署项目，如果每次都要进入容器是不是十分麻烦？要是在容器外部提供一个映射路径，webapps，我们在外部放置项目，就自动同步到哪步就好了。
