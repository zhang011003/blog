#mac上的$JAVA_HOME环境变量设置
mac下的java是默认安装的，所以从来没有设置过$JAVA_HOME环境变量，今天在fish下运行mvn命令，比如`mvn package`，提示没有mvn命令，设置好mvn的path后再运行，提示`Error: JAVA_HOME is not defined correctly`，echo $JAVA_HOME才发现JAVA_HOME环境变量为空，怎么设置呢？google了一下，才有了解决方案，参考这篇文章[Important Java Directories on Mac OS X](https://developer.apple.com/library/mac/qa/qa1170/_index.html)，主要是用到了`/usr/libexec/java_home`这个工具命令，执行该命令，就会输出JAVA_HOME路径，还是动态的，后续如果升级java后也不需要修改JAVA_HOME值，非常方便

