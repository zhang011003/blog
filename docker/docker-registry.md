寄存服务(regisry)

* docker中的寄存服务（registry）负责托管和发布镜像服务，默认为docker hub。可以理解为maven的仓库。
* docker中的仓库（repository）则是一组相关镜像（通常是一个应用或服务的不同版本）的集合。可以理解为maven中仓库里的jar包集合。
* 默认情况下，寄存数据将会以docker卷存储在主机文件系统中（By default, your registry data is persisted as a docker volume on the host filesystem），但我一直没有找到具体是存储在主机的什么位置。如果需要指定存储的位置，就需要在run的时候使用-v参数了，其实就是将本机目录挂载到容器目录

