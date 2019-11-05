### 参考资料
https://source.android.google.cn/security/selinux


android 8.1 安全机制 — SEAndroid & SELinux：
https://blog.csdn.net/qq_19923217/article/details/81240027

Android 9 SELinux:
https://www.jianshu.com/p/e95cd0c17adc

## SELinux
SELinux是「Security-Enhanced Linux」的简称，是美国国家安全局「NSA=The National Security Agency」 和SCC（Secure Computing Corporation）开发的 Linux的一个扩张强制访问控制安全模块。原先是在Fluke上开发的，2000年以 GNU GPL 发布。

相比其他强制性访问控制系统，SELinux 有如下优势：

- 控制策略是可查询而非程序不可见的。
- 可以热更改策略而无需重启或者停止服务。
-  可以从进程初始化、继承和程序执行三个方面通过策略进行控制。
-  控制范围覆盖文件系统、目录、文件、文件启动描述符、端口、消息接口和网络接口。


安全增强型 Linux (SELinux) 是适用于 Linux 操作系统的强制访问控制 (MAC) 系统。作为 MAC 系统，它与 Linux 中用户非常熟悉的自主访问控制 (DAC) 系统不同。在 DAC 系统中，存在所有权的概念，即特定资源的所有者可以控制与该资源关联的访问权限。这种系统通常比较粗放，并且容易出现无意中提权的问题。MAC 系统则会在每次收到访问请求时都先咨询核心机构，再做出决定。

SELinux 已作为 Linux 安全模块 (LSM) 框架的一部分实现，该框架可识别各种内核对象以及对这些对象执行的敏感操作。其中每项操作要执行时，系统都会调用 LSM 钩子函数，以便根据不透明安全对象中存储的关于相应操作的信息来确定是否应允许执行相应操作。SELinux 针对这些钩子以及这些安全对象的管理提供了相应的实现，该实现可结合自己的政策来决定是否允许相应访问。

在 Android 4.3 及更高版本中，SELinux 开始为传统的自主访问控制 (DAC) 环境提供强制访问控制 (MAC) 保护功能。例如，软件通常情况下必须以 Root 用户帐号的身份运行，才能向原始块设备写入数据。在基于 DAC 的传统 Linux 环境中，如果 Root 用户遭到入侵，攻击者便可以利用该用户身份向每个原始块设备写入数据。不过，可以使用 SELinux 为这些设备添加标签，以便被分配了 Root 权限的进程可以只向相关政策中指定的设备写入数据。这样一来，该进程便无法重写特定原始块设备之外的数据和系统设置。



Android 安全模型部分基于应用沙盒的概念。每个应用都在自己的沙盒内运行。在 Android 4.3 之前的版本中，这些沙盒是通过为每个应用创建独一无二的 Linux UID（在应用安装时创建）来定义的。 Android 4.3 及更高版本使用 SELinux 进一步定义 Android 应用沙盒的边界。

在 Android 5.0 及更高版本中，已全面强制执行 SELinux（基于 Android 4.3（宽容模式）和 Android 4.4（部分强制模式））。 通过此项变更，Android 已从对有限的一组关键域（installd、netd、vold 和 zygote）强制执行 SELinux 转为对所有域（超过 60 个域）强制执行 SELinux。具体而言：

    在 Android 5.x 及更高版本中，所有域均处于强制模式。
    init 以外的任何进程都不应在 init 域中运行。
    如果出现任何常规拒绝事件（对于 block_device、socket_device、default_service 等），都表示设备需要一个特殊域。


SELinux 按照默认拒绝的原则运行：任何未经明确允许的行为都会被拒绝。SELinux 可按两种全局模式运行：

    宽容模式：权限拒绝事件会被记录下来，但不会被强制执行。
    强制模式：权限拒绝事件会被记录下来并强制执行。

Android 中包含 SELinux（处于强制模式）和默认适用于整个 AOSP 的相应安全政策。在强制模式下，非法操作会被阻止，并且尝试进行的所有违规行为都会被内核记录到 dmesg 和 logcat 中。

#### 强制执行级别
SELinux 可以在各种模式下实现：

- 宽容模式 - 仅记录但不强制执行 SELinux 安全政策。
- 强制模式 - 强制执行并记录安全政策。如果失败，则显示为 EPERM 错误。 

在选择强制执行级别时只能二择其一，您的选择将决定您的政策是采取操作，还是仅允许您收集潜在的失败事件。宽容模式在实现过程中尤其有用。

#### 标签、规则和域
SELinux 依靠标签来匹配操作和政策。标签用于决定允许的事项。套接字、文件和进程在 SELinux 中都有标签。SELinux 在做决定时需参照两点：一是为这些对象分配的标签，二是定义这些对象如何交互的政策。

在 SELinux 中，标签采用以下形式：user:role:type:mls_level，其中 type 是访问决定的主要组成部分，可通过构成标签的其他组成部分进行修改。对象会映射到类，对每个类的不同访问类型由权限表示。

政策规则采用以下形式：allow domains types:classes permissions;，其中：

- Domain - 一个进程或一组进程的标签。也称为域类型，因为它只是指进程的类型。
- Type - 一个对象（例如，文件、套接字）或一组对象的标签。
- Class - 要访问的对象（例如，文件、套接字）的类型。
- Permission - 要执行的操作（例如，读取、写入）。 

使用政策规则时将遵循的结构示例：
```
allow appdomain app_data_file:file rw_file_perms;
```
这表示所有应用域都可以读取和写入带有 app_data_file 标签的文件。请注意，该规则依赖于在 global_macros 文件中定义的宏，您还可以在 te_macros 文件中找到一些其他非常实用的宏。其中提供了一些适用于常见的类、权限和规则分组的宏。应尽可能使用这些宏，以便降低因相关权限被拒而导致失败的可能性。这些宏文件位于 system/sepolicy 目录中。在 Android 8.0 及更高版本中，它们位于 public 子目录中（其中包含其他受支持的公共 sepolicy）。

#### 安全上下文
前面已经描述过了，SEAndroid 是一种基于安全策略的 MAC 安全机制。这种安全策略又是建立在对象的安全上下文的基础上的，SEAndroid 中的对象分为主体（Subject）和客体（Object），主体通常是指进程，而客体是指进程所要访问的资源，如文件、系统属性等

安全上下文实际上就是一个附加在对象上的标签（Tag）。这个标签实际上就是一个字符串，它由四部分内容组成，分别是 SELinux 用户、SELinux 角色、类型、安全级别，每一个部分都通过一个冒号来分隔，格式为 “user:role:type:sensitivity”
```
通过adb shell ls –Z指令查看文本的sContext

通过adb shell ps -Z指令可以查看进程的sContext
```
