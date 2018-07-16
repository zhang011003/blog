记录一下docker提供的一些常用命令

类似于git，docker所有命令都是以docker开头加空格加命令的形式。
所有命令详细解释都可以通过如下命令来获取帮助信息
```
docker [command] --help
```
[command]为你想查看的命令。

下面的命令以运行redis为例。

1. docker pull
	获取镜像。这个比较简单，比如获取redis镜像
```
docker pull redis
```
2. docker run
	用来启动一个容器，这个命令参数较多，只记录一下常用的几个参数，后续如果发现有遗漏再补。
启动刚刚获取的redis容器
```
docker run --name myredis -d redis
```

* --name
标记该容器名称，如果没有，则为一串数字，
* -d
让容器在后台运行，detach单词缩写。

现在redis已经启动了，接下来需要使用redis-cli连接该redis了，还是使用run命令(启动一个新的容器去连接之前的容器)
```
docker run --rm -it --link myredis:redis redis /bin/bash
```
其中

* --link
将两个容器连接了起来，比如上条命令即将myredis容器与新的redis容器连接
* --rm
当该容器停止后自动删除。
* -i
交互式访问，interactive,也就是说保持stdin打开状态，输入的内容能看到。
* -t
分配一个伪终端，也就是说响应显示在界面上。
* /bin/bash
在该容器上执行的命令，也可以简单地写成bash。

其它常用命令如下：

* -v：将主机目录挂载到容器
```
docker run -v /test:/test --rm -it redis bash
```
该命令将主机目录下/test目录挂载到容器的/test目录

* --volumes-from:将已经运行中的容器的目录挂载到新容器
```
docker run --volumes-from myredis --rm -it redis bash
```
该命令将myredis目录挂载到新的redis容器中

3.docker ps
用来查看启动的容器，这个命令比较简单
```
docker ps
```
4.docker log
用来查看容器日志，比较简单

5.docker start/stop
用来启动/停止运行中的容器，比较简单。

6.docker rm
用来删除停止的容器。可以使用如下命令删除所有的已经停止的容器
```
docker rm $(docker ps -aq)
```
7.docker exec
用于进入一个正在运行的容器。比如之前启动了redis服务，现在需要使用redis客户端连接服务端，可以执行如下命令
```
docker exec -it redis bash
```
全部命令详细解释可参考官网 [docker命令](https://docs.docker.com/engine/reference/commandline/docker/)
