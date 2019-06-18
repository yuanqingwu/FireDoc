# 虚拟机发展史
- Sun Classic/Exact VM
 

sun jdk 版本 | 虚拟机
---|---
1.0-1.2 | 1996年1月23日Sun公司发布JDK1.0,所带的虚拟机就是Classic VM。在JDK1.2之前是Sun JDK中唯一的虚拟机。
1.2 | 默认Classic VM，可通过java-hotspot切换到HotSpot VM
1.3 | 默认HotSpot VM，Classic VM作为备用选择（可使用java-classic切换）
1.4 | Classic VM退出历史舞台，与Exact VM一起进入了Sun Labs Research VM中



- Sun HotSpot VM

最初由一家叫Longview Technologies的小公司设计，1997年Sun收购此公司而获得HotSpot VM.

名称中的HotSpot指的是热点代码探测技术


- BEA JRockit VM

# 运行时数据区域
- 程序计数器
- Java虚拟机栈
- 本地方法栈
- Java堆
- 方法区
- 运行时常量池
- 直接内存