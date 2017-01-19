#Tomcat相关信息
* tomcat-maven-plugin

maven2有一个把web应用部署到tomcat下的插件 tomcat-maven-plugin，到现在才知道，真是不应该

具体参考[tomcat-maven-plugin](http://tomcat.apache.org/maven-plugin.html)

* tomcat日志按天分隔

一直不清楚怎么做到，网上有人说用插件，有人说自己写cron脚本，总觉得不优雅。其实tomcat本身就提供将默认的java.util.logging日志记录转换为Log4j，并且官方文档有详细的操作说明。唉~~，以后一定要仔细阅读官方文档了

[tomcat日志说明](http://tomcat.apache.org/tomcat-8.0-doc/logging.html)
