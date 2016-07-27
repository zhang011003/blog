#PHP环境搭建
对于一个像我这样的php新手来说，搭建php环境也是需要记录一下的，否则后续估计又忘记了。

1. 下载php
在[php官网](http://www.php.net/)下载php，当前有三个不同版本号的稳定版本，具体没有看它们的区别。我下载的是5.6.23版本
2. 配置php.ini
下载完成后解压缩，进入解压缩目录后，发现有两个ini文件：php.ini-development和php.ini-production，复制php.ini-development并更名为php.ini，然后搜索如下内容，并去掉前面的分号注释
extension_dir = "ext"
cgi.fix_pathinfo=1
extension=php_openssl.dll
3. 启动php-cgi.exe
打开命令行，进入php解压缩目录，执行`./php-cgi.exe -h`，发现有一个-b选项是绑定外部fastcgi服务模式路径的
-b <address:port>|<port> Bind Path for external FASTCGI Server mode
执行./php-cgi.exe -b 127.0.0.1:9000
4. nginx中配置
参考这篇文章[如何正确配置 Nginx 和 PHP](http://blog.jobbole.com/50121/)
```
    server {
    	listen 80;
    	server_name  localhost;

        index index.html index.htm index.php;
        location / {
            try_files $uri $uri/ /index.php;
        }

        location ~ \.php$ {
            try_files $uri =404;
            root           d:\study\yii2\yii-advanced-app-2.0.9\advanced;
            include        fastcgi.conf;
            fastcgi_pass   127.0.0.1:9000;
        }
    }
```

上述这些步骤太麻烦了，于是有人做了一个类似于xampp的软件，叫做wnmp，直接下载[wnmp](https://www.getwnmp.org/downloads/)，然后启动nginx和php即可，好傻瓜的样子
