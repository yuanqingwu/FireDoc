### ADB 命令整理

https://www.jianshu.com/p/980fea2c9457

### 安装应用
```
adb installl -r -t -d xxx.apk

-r : 替换已经存在的应用
-t : 允许安装测试包
-d : 允许降级覆盖安装
```
### 替换系统应用
```
adb root
adb remount
adb push xxx.apk /system/app/xxx/

查看：
adb  shell  cd /system/app/xxx/
ls -l

adb reboot

查看版本：
adb shell pm dump 包名 | grep version

```

### 结束应用进程
adb shell am force-stop XXX

```
if (app.persistent && !evenPersistent) {
// we don't kill persistent processes
continue;
}
应用中设置了android:persistent="true"，则不能被杀死
```

### adb启动分屏
adb shell am start -n 应用的包名/Activity名称  --stack 3

### 查看安装的应用包名
adb shell pm list packages | grep xxx

查看包安装位置:
adb shell pm list packages -f | grep xxx


#### 查看应用版本
```
adb shell pm dump 包名 | grep version
```
#### 查看应用签名
```
keytool -printcert -jarfile xxx.apk
```

### 打印当前系统信息（adb shell dumpsys）
1.通过"adb shell dumpsys"或"adb shell service list"获取运行的所有system service
2.在dumpsys后面加上上面查到的service名字，查看指定的service信息
3.添加-h查看帮助信息，例如通过"adb shell dumpsys window -h"查看此service支持的额外参数

### 查看当前显示在前台的activity
```
linux:

adb shell dumpsys activity | grep "mFocusedActivity"

windows:

adb shell dumpsys activity | findstr "mFocusedActivity"
```

### 查看主动启动activity的日志

```
adb logcat |grep -inE "start u0"

过滤出来的日志如下：
4260:01-01 08:04:26.261   950  3127 I ActivityManager: START u0 {flg=0x10000000 cmp=com.wyq.appstore/.main.MainActivity (has extras)} from uid 10003
```

### adb查看activity信息
dumpsys activity recents

dumpsys activity activities

dumpsys activity processes

### adb查看windows信息
dumpsys window windows

### 查看系统日志路径
adb shell getprop vendor.MB.realpath

### 从Android 手机取出已安装apk文件
查看包名 如com.zhangyou.plamreading

$ adb shell pm path com.zhangyou.plamreading

得到package:/data/app/com.zhangyou.plamreading-GYHKFVL_NdPeZUWBnVmJQA==/base.apk
然后

$ adb pull /data/app/com.zhangyou.plamreading-GYHKFVL_NdPeZUWBnVmJQA==/base.apk .

即可保存在当前目录
签名相关

### 设置logcat日志缓冲区大小
设置为20M：
adb logcat -G 20M
查看设置结果：
adb logcat -g     

重启以后需要再次设置，也许需要重启adb服务

###  signapk.jar给apk签名
```
将需要签名的apk放在签名文件同一目录下

java -jar signapk.jar platform.x509.pem platform.pk8 app-release.apk app_release_signed.apk
```

使用apksigner.jar给APK签名：
```
java -jar apksigner.jar sign --ks android_platform.keystore --ks-key-alias android_platform --ks-pass pass:android --key-pass pass:android --out app_signed.apk app.apk
```

### remount 系统
mount -o remount -rw /system

mount -o remount,rw /system

一般在终端下操作Android系统，我们访问系统分区的时候，经常遇到Only Read的问题，此时需要以读写方式重新挂载需要操作的分区

 
1、重新挂载根分区  mount -o remount  / 

 
2、以读写的模式重新挂载 根分区   mount -o remount, rw /
 
3、以不含suid的模式重新挂载根分区  mount -o remount, nosuid / 
 
4、重新挂载system分区，这个也是我个人在开发的时候用得最多的，因为我经常需要读写system/app/目录下系统自带的apk安装包。
mount -o remount, system/