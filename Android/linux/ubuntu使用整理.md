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
