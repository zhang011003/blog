## docker machine ##

* 通过docker machine命令，我们可以快速安装docker环境

* windows的docker toolbox自带docker machine

* 命令

  * 创建本地虚拟机 

    windows上安装好docker toolbox后，已经默认创建了一个叫做default的docker环境，其实就是一个virtualbox的虚拟机。我们现在再创建一个default2的docker环境

    ```
    docker-machine.exe create -d virtualbox default2
    ```

  * 查看主机

    ```
    docker-machine.exe ls
    
    NAME       ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER   ERRORS
    default    *        virtualbox   Running   tcp://192.168.99.100:2376           v18.09.0
    default2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.09.0
    ```

  * 登录到default主机进行操作

    ```
    docker-machine.exe ssh default
    ```

  * 其它命令可以通过直接输入`docker-machine.exe`来查看


