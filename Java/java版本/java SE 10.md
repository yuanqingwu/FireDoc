https://blogs.oracle.com/java-platform-group/introducing-java-se-10

http://openjdk.java.net/projects/jdk/10/

In 2017, Oracle and the Java community announced its intentions to shift to a new six-month cadence for Java meant to reduce the latency between major releases.  At the same time Oracle announced it's plans to build and ship OpenJDK binaries as well. This release model takes inspiration from the release models used by other platforms and by various operating-system distributions addressing the modern application development landscape.  The pace of innovation is happening at an ever-increasing rate and this new release model will allow developers to leverage new features in production as soon as possible.  Modern application development expects simple open licensing and a predictable time-based cadence, and the new release model delivers on both.

With that said, Oracle is pleased to announce the general availability of Java 10, the first time-bound release as part of the new six-month release cycle.  This release is more than a simple stability and performance fix over Java SE 9, rather it introduces twelve new enhancements defined through the JDK Enhancement Proposals (JEPS) that developers can immediate pick up and start using:

1. (JEP 286) Local-Variable Type Inference: Enhances the Java Language to extend type inference to declarations of local variables with initializers. It introduces var to Java, something that is common in other languages.
2. (JEP 296) Consolidate the JDK Forest into a Single Repository: Combine the numerous repositories of the JDK forest into a single repository in order to simplify and streamline development.
3. (JEP 204) Garbage Collector Interface: Improves the source code isolation of different garbage collectors by introducing a clean garbage collector (GC) interface.
4. (JEP 307) Parallel Full GC for G1: Improves G1 worst-case latencies by making the full GC parallel.
5. (JEP 301) Application Data-Class Sharing: To improve startup and footprint, this JEP extends the existing Class-Data Sharing ("CDS") feature to allow application classes to be placed in the shared archive.
6. (JEP 312) Thread-Local Handshakes: Introduce a way to execute a callback on threads without performing a global VM safepoint. Makes it both possible and cheap to stop individual threads and not just all threads or none.
7. (JEP 313) Remove the Native-Header Generator Tool: Remove the javah tool from the JDK since it has been superseded by superior functionality in javac.
8. (JEP 314) Additional Unicode Language-Tag Extensions: Enhances java.util.Locale and related APIs to implement additional Unicode extensions of BCP 47 language tags.
9. (JEP 316) Heap Allocation on Alternative Memory Devices: Enables the HotSpot VM to allocate the Java object heap on an alternative memory device, such as an NV-DIMM, specified by the user.
10. JEP 317) Experimental Java-Based JIT Compiler: Enables the Java-based JIT compiler, Graal, to be used as an experimental JIT compiler on the Linux/x64 platform.
11. (JEP 319) Root Certificates: Provides a default set of root Certification Authority (CA) certificates in the JDK.
12. (JEP 322) Time-Based Release Versioning: Revises the version-string scheme of the Java SE Platform and the JDK, and related versioning information, for present and future time-based release models.
13. 




# 中文
2017年，甲骨文和Java社区宣布其意图转向新的六个月Java节奏，旨在减少主要版本之间的延迟。与此同时，甲骨文宣布计划构建和发布OpenJDK二进制文件。此发布模型从其他平台使用的发布模型以及解决现代应用程序开发领域的各种操作系统发行版中获得灵感。创新的步伐正在以不断增长的速度发生，这种新的发布模式将允许开发人员尽快利用生产中的新功能。现代应用程序开发期望简单的开放许可和可预测的基于时间的节奏，新版本模型可以实现这两者。

话虽如此，甲骨文很高兴地宣布Java 10的普遍可用性，这是新的六个月发布周期的第一个有时限版本。此版本不仅仅是针对Java SE 9的简单稳定性和性能修复，而是引入了通过JDK增强建议（JEPS）定义的12个新增强功能，开发人员可以立即启动并开始使用：

1. （JEP 286）局部变量类型推断：增强Java语言以使用初始化器将类型推断扩展为局部变量的声明。它将var引入Java，这在其他语言中很常见。
2. （JEP 296）将JDK Forest整合到单个存储库中：将JDK林的众多存储库组合到单个存储库中，以简化和简化开发。
3. （JEP 204）垃圾收集器接口：通过引入干净的垃圾收集器（GC）接口，改进了不同垃圾收集器的源代码隔离。
4. （JEP 307） G1的并行全GC：通过使完整GC并行来改善G1最坏情况延迟。
5. （JEP 301）应用程序数据类共享：为了改善启动和占用空间，此JEP扩展了现有的类数据共享（“CDS”）功能，以允许将应用程序类放在共享存档中。
6. （JEP 312）线程局部握手：介绍一种在线程上执行回调而不执行全局VM安全点的方法。使得停止单个线程既可能又便宜，而不仅仅是所有线程或没有线程。
7. （JEP 313）删除Native-Header生成器工具：从JDK中删除javah工具，因为它已被javac中的高级功能取代。
8. （JEP 314）其他Unicode语言标记扩展：增强java.util.Locale和相关API，以实现BCP 47语言标记的其他Unicode扩展。
9. （JEP 316）备用内存设备上的堆分配：使HotSpot VM能够在另一个内存设备（例如用户指定的NV-DIMM）上分配Java对象堆。
10. （JEP 317）基于实验Java的JIT编译器：使基于Java的JIT编译器Graal能够用作Linux / x64平台上的实验性JIT编译器。
11. （JEP 319）根证书：在JDK中提供一组默认的根证书颁发机构（CA）证书。
12. （JEP 322）基于时间的发布版本控制：针对当前和未来基于时间的发布模型，修订Java SE平台和JDK的版本字符串方案以及相关的版本控制信息。