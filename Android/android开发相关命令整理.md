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

#### 查看应用版本
```
adb shell pm dump 包名 | grep version
```

### 查看当前显示在前台的activity
```
linux:

adb shell dumpsys activity | grep "mFocusedActivity"

windows:

adb shell dumpsys activity | findstr "mFocusedActivity"
```



###  signapk.jar给apk签名
```
将需要签名的apk放在签名文件同一目录下

java -jar signapk.jar platform.x509.pem platform.pk8 app-release.apk app_release_signed.apk
```


