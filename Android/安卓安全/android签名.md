## APK签名
通过应用签名，开发者可以标识应用创作者并更新其应用，而无需创建复杂的接口和权限。在 Android 平台上运行的每个应用都必须有开发者的签名。

在 Android 上，应用签名是将应用放入其应用沙盒的第一步。已签名的应用证书定义了哪个用户 ID 与哪个应用相关联；不同的应用要以不同的用户 ID 运行。应用签名可确保一个应用无法访问任何其他应用的数据，通过明确定义的 IPC 进行访问时除外。

当应用（APK 文件）安装到 Android 设备上时，软件包管理器会验证 APK 是否已经过适当签名（已使用 APK 中包含的证书签名）。如果该证书（或更准确地说，证书中的公钥）与设备上的任何其他 APK 使用的签名密钥一致，那么这个新 APK 就可以选择在清单中指定它将与其他以类似方式签名的 APK 共用一个 UID。

应用可以由第三方（OEM、运营商、其他应用市场）签名，也可以自行签名。Android 提供了使用自签名证书进行代码签名的功能，而开发者无需外部协助或许可即可生成自签名证书。应用并非必须由核心机构签名。Android 目前不对应用证书进行 CA 认证。 

### APK 签名方案
 Android 支持以下三种应用签名方案：

-  v1 方案：基于 JAR 签名。
-  v2 方案：APK 签名方案 v2（在 Android 7.0 中引入）。
-  v3 方案：APK 签名方案 v3（在 Android 9 中引入）。

为了最大限度地提高兼容性，请按照 v1、v2、v3 的先后顺序采用所有方案对应用进行签名。与只通过 v1 方案签名的应用相比，还通过 v2+ 方案签名的应用能够更快速地安装到 Android 7.0 及更高版本的设备上。更低版本的 Android 平台会忽略 v2+ 签名，这就需要应用包含 v1 签名。 


##### JAR 签名（v1 方案）
https://docs.oracle.com/javase/8/docs/technotes/guides/jar/jar.html#Signed_JAR_File

v1 签名不保护 APK 的某些部分，例如 ZIP 元数据。APK 验证程序需要处理大量不可信（尚未经过验证）的数据结构，然后会舍弃不受签名保护的数据。这会导致相当大的受攻击面。此外，APK 验证程序必须解压所有已压缩的条目，而这需要花费更多时间和内存。为了解决这些问题，Android 7.0 中引入了 APK 签名方案 v2。 

##### APK 签名方案 v2 和 v3（v2+ 方案）
 在验证期间，v2+ 方案会将 APK 文件视为 Blob，并对整个文件进行签名检查。对 APK 进行的任何修改（包括对 ZIP 元数据进行的修改）都会使 APK 签名作废。这种形式的 APK 验证不仅速度要快得多，而且能够发现更多种未经授权的修改。

新的签名格式向后兼容，因此，使用这种新格式签名的 APK 可在更低版本的 Android 设备上进行安装（会直接忽略添加到 APK 的额外数据），但前提是这些 APK 还带有 v1 签名。 




# 重新签名
## 指令
jarsigner -verbose -keystore SignDemo.jks -signedjar MyApp_signed.apk myapp.apk keyAlias

## 签名文件转化
1.keystore文件转化为pk8+pem

keytool -importkeystore -srckeystore my.keystore -destkeystore tmp.p12 -srcstoretype JKS -deststoretype PKCS12

2. 将PKCS12 dump成pem

openssl pkcs12 -in tmp.p12 -nodes -out tmp.rsa.pem
tmp.rsa.pem 是文本格式可以直接查看。
打开文本可以看到私钥(PRIVATE KEY )和证书(CERTIFICATE）；

复制“BEGIN CERTIFICATE” “END CERTIFICATE” 到（新建个文件） cert.x509.pem

复制 “BEGIN RSA PRIVATE KEY” “END RSA PRIVATE KEY” 到（同上） private.rsa.pem

cert.x509.pem 文件即是我们最后需要的证书文件

3.生成pk8格式的私钥
openssl pkcs8 -topk8 -outform DER -in private.rsa.pem -inform PEM -out private.pk8 -nocrypt

最后的cert.x509.pem private.pk8即是我们最后需要的文件。

*备注:
-nocrypt 这个参数设定key加密 如果设置了这个参数 下面签名 只要证书+key 不需要密码了 如果加密 应该
openssl pkcs8 -topk8 -outform
DER -in private.rsa.pem -inform PEM -out private.pk8 接下来输入密码*


4.用法
java -jar signapk.jar cert.x509.pem private.pk8 unsigned.apk signed.apk