https://developer.android.google.cn/topic/performance/memory-overview.html#gc

https://developer.android.google.cn/topic/performance/memory

[Android内存优化之OOM](http://hukai.me/android-performance-oom/)

https://www.jianshu.com/p/c4b283848970




## Shallow Size、Retained Size、Heap Size 和 Allocated

### Shallow Size
Shallow size就是对象本身占用内存的大小，不包含其引用的对象。

- 常规对象（非数组）的 Shallow size 由其成员变量的数量和类型决定。
-  数组的shallow size有数组元素的类型（对象类型、基本类型）和数组长度决定。


### Retained Size
对象的Retained Size = 对象本身的Shallow Size + 对象能直接或间接访问到的对象的Shallow Size
也就是说 Retained Size 就是该对象被 Gc 之后所能回收内存的总和。

为了更好地理解Retained Size，看下图对象的引用对象可以归纳为：该对象到其他对象有引用关系并且该引用对象到 Gc Root 节点是不可达的

### Heap Size:

堆的大小，当资源增加，当前堆的空间不够时，系统会增加堆的大小，若超过上限（如64M，阈值视平台而定）则会被杀掉 。


### Allocated:

堆中已分配的大小，即 App 应用实际占用的内存大小，资源回收后，此项数据会变小。


### Depth：
从任意 GC 根到所选实例的最短 hop 数。

### native size：
8.0之后的手机会显示，主要反应Bitmap所使用的像素内存（8.0之后，转移到了native）

