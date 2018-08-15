---
title: docker学习笔记
date: 2017-11-19 21:43:49
tags: Docker
categories: Linux 

---



## Docker基础概念

> Docker 是一个开源的应用容器引擎，基于 [Go 语言](http://www.runoob.com/go/go-tutorial.html) 并遵从Apache2.0协议开源。
> Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。
> 容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。
>
> <!--more-->
>
> Docker 镜像是用于创建 Docker 容器的模板。   
> Docke 容器是独立运行的一个或一组应用。  
> Docker 仓库用来保存镜像。 

## Docker常用命令

### 镜像操作 

下载镜像

`docker pull ubuntu`

搜索镜像nginx

`docker search nginx `

列出所有镜像

```
➜  ~ docker image ls                  
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              113a43faa138        3 weeks ago         81.2MB
hello-world         latest              e38bc07ac18e        2 months ago        1.85kB
```

包含了仓库名，标签，镜像 ID，创建时间和占用大小。

删除镜像

`docker image rm hello-world `

### 容器操作 

启动容器

`docker run  hello-world `

docker run会直接启动镜像，产生一个容器。

参数 -i -t会产生交互式命令

```
➜  ~ docker run -t -i ubuntu /bin/bash
root@211a08dade54:/#  cat /etc/issue
Ubuntu 18.04 LTS \n \l
```

带参数为 -d后台启动  --name 可以自定义容器名称

```
➜  ~ docker run -d --name=hello hello-world 
3c71798fb3fb471d6ccd11b0cddd4c38d88ee9b59e40769f87d598e6c3563257
```

查看日志

```
➜  ~ docker logs hello                
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

可以查看刚才启动的hello容器的输出日志。 

查看正在运行中的容器

```
➜  ~ docker ps          
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
fdc7936d58b4        nginx               "nginx -g 'daemon of…"   2 seconds ago       Up 1 second         80/tcp              amazing_jepsen
```

查看所有的容器

```
➜  ~ docker container ls -a                  
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
a8178337139d        hello-world         "-d --name hello"   38 seconds ago      Created                                         keen_shtern
ea14c9b032ce        ubuntu              "/bin/bash"         53 seconds ago      Exited (0) 52 seconds ago                       keen_chandrasekhar
```

会显示容器ID，镜像，启动命令，创建时间，状态，端口，名称等信息。



停止/启动/重启 容器

`docker container stop/start/restart  ea14c9b032ce`

进入容器交互

`docker exec -it ea14c9b032ce bash`

查看容器元数据

`docker inspect hello`

会列出容器的各种基础信息

删除容器

`docker container rm ea14c9b032ce`

删除所有终止容器 

`docker container prune ` 

### 数据管理 

1.使用 -v 参数

-v PWD1:PWD2 将宿主机目录PWD1挂载到容器内目录PWD2

```
➜  ~ docker run -i -t -v $PWD/test.txt:/home/test.txt  ubuntu
root@f2a683801ffd:/# ls /home
test.txt
```

2.使用volume 数据卷 

创建数据卷

`docker volume create my-vol `

列出所有的数据卷 

`docker volume ls `

查看数据卷信息

```
➜  Downloads docker volume inspect my-vol
[
    {
        "CreatedAt": "2018-07-03T22:43:25+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```



删除数据卷

`docker volume rm my-vol `

删除无主的数据卷 

`docker volume prune  `

挂载数据卷 

```
➜  ~ docker run -t -i --mount source=my-vol,target=/home/ ubuntu /bin/bash
root@e3b9b5a3936d:/# touch /home/test.txt

➜  ~ ls /var/lib/docker/volumes/my-vol/_data 
test.txt
```

使用`--mount source=数据卷, target=容器目录` 进行挂载 ，数据卷是持久化的，不会自动删除。默认是可读写的。

### 网络访问 

 随机端口映射

```
➜  ~ docker run -d -P nginx
058f1e6909b52b7cb75f9955ee0039407203dd62d2ac07f82219cceaf820e6a5
➜  ~ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
058f1e6909b5        nginx               "nginx -g 'daemon of…"   18 seconds ago      Up 17 seconds       0.0.0.0:1024->80/tcp   modest_aryabhata
```

使用大写`-P`参数会将宿主机的端口随机映射到容器。如上例，本地主机的1024端口被映射到了容器的80端口，此时访问`http://127.0.0.1:1024/`就可以查看到nginx主页。

指定端口映射

```
➜  ~ docker run -d -p 80:80 nginx                
f92cc24e5bfe11cbadfc6c15e5477f4ad78e272e9db3a59996f59776f8b58e11
➜  ~ curl 127.0.0.1:80 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

使用小写`-p`参数可以指定端口映射。

容器互联

新建网络

`docker network create -d bridge my-net`

连接容器

```
➜  ~ docker run -it --name test1 --network my-net busybox sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:02  
          inet addr:172.18.0.2  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:19 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2813 (2.7 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
---

➜  ~ docker run -it --name test2 --network my-net busybox sh
/ # ping 172.18.0.2
PING 172.18.0.2 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.212 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.184 ms
```

此时，test1容器和test2可以相互访问。

## 创建新的镜像 

1.使用 docker commit

以使用ubuntu镜像制作python3为例

```
➜  ~ docker run -it ubuntu /bin/bash
root@9cac2dab97dd:/# apt update
Get:1 http://security.ubuntu.com/ubuntu bionic-security InRelease [83.2 kB]    
...
root@9cac2dab97dd:/# apt install python3
Reading package lists... Done
...
➜  ~ docker commit --author='tester' --message='安装python3' 9cac2dab97dd my-python3
sha256:9211535c8668b7e4cc195504326a3903b0a7c9fdbbb599a5acabbc06aae1c92e
➜  ~ docker run -ti my-python3 /usr/bin/python3
Python 3.6.5 (default, Apr  1 2018, 05:46:30) 
[GCC 7.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

docker commit 方法是用来在现有镜像的基础上生成新镜像的，不过不推荐这种方法。

2.使用 docker file 

还是以制作python3为例，创建Dockerfile文件
```
FROM ubuntu
RUN apt update&&apt install -y python3
CMD ["python3"]

```
使用`docker build`构建镜像
`docker build -t my-python3 .`
运行
```
➜  docker run -ti my-python3
Python 3.6.5 (default, Apr  1 2018, 05:46:30) 
[GCC 7.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

### 导入导出镜像

保存镜像

```
docker save hello-world |gzip hello.tar.gz
```

导入镜像

```
docker load -i hello.tar.gz
```

这种方法适用于直在没有网络时进行镜像传输
