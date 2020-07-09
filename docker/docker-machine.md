## docker machine

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
  
  * 查看default的变量配置
    
    ```
    docker-machine.exe env default
    ```
  
  * 其它命令可以通过直接输入`docker-machine.exe`来查看
  
  * 

* 注意
  
  * 默认的docker-machine会将虚拟机安装在C盘，如果不希望安装在C盘，则需要在create的时候增加参数`--storage-path`或者`-s`指定存储的路径
  
  * 也可以通过配置环境变量`MACHINE_STORAGE_PATH`设置默认的存储路径
  
  * 如果已经安装了虚拟机，且安装在了C盘，则可以通过VirtualBox界面将安装的虚拟机挪动到其它的盘
    
    * 在VirtualBox的全局工具中，选择需要移动的虚拟硬盘，然后点击移动即可
    
    * 选择虚拟机->设置->存储，在存储页面，可以在控制器下看到boot2docker.iso，找到boot2docke.iso文件，将其移动到其它盘，然后删除原来的boot2docker.iso，并新增新位置的boot2docker.iso文件






