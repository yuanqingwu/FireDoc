
## Android AAPT详解
https://www.jianshu.com/p/8d691b6bf8b4

https://www.jianshu.com/p/3cc131db2002

## R.java
每个资源文件在R中都是一个class，每个资源项名称都分配了一个id，id值是一个四字节无符号整数，格式是这样的：0xpptteeee，（p代表的是package，t代表的是type，e代表的是entry），最高字节代表Package ID，次高字节代表Type ID，后面两个字节代表Entry ID。

Package ID相当于是一个命名空间，限定资源的来源。Android系统当前定义了两个资源命令空间，其中一个系统资源命令空间，它的Package ID等于0x01，另外一个是应用程序资源命令空间，它的Package ID等于0x7f。所有位于[0x01, 0x7f]之间的Package ID都是合法的，而在这个范围之外的都是非法的Package ID。

Type ID是指资源的类型ID。资源的类型有animator、anim、color、drawable、layout、menu、raw、string和xml等等若干种，每一种都会被赋予一个ID。

Entry ID是指每一个资源在其所属的资源类型中所出现的次序。注意，不同类型的资源的Entry ID有可能是相同的，但是由于它们的类型不同，我们仍然可以通过其资源ID来区别开来。

