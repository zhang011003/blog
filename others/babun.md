# babun使用
windows下模拟linux环境当然是用[babun](http://babun.github.io/)了，这里列出了我自己使用的一些习惯配置。

- babun使用代理上网
  公司上网需要设置代理，可以通过修改~/.babunrc文件，然后执行`source ~/.babunrc`使之生效
- totalcmd集成babun
  在windows下一直用totalcmd，但安装完后babun默认是集成进资源管理器右键，可以通过开始->更改开始菜单来增加一个babun的菜单，其中的命令、参数可以在注册表里找到，如图
  ![babun](https://camo.githubusercontent.com/c48ebf7cf1c5c23263b09504cde8ed08c595a143/687474703a2f2f692e696d6775722e636f6d2f623054614150382e706e67)
  复制下内容，命令就是mintty.exe前的字符串，参数就是剩下的字符串，菜单标题可以设置为b&abun，注意a之前增加了一个&符号，用来定义加速键
- babun下使用fish
  首先是在babun下安装fish，使用命令`pact install fish`即可，安装完成后可以使用`pact show|grep fish`查看是否安装成功
    其次是让babun默认使用fish，linux下可以使用`chsh -s `加fish路径修改fish为默认shell，如果不知道fish路径，可以执行`less /etc/shells`来查看fish路径，babun下没有办法这样操作，但刚刚在totalcmd下集成babun时我们输入的命令是这样子的`/bin/env CHERE_INVOKING=1 /bin/zsh.exe`，可以仿造它来修改默认shell，在babun下查看是否有/bin/fish.exe，果然没有猜错，修改命令为`/bin/env CHERE_INVOKING=1 /bin/fish.exe`,重启babun，哈哈，又开始fish了

2018-9-6补充：

- 由于换了新工作，又可以上网了，又下载了babun集成fish，发现fish集成没有效果，不知道是不是fish版本和babun不兼容的问题，之前也没有记录版本，只能怀疑版本的问题了。目前下载使用的babun版本是1.2.0。可以通过如下命令查看版本信息。

  ```shell
  babun --version
  ```

  fish的版本是2.7.0-1。可以通过下载的目录查看。下载目录地址可以通过如下命令的输出信息查看。

  ```shell
  pact find fish 	
  ```

- babun使用之前，最好先阅读以下[babun的github文档](https://github.com/babun/babun)，这样上手起来更快。