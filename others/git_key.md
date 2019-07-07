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

此时需要在～/.ssh 目录下新建一个config文件，在里面填入如下rsa信息:

```
Host xxx                  //这里填git地址别名，写一个便于记忆的名字即可
        HostName xxx.com  //这里填git地址
        User git          //这里必须是git，测试发现如果是别的用户名会报错
        PreferredAuthentications publickey //如果注释，则公钥认证不通过则会使用密码认证
        IdentityFile ~/.ssh/test_rsa //生成的rsa私钥的文件路径
        AddKeysToAgent yes //将key增加到agent中，如果没有，则需要手工通过命令增加
        UseKeychain yes    //mac中用到，应该是加到keychain中的意思。测试发现加不加没有什么区别 
```
保存退出即可。

**这里和gitlab上给的说明文档有区别，使用gitlab提供的示例，会提示无法识别RSAAuthentication**

配置完成后进行测试，执行命令`ssh -Tv xxx`可以测试是否配置错误，如果还需要输入密码，说明配置是有错误的，需要重新检查哪里配错了。其中命令后面的参数是配置的host的值。

如果配置成功后没有进行测试，那么需要使用命令`ssh-add ~/.ssh/test_rsa`手工增加key到agent中，否则执行`git pull`时仍然无法获取代码。

如果按照上面那样配置的话，每回重启电脑还需要执行一下ssh-add命令，否则还是无法获取代码，网上查了很多资料都是说把这个命令写在启动脚本中，但我感觉这种方式不是很优雅，在我还没有找到更好的方法之前，先这样用吧。后续找到更好的方法再在这里补上。

另外，研究了一个上午的ssh的公钥私钥配置，发现自己对ssh一点都不熟悉，看来需要恶补一下ssh的基础知识了。

------

2019-7-7补充

今天无意中看到github上有详细描述git密钥的配置的文章，看完后发现有些技术性的文章真的不能在网上乱找，真是害死人啊。

1. 配置多个ssh key每次重启电脑后需执行ssh-add命令嫌麻烦？

   打开git bash，在~/.profile中增加如下代码

   ```bash
   env=~/.ssh/agent.env
   
   agent_load_env () { test -f "$env" && . "$env" >| /dev/null ; }
   
   agent_start () {
       (umask 077; ssh-agent >| "$env")
       . "$env" >| /dev/null ; }
   
   agent_load_env
   
   # agent_run_state: 0=agent running w/ key; 1=agent w/o key; 2= agent not running
   agent_run_state=$(ssh-add -l >| /dev/null 2>&1; echo $?)
   
   if [ ! "$SSH_AUTH_SOCK" ] || [ $agent_run_state = 2 ]; then
       agent_start
       ssh-add
   elif [ "$SSH_AUTH_SOCK" ] && [ $agent_run_state = 1 ]; then
       ssh-add
   fi
   
   unset env
   ```

   其中ssh-add命令默认是增加默认的id_rsa文件，需要修改为自己生成的rsa文件名称，如

   ```bash
   ssh-add ~/.ssh/id_github_rsa
   ```

   [^]:参考 [Working with SSH key passphrases](https://help.github.com/en/articles/working-with-ssh-key-passphrases)

2. 执行ssh -Tv 时输出的内容看不懂？

   [^]:参考[Error: Permission denied (publickey)](https://help.github.com/en/articles/error-permission-denied-publickey)

3. 如果别人使用你的电脑，则可以使用你的ssh key访问所有系统怎么办？

   需要增加一个额外的安全层，此时需要用到passphase

   [^]:参考[Working with SSH key passphrases](https://help.github.com/en/articles/working-with-ssh-key-passphrases)




