#### Docker FAQ

- Docker 日志文件目录
  
  按照 [docker官网上相关的日志的描述](https://docs.docker.com/config/daemon)，在我本地根本没有对应的日志文件目录，最后终于发现，docker的日志路径为 /var/lib/boot2docker/log/docker.log

- 如何挂载windows的文件到Docker
  
  困扰我好久的问题，今天终于找到答案了。
  
  参考[Docker: Permanently Mount a VirtualBox Shared Folder](http://developmentalmadness.com/2016/03/05/docker-permanently-mount-a-virtualbox-shared-folder/)


