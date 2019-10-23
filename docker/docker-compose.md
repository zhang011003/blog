# Docker Compose

## docker compose介绍



## docker compose命令

1. 启动docker compose

   ```
   docker-compose up
   ```

   此时启动是在前台，会将日志也打印出来。但是当退出的时候，对应的容器也就退出了。如果希望后台启动，可以使用如下命令

   ```
   docker-compose up -d
   ```

   此时启动是在后台，如果想查看日志，可以使用如下命令

   ```
   docker logs 容器名称
   ```

2. 查看启动状态

   ```
   docker-compose ps
   ```

   可以查看所有启动的容器的状态

3. 停止但不移除容器

   ```
   docker-compose stop
   ```

   默认是停止所有的容器。也可以将容器名称作为参数，来指定停止哪个容器

   此时如果再通过ps命令查看，还是可以看到所有的容器，说明容器并没有被移除，只是停止了。如果想要让容器再次启动，可以使用如下命令

   ```
   docker-compose start
   ```

4. 停止并移除容器

   ```
   docker-compose down
   ```

   此时容器不光是被停止，也被移除。如果再次使用ps命令查看，已经无法查看到对应的容器了

5. 

## docker compose文件格式

