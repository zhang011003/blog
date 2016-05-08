#fish设置环境变量
这几天在网上看到fish，就试着在家里mac上安装尝鲜。虽然很好用，但是之前bash的环境变量却都没有了，比如运行`mvn package`，提示没有mvn命令，只好重新设置fish的环境变量。

编辑~/.config/fish/config.fish文件，添加如下内容保存并重新打开终端
set -x PATH /your/maven/home/bin $PATH
再运行`mvn package`，运行成功
