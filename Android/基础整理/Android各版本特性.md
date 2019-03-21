[Android各个版本新特性](https://www.jianshu.com/p/8a66806588bc)

## 4.4

https://developer.android.google.cn/about/versions/kitkat

##### 低功耗传感器

##### 添加全屏沉浸模式

##### 支持OpenGL ES 3.0

##### 添加转场动画




## 5.0
https://developer.android.google.cn/about/versions/lollipop

##### Android Runtime (ART)默认运行平台设置
##### 引入Material Design设计

##### 支持Android NDK中的64位



## 6.0
##### 运行时请求权限

##### 低电耗模式和应用待机模式

##### 取消支持 Apache HTTP 客户端

##### 支持文本选择

## 7.0 (Nougat 牛轧糖)
##### 电池和内存
 - 低电耗模式
 - Project Svelte：后台优化
##### 权限更改
 - 系统权限更改
##### 在应用间文件共享权限控制
##### 多窗口支持
##### 通知栏快捷回复
##### 支持VR
##### 引入JIT编译器
##### 画中画
##### App快捷菜单

## 8.0 (Oreo API:26 )   8.1(API:27)
https://developer.android.google.cn/about/versions/oreo/android-8.0

##### 通知
在 Android 8.0 中，我们已重新设计通知，以便为管理通知行为和设置提供更轻松和更统一的方式。这些变更包括：
通知渠道，通知标志，休眠，通知超时，通知设置，通知清除，背景颜色，消息样式。


##### 自动填充框架
Android 8.0 通过引入自动填充框架，简化了登录和信用卡表单之类表单的填写工作。在用户选择接受自动填充之后，新老应用都可使用自动填充框架。

##### 画中画模式
Android 8.0 允许以画中画 (PIP) 模式启动操作组件。PIP 是一种特殊的多窗口模式，最常用于视频播放。目前，PIP 模式可用于 Android TV，而 Android 8.0 则让该功能可进一步用于其他 Android 设备。

当某个 Activity 处于 PIP 模式时，它会处于暂停状态，但仍应继续显示内容。因此，您应确保您的应用在 onPause() 处理程序中进行处理时不会暂停播放。相反，您应在 onStop() 中暂停播放视频，并在 onStart() 中继续播放。如需了解详细信息，请参阅多窗口生命周期。

##### 可下载字体
Android 8.0 和 Android 支持库 26 允许您从提供程序应用请求字体，而无需将字体绑定到 APK 中或让 APK 下载字体。此功能可减小 APK 大小，提高应用安装成功率，使多个应用可以共享同一种字体。

##### XML 中的字体
Android 8.0 推出一项新功能，即 XML 中的字体，允许您使用字体作为资源。

##### 自动调整 TextView 的大小

##### 自适应图标
Android 8.0 引入自适应启动器图标。自适应图标支持视觉效果，可在不同设备型号上显示为各种不同的形状。

##### 颜色管理
图像应用的 Android 开发者现在可以利用支持广色域彩色显示的新设备。

##### WebView API
Android 8.0 提供多种 API，帮助您管理在应用中显示网页内容的 WebView 对象。这些 API 可增强应用的稳定性和安全性，它们包括：

- Version API
- Google SafeBrowsing API
- Termination Handle API
- Renderer Importance API

##### 固定快捷方式和小部件
##### 最大屏幕纵横比
以 Android 7.1（API 级别 25）或更低版本为目标平台的应用默认的最大屏幕纵横比为 1.86。针对 Android 8.0 或更高版本的应用没有默认的最大纵横比。如果您的应用需要设置最大纵横比，请使用定义您的操作组件的清单文件中的 maxAspectRatio 属性。

##### 多显示器支持

##### 统一的布局外边距和内边距
- layout_marginVertical，同时定义 layout_marginTop 和 layout_marginBottom。
- layout_marginHorizontal，同时定义 layout_marginLeft 和 layout_marginRight。
- paddingVertical，同时定义 paddingTop 和 paddingBottom。
- paddingHorizontal，同时定义 paddingLeft 和 paddingRight。

##### 指针捕获
##### 应用类别
##### AnimatorSet
从 Android 8.0 开始，AnimatorSet API 现在支持寻道和倒播功能。

#### 系统
##### 新的 StrictMode 检测程序
##### 缓存数据
##### 内容提供程序分页
##### 内容刷新请求
##### JobScheduler 改进
Android 8.0 引入了对 JobScheduler 的多项改进。由于您通常可以使用计划作业替代现在受限的后台服务或隐式广播接收器，这些改进可以让您的应用更轻松地符合新的后台执行限制。

##### 自定义数据存储
##### findViewById() 签名变更
现在，findViewById() 函数的全部实例均返回 <T extends View> T，而不是 View。





## 9.0 (Pie API:28)
https://developer.android.google.cn/about/versions/pie

##### 利用 Wi-Fi RTT 进行室内定位
Android 9 添加了对 IEEE 802.11mc Wi-Fi 协议（也称为 Wi-Fi Round-Trip-Time (RTT)）的平台支持，从而让您的应用可以利用室内定位功能。

##### 显示屏缺口支持
Android 9 支持最新的全面屏，其中包含为摄像头和扬声器预留空间的屏幕缺口。 通过 DisplayCutout 类可确定非功能区域的位置和形状，这些区域不应显示内容。 要确定这些屏幕缺口区域是否存在及其位置，请使用 getDisplayCutout() 函数。

##### 通知
###### 提升短信体验
###### 渠道设置、广播和请勿打扰
###### 多摄像头支持和摄像头更新
在运行 Android 9 的设备上，您可以通过两个或更多物理摄像头来同时访问多个视频流。] 在配备双前置摄像头或双后置摄像头的设备上，您可以创建只配备单摄像头的设备所不可能实现的创新功能，例如无缝缩放、背景虚化和立体成像。 通过该 API，您还可以调用逻辑或融合的摄像头视频流，该视频流可在两个或更多摄像头之间自动切换。

