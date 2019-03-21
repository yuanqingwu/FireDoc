cover:
https://blogs.oracle.com/java-platform-group/introducing-java-se-11


自从Java 10（作为六个月发布节奏的一部分的第一个功能版本）以来已经过去了六个月，Oracle现在提供Java 11。 

Oracle不仅在Oracle OpenJDK版本下使用开源GNU通用公共许可证v2，使用类路径异常（GPLv2 + CPE），而且在使用Oracle JDK作为Oracle产品的一部分的商业许可下提供JDK或服务，或不希望使用开源软件的人。  这些许可证取代了历史悠久的“BCL”许可证，该许可证包含免费和付费商业条款。

这意味着用户可以使Java 11满足他们的需求：

- Java 11是一个长期支持（LTS）版本。这意味着对平台采用保守且需要长期支持的用户可以通过Java SE订阅产品许可Oracle JDK二进制文件。它允许用户获得Java 11 LTS版本的更新至少八年。该订阅可直接从Oracle访问经过测试和认证的Java SE性能，稳定性和安全性更新。它还包括全天候访问My Oracle Support（MOS），支持27种语言，Java SE 8桌面管理，监控和部署功能，以及其他优势。
- 喜欢快速访问新增强功能的用户可以继续使用Oracle OpenJDK版本。与Java 9和Java 10一样，此版本的用户可以通过Oracle提供经过全面测试的开源OpenJDK构建版本。

Java 11中提供了17项增强功能，其中最值得注意的是：

- JEP 321 - HTTP客户端（标准）：此JEP通过JEP 110标准化JDK 9中引入的孵化HTTP客户端API，并在JDK 10中进行更新。
- JEP 332 - 传输层安全性（TLS）1.3： TLS 1.3是TLS协议的重大改进，与以前的版本相比，它提供了显着的安全性和性能改进。
- JEP 328 - Java飞行记录器（JFR）：JFR提供高性能飞行记录引擎和低开销数据收集框架，用于对任务关键型Java应用程序进行故障排除。
- JEP 333 - ZGC项目：ZGC是一个实验性但可预测的低延迟垃圾收集器（GC），可以处理从相对较小（几百兆字节）到非常大（几兆兆字节）大小的堆。
- JEP 330 - 启动单文件源代码程序：此增强功能通过增强java启动程序来运行作为单个Java源代码文件提供的程序，包括脚本中的使用，简化了“入口”或新Java用户和/或相关技术。

现在Java 11已经普及，开发已经转移到Java 12（计划于2019年3月交付）形式的下一个为期六个月的功能发布，目前有两个有针对性的增强功能，并且随着工作的完成需要添加更多功能。


1. JEP 181：基于嵌套的访问控制
2. JEP 309：动态类 - 文件常量
3. JEP 315：改进Aarch64内在函数
4. JEP 318：Epsilon：无操作垃圾收集器
5. JEP 320：移除Java EE和CORBA模块
6. JEP 321：HTTP客户端（标准）
7. JEP 323：本地变量Lambda参数的语法
8. JEP 324：与Curve25519和Curve448的密钥协议
9. JEP 327：Unicode 10
10. JEP 328：飞行记录器
11. JEP 329：ChaCha20和Poly1305加密算法
12. JEP 330：启动单文件源代码程序
13. JEP 331：低开销堆分析
14. JEP 332：传输层安全性（TLS）1.3
15. JEP 333：ZGC：可扩展的低延迟垃圾收集器（实验性）
16. JEP 335：弃用Nashorn JavaScript引擎
17. JEP 336：弃用Pack200工具和API