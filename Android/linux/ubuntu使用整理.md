### 关机重启

#### 桌面无响应后安全重启
- 不要直接按电源建断电

-  reisub
在系统罢工后，如果开启了 SysRq 而且系统仍能响应键盘按键，可以使用 SysRq 键进行重启。按

Ctrl + Alt + PrtSc (SysRq) + reisub

(单词 busier (英语"更忙"的意思)的倒写）

按住Ctrl，Alt和PtrSc（SysRq），按住他们的同时你需要按r，e，i，s，u，b


    R: Switch the keyboard from raw mode to XLATE mode. 将键盘控制从 X Server 那里抢回来（unRaw） E: Send the SIGTERM signal to all processes except init. 给所有进程发送 SIGTERM 信号，让他们自己解决善后（tErminate）
    I: Send the SIGKILL signal to all processes except init. 给所有进程发送 SIGKILL 信号，强制他们马上关闭（kIll）
    S: Sync all mounted file systems (IMPORTANT). 将所有数据同步至磁盘（Sync）
    U: Remount all mounted file systems in read-only mode. 将所有分区挂载为只读模式（Unmount）
    B: Immediately reboot the system, without un-mounting partitions or syncing. 重启（reBoot）

    需要注意的是这些按键之间有顺序，而且按键之间有时间间隔（因为要等待前一个操作的完成），推荐的时间间隔是 R – 1 秒 – E – 30 秒
    – I – 10 秒 – S – 5 秒 – U – 5 秒 – B.


### 文件系统修复
```
1.确认要运行命令的分区是否挂载,可以用umount命令进行卸载：
sudo umount /dev/sda1

2.确认这个分区的文件系统是什么，这个可以用命令：
sudo fdisk -l

3.确认没有挂载和文件系统是什么后，输入下面的命令：
fsck -t  ext4 /dev/sda1

t参数是指明文件系统是什么。/dev/sda1则是指定分区。

这个命令还有另外一种输入法，这就是：

fsck.ext4 /dev/sda1

fsck默认只对有错误的档案进行检测，但是，我们可以加一个参数-f，让fsck对于没有错的档案也强行检测。

fsck.ext4 -f /dev/sda1

fsck还有检测硬盘坏道的功能，参数是-c

fsck.ext4 -fc /dev/sda1



```


boot-repair工具:
```
add-apt-repository ppa:yannubuntu/boot-repair
apt-get update
apt-get install -y boot-repair
sudo boot-repair
```


### 设置IP地址及网关
查看当前网卡：ifconfig -a

设置IP与子网掩码：sudo ifconfig enp0s31f6 11.10.10.3 netmask 255.255.255.0

设置网关：sudo route add default gw 11.10.10.10


### 查看本机IP地址
```
    ip addr show   (  同：ip a)

    如果你希望获得最少的细节：hostname -I


    老用户可能会想要使用 ifconfig（net-tools 软件包的一部分），但该程序已被弃用。一些较新的 Linux 发行版不再包含此软件包，如果你尝试运行它，你将看到 ifconfig 命令未找到的错误。
```



### 安装企业微信
https://github.com/wszqkzqk/deepin-wine-ubuntu

软件包列表：https://deepin-wine.i-m.dev/

添加库：
wget -qO- https://deepin-wine.i-m.dev/setup.sh | sudo sh

sudo apt install ... 安装对应软件包 

例如：
安装企业微信： sudo apt install com.qq.weixin.work.deepin
应用图标需要注销重登录后才会出现


### 完美解决ubuntu20中tracker占用过多cpu，硬盘的bug
ubuntu20中磁盘不够用时可按如下操作。500G的硬盘，tracker数据库占了100多G。
https://zhuanlan.zhihu.com/p/360892389



### ubuntu解压ZIP乱码
##### 一、通过unzip行命令解压，指定字符集
由于zip格式中并没有指定编码格式，Windows下生成的zip文件中的编码是GBK/GB2312等，因此，导致这些zip文件在Linux下解压时出现乱码问题，因为Linux下的默认编码是UTF8。目前网上流行的是unzip -O cp936的方法，但一些linux发行版unzip是没有-O这个选项的。Ubuntu 12.04后续版本是有的。
命令格式：
------------------------------------------------------------
pipci@Ubuntu:~$ unzip -O CP936 xxx.zip

下面这两个参数也行
unzip -O GBK
unzip -O GB18030

##### 二、通过unar命令最简单

1、安装unar软件
-----------------------------------------------
root@Ubuntu:~# apt install unar
-----------------------------------------------

2、命令格式：
-------------------------------------------------------------------------------------------------------
pipci@Ubuntu:~$ unar xxx.zip                        #不需要加参数，自动识别编码
-------------------------------------------------------------------------------------------------------


3.unar常用选项解释

参数[-o]
解释：指定解压结果保存的位置
~$ unar test.zip -o /home/dir/

参数[-e]
解释：指定编码
~$ unar -e GBK test.zip

参数[-p]
解释：指定解压密码
~$ unar -p 123456 test.zip

4、列出压缩包内容

~$ lsar  xxx.zip



### 创建或修改应用图标

系统 .desktop 文件存放在文件夹/usr/share/applications/中。

创建或者修改：sudo gedit xxxx.desktop

改变权限：
sudo chown user xxxx.desktop
sudo chmod 755 xxxx.desktop

复制到桌面：
cp xxxx.desktop /home/user/Desktop/

双击刚刚复制的xxxx.desktop文件，点击Trust and Launch