##### 适用于可绘制对象和位图的 ImageDecoder
Android 9 引入了 ImageDecoder 类，可提供现代化的图像解码方法。 使用该类取代 BitmapFactory 和 BitmapFactory.Options API。

##### 动画
Android 9 引入了 AnimatedImageDrawable 类，用于绘制和显示 GIF 和 WebP 动画图像。

##### HDR VP9 视频、HEIF 图像压缩和 Media API

##### JobScheduler 中的流量费用敏感度

##### Neural Networks API 1.1
Android 8.1（API 级别 27）中引入了 Neural Networks API 以加快 Android 设备上机器学习的速度。 Android 9 扩展和改进了该 API，增加了对九种新运算的支持

##### 自动填充框架

##### 安全增强功能
###### Android Protected Confirmation

###### 统一生物识别身份验证对话框
在 Android 9 中，系统代表您的应用提供生物识别身份验证对话框。 该功能可创建标准化的对话框外观、风格和位置，让用户更加确信，他们在使用可信的生物识别凭据检查程序进行身份验证。

###### 硬件安全性模块
运行 Android 9 或更高版本的受支持设备可拥有 StrongBox Keymaster，它是位于硬件安全性模块中的 Keymaster HAL 的一种实现。 该模块包含以下组成部分：
- 自己的 CPU。
- 安全存储空间。
- 真实随机数生成器。
- 可抵御软件包篡改和未经授权线刷应用的附加机制。

检查存储在 StrongBox Keymaster 中的密钥时，系统会通过可信执行环境 (TEE) 证实密钥的完整性。

###### 保护对密钥库进行的密钥导入
Android 9 通过利用 ASN.1‑编码密钥格式将已加密密钥安全导入密钥库的功能，提高了密钥解密的安全性。 Keymaster 随后会在密钥库中将密钥解密，因此密钥的内容永远不会以明文形式出现在设备的主机内存中。

###### 具有密钥轮转的 APK 签名方案
###### 只允许在未锁定设备上进行密钥解密的选项

##### Android 备份

##### 旋转
一个新的旋转模式允许用户在必要时利用系统栏上的一个按钮手动触发旋转。

##### 文本
- 文本预先计算：PrecomputedText 类使您能提前计算和缓存所需信息，改善了文本渲染性能。 它还使您的应用可以在主线程之外执行文本布局。

- 放大器：Magnifier 类是一种可提供放大器 API 的微件，可在所有应用中实现一致的放大器功能体验。

- Smart Linkify：Android 9 增强了 TextClassifier 类，该类可利用机器学习在选定文本中识别一些实体并建议采取相应的操作。 例如，TextClassifier 可以让您的应用检测到用户选择了电话号码。 然后，您的应用可以建议用户使用该号码拨打电话。 TextClassifier 中的功能取代了 Linkify 类的功能。

- 文本布局：借助几种便捷函数和属性，可以更轻松地实现界面设计。 如需了解详细信息，请参阅 TextView 参考文档。

##### DEX 文件的 ART 提前转换


