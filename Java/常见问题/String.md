## Java中的String，StringBuilder，StringBuffer三者的区别

String的值是不可变的，这就导致每次对String的操作都会生成新的String对象，不仅效率低下，而且浪费大量优先的内存空间

StringBuffer是可变类，和线程安全的字符串操作类，任何对它指向的字符串的操作都不会产生新的对象。每个StringBuffer对象都有一定的缓冲区容量，当字符串大小没有超过容量时，不会分配新的容量，当字符串大小超过容量时，会自动增加容量

和 String 类不同的是，StringBuffer 和 StringBuilder 类的对象能够被多次的修改，并且不产生新的未使用对象。
StringBuilder 类在 Java 5 中被提出，它和 StringBuffer 之间的最大不同在于 StringBuilder 的方法不是线程安全的（不能同步访问）。

由于 StringBuilder 相较于 StringBuffer 有速度优势，所以多数情况下建议使用 StringBuilder 类。然而在应用程序要求线程安全的情况下，则必须使用 StringBuffer 类。


## 多线程String
JVM内存区域里面有一块常量池，关于常量池的分配：

1. JDK6的版本，常量池在持久代PermGen中分配
2. JDK7的版本，常量池在堆Heap中分配

字符串是存储在常量池中的，有两种类型的字符串数据会存储在常量池中：

1. 编译期就可以确定的字符串，即使用""引起来的字符串，比如String a = "123"、String b = "1" + B.getStringDataFromDB() + "2" + C.getStringDataFromDB()、这里的"123"、"1"、"2"都是编译期间就可以确定的字符串，因此会放入常量池，而B.getStringDataFromDB()、C.getStringDataFromDB()这两个数据由于编译期间无法确定，因此它们是在堆上进行分配的
2. 使用String的intern()方法操作的字符串，比如String b = B.getStringDataFromDB().intern()，尽管B.getStringDataFromDB()方法拿到的字符串是在堆上分配的，但是由于后面加入了intern()，因此B.getStringDataFromDB()方法的结果，会写入常量池中

常量池中的String数据有一个特点：每次取数据的时候，如果常量池中有，直接拿常量池中的数据；如果常量池中没有，将数据写入常量池中并返回常量池中的数据。


在调用”ab”.intern()方法的时候会返回”ab”，但是这个方法会首先检查字符串池中是否有”ab”这个字符串，如果存在则返回这个字符串的引用，否则就将这个字符串添加到字符串池中，然会返回这个字符串的引用。

## String a="a"和String a=new String("a")的的关系和异同？

区别：

1、直接定义的String a = "a"是储存在 常量存储区中的字符串常量池中,始终只有一个内存地址被分配，之后都是String的copy。这种被称为‘字符串驻留’，所有的字符串都会在编译之后自动驻留。；new String("a")是存储在堆中；

2、常量池中相同的字符串只会有一个，但是new String()，每new一个对象就会在堆中新建一个对象，不管这个值是否相同；

String a = "a";  String b = "a"; a和 b都指向字符串常量池中的"a",所以 a==b 为 true;

String a = new String("a");  String b = new String("a");是会在堆中创建两个对象new String() “a”是常量池中的”a”，这两个对象的值都为 a,所以a==b 返回false；a.equals(b)返回true;

3、
String a = "a";在编译阶段就会在内存中创建；

String a = new String("a");是在运行时才会在堆中创建对象

而new String("abc")这样会直接在堆中创建新的对象，不会进入String常量池。要把这样的对象引用放入常量池中就涉及另一个String类的方法intern()，这个方法就是返回一个String对象的常量池引用。如果这个对象不在常量池中，就会把这个String对象放入常量池中并返回对应的对象引用。