# 安裝雨林木风win7操作系统遇到的问题

由于mac操作系统的快捷键和win操作系统的快捷键差别很大，在家里的mac电脑上用eclipse很是不习惯，还是装一个win7操作系统虚拟机方便一点。

下载了一个雨林木风的win7操作系统iso，但在安装的过程中，一直提示nor find gwin7.gho，直接运行ghost.exe时则是卡死，但是打开iso文件是可以找到gwin7.gho文件的。这是怎么回事啊？

经过坚持不懈的百度，终于发现问题了。原来需要设置的时候硬件选项卡->硬盘1->高级设置，将位置修改为IDE 0:0，在CD/DVD 1->高级设置，将位置修改为IDE 0:1
再次运行ghost.exe，一切正常，选择gwin7.gho文件时，先选择D盘就可以找到了

设置比较简单，就不截图了。