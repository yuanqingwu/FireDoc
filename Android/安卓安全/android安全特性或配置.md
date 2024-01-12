
## Android之allowBackup属性
AllowBackup是在Android 2.2中引入的一个系统备份的功能。允许用户备份系统应用和第三方应用的apk安装包和应用数据，以便在刷机或者数据丢失后恢复应用，用户即可通过adb backup和adb restore来进行对应用数据的备份和恢复。第三方应用开发者需要在应用的 AndroidManifest.xml 文件中配置 allowBackup 标志(默认为 true )来设置应用数据是否能能够被备份或恢复。

allowBackup会引起的高危漏洞是什么？

Android属性allowBackup安全风险源于adb backup容许任何一个能够打开USB 调试开关的人从Android手机中复制应用数据到外设，一旦应用数据被备份之后，所有应用数据都可被用户读取；adb restore容许用户指定一个恢复的数据来源（即备份的应用数据）来恢复应用程序数据的创建。因此，当一个应用数据被备份之后，用户即可在其他Android手机或模拟器上安装同一个应用，以及通过恢复该备份的应用数据到该设备上，在该设备上打开该应用即可恢复到被备份的应用程序的状态。

尤其是通讯录应用，一旦应用程序支持备份和恢复功能，攻击者即可通过adb backup和adb restore进行恢复新安装的同一个应用来查看聊天记录等信息；对于支付金融类应用，攻击者可通过此来进行恶意支付、盗取存款等；因此为了安全起见，开发者务必将allowBackup标志值设置为false来关闭应用程序的备份和恢复功能，以免造成信息泄露和财产损失。

该漏洞的解决方案：

1.将allowBackup 的值设置为false；（allowBackup的值为false 对项目运行没有任何影响）

2.通过手机设备的IMEI号来辨识来设备编号和备份前是否一致，不一致则直接跳转登陆页面的同时清空当前应用数据及缓存；\


###### 破解例子

Android逆向之旅—Android中如何获取在非Root设备中获取应用隐私数据： http://www.520monkey.com/archives/615