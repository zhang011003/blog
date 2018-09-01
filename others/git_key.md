# git密钥的配置

之前用git时配置过密钥，现在需要记录一下，后续忘记也可以看到这篇博客回忆一下。

使用git免密码操作（下载代码，提交代码等），需要配置git的ssh key。公司一般使用gitlab作为git的代码仓库，登录以后，可以在User Settings->SSH Keys中增加ssh key。

在SSH Keys页面，有使用提示，可以点击generate one链接查看如何生成key。

1. 首先需要安装 [git](https://gitforwindows.org)，安装完成后，可以在git bash中执行如下命令：
```bash
ssh-keygen -t rsa -C "your.email@example.com" -b 4096
```
把your.email@example.com替换成自己的邮箱地址，如果没有邮箱地址，替换成自己名字也可以。

2. 一路回车完成后，在～/.ssh 目录下就会生成两个文件id_rsa和id_rsa.pub，其中id_rsa是私钥文件，id_rsa.pub是公钥文件。

3. 使用cat打开id_rsa.pub文件，将内容复制到SSH Keys页面的key对话框中，此时就会发现Title里填的内容就是执行命令时-C参数的值

4. 点击Add Key按钮。

大功告成！

还有一种情况，就是在执行ssh-keygen命令时，已经有id_rsa文件，比如公司gitlab需要id_rsa文件，此时需要生成github用到的rsa文件时该怎么办？

这时在执行ssh-keygen命令时就不能一路回车了，看到`Enter file in which to save the key(/Users/xxx/.ssh/id_rsa):`的提示后，需要输入生成的rsa文件的路径以及rsa文件的名称，如果执行ssh-keygen命令时就在～/.ssh目录下，那可以直接键入rsa文件名称，比如输入test_rsa，然后再一路回车完成后，在～/.ssh 目录下就会生成两个文件test_rsa和test_rsa.pub文件，

此时需要执行命令`ssh-add ~/.ssh/test_rsa`命令，并且需要在～/.ssh 目录下新建一个config文件，在里面填入如下rsa信息:

```
Host xxxhost
        HostName xxx.com  //这里填git地址
        User XXX
        PreferredAuthentications publickey
        IdentityFile ~/.ssh/test_rsa
```
保存退出即可。

==**（这里和gitlab上给的帮助说明有区别，使用gitlab的示例，会提示无法识别RSAAuthentication）**==

最后还可以进行测试，执行命令`ssh -Tv xxxhost`可以测试是否配置错误，如果还需要输入密码，说明配置是有错误的，需要重新检查哪里配错了。其中命令后面的参数是配置的host的值











