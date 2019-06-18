## Android系统启动
当我们按下电源启动时，系统启动后会加载引导程序BootLoader,引导程序又启动Linux内核，在Linux内核启动完成后，首先在系统文件中寻找init.rc文件，并启动init进程。

###### init.rc文件
init.rc是由Android初始化语言(Android Init Language)编写的脚本，这种语言主要包含五种类型的语句：Action,Command,Service,Option和Import。

init.rc中的Action类型语句和Service类型语句都有相应的类来进行解析，Action类型语句采用ActionParser来进行解析，Service类型语句采用ServiceParser来进行解析。

#### init进程
主要做了以下三件事：
- 创建和挂载启动所需的文件目录
- 初始化和启动属性服务
- 解析init.rc配置文件并启动Zygote进程

##### 创建和挂载启动所需的文件目录
init的main函数在开始的时候创建和挂载启动所需的文件目录，其中挂载了tmpfs,devpts,proc,sysfs和selinuxfs共五种文件系统，这些都是系统运行时目录，顾名思义，只有在系统运行时才会存在，系统停止时就会消失。

##### 属性服务
源码参考：service/core/init/property_service.cpp

属性服务接收到客户端请求时，会调用handle_property_set_fd函数进行处理。

系统属性分为两种：普通属性和控制属性

控制属性用来执行一些命令。比如开机的动画就用了这种属性。属性名称以 "ctl."开头，就说明是控制属性。

##### 启动Zygote
Zygote执行程序的路径为/system/bin/app_process64,对应的文件名为app_main.cpp,Zygote的main函数通过调用runtime的start函数启动Zygote。

#### Zygote进程启动过程
在Android系统中，DVM或ART,应用程序进程以及运行系统的关键服务的SystemServer进程都是有Zygote进程创建的，我们也将它称为孵化器。他通过fork(复制进程)的形式来创建应用程序进程和SystemServer进程，由于Zygote进程在启动时会创建DVM或者ART，因此通过fork创建的应用程序进程和SystemServer进程可以在内部获取一个DVM或者ART的实例副本。

Zygote进程是在init进程启动时创建的，起初Zygote进程的名称并不是叫 "zygote"，而是叫 "app_process",这个名称是在Android.mk中定义的，Zygote进程启动后，Linux系统下的pctrl系统会调用app_process，并将其名称换成了 "zygote"。

##### Zygote启动脚本
在init.rc文件中采用了Import语句来引入Zygote启动脚本，这些脚本都是由Android初始化语言来编写的：

```
import /init.${ro.zygote}.rc
```
可以看出init.rc是根据ro.zygote的内容来引入不同的文件。

从Android5.0开始，Android开始支持64位程序，Zygote也就有了32位和64位的区别，ro.zygote属性的取值有以下四种：
- init.zygote32.rc
- init.zygote32_64.rc
- init.zygote64.rc
- init.zygote64_32.rc
这些Zygote启动脚本都放在system/core/rootdir目录中。

###### init.zygote32_64.rc
脚本中有两个Service类型语句，说明会启动两个Zygote进程，一个名为zygote,执行程序为app_process32,作为主模式；另一个名称为zygote_secondary，执行程序为app_progress64,作为辅模式。

###### Zygote进程启动总结
Zygote进程启动共做了如下几件事：
- 创建AppRuntime并调用其start方法，启动Zygote进程。
- 创建Java虚拟机并为Java虚拟机注册JNI方法
- 通过JNI调用ZygoteInit的main函数进入Zygote的Java层
- 通过registerZygoteSocket方法创建服务端Socket，并通过runSelectLoop方法等待AMS的请求来创建新的应用程序进程。
- 启动SystemServer进程。


##### SystemServer处理过程
SystemServer进程主要用于创建系统服务，我们熟知的AMS,WMS和PMS都是由他来创建的。

官方把系统服务分为了三种类型，分别是引导服务(BootstrapServices)，核心服务(CoreService)和其他服务(OtherServices)，其中其他服务是一些非紧要和不需要立即启动的服务。这些系统服务总共有100多个。

SystemServer进程被创建以后，主要做了如下工作：
- 启动Binder线程池，这样就可以与其他进程进行通信
- 创建SystemServiceManager,其作用对系统的服务进行创建,启动和生命周期管理
- 启动各种系统服务
 

#### Launcher启动过程


### Android系统启动流程总结

1. 启动电源以及系统启动

当电源按下时引导芯片代码从预定义的地方（固化在ROM）开始执行。加载引导程序BootLoader到RAM，然后执行。
2. 引导程序BootLoader

引导程序BootLoader是在Android操作系统开始运行前的一个小程序，他的主要作用是把OS拉起来并运行。
3. Linux内核启动

当内核启动时，设置缓存，被保护存储器，计划列表，加载驱动。当内核完成系统设置时，它首先在系统文件中寻找init.rc文件，并启动init进程。
4. init进程启动

初始化和启动属性服务，并且启动Zygote进程。
5. Zygote进程启动

创建Java虚拟机并为Java虚拟机注册JNI方法，创建服务端Socket，启动SystemServer进程。
6. SystemServer进程启动

启动Binder线程池和SystemServiceManager，并且启动各种系统服务。
7. Launcher启动

被SystemServer进程启动的AMS会启动Launcher,Launcher启动后会将已安装应用的快捷图标显示到界面上。

