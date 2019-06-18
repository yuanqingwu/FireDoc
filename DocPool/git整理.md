
![image](https://upload-images.jianshu.io/upload_images/5494434-33c67581c650c8ed?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

## git stash删除恢复
1. git fsck --lost-found
使用git show 查看dangling commit的id，看看是否是自己想要的
2. git show 8dd73fa8d14880182f11e24dc10bca570b6127d7
git merge进行恢复
3. git merge 8dd73fa8d14880182f11e24dc10bca570b6127d7
4. 

## git ssh
###### 设置Git的user name和email

```
    $ git config --global user.name "yuanqingwu"
    $ git config --global user.email "wuyuanqing527@gmail.com"
```
###### 生成密钥

```
$ ssh-keygen -t rsa -C "wuyuanqing527@gmail.com"
```


###### 添加密钥到ssh-agent
确保 ssh-agent 是可用的。ssh-agent是一种控制用来保存公钥身份验证所使用的私钥的程序，其实ssh-agent就是一个密钥管理器，运行ssh-agent以后，使用ssh-add将私钥交给ssh-agent保管，其他程序需要身份验证的时候可以将验证申请交给ssh-agent来完成整个认证过程。

```
    # start the ssh-agent in the background
    eval "$(ssh-agent -s)"
    Agent pid 59566
```
###### 添加生成的 SSH key 到 ssh-agent。
```
   $ ssh-add ~/.ssh/id_rsa
```
###### 测试：

```
    $ ssh -T git@github.com
```

## git flow
https://segmentfault.com/a/1190000018359245

