

https://blog.csdn.net/abm1993/article/details/88818361


## 经验

#### 打开系统任务管理器
adb root
adb shell am start -n com.android.systemui/.recents.RecentsActivity

#### 系统快照保存路径
data/system_ce/0/snapshots

代码见：frameworks/base/services/core/java/com/android/server/wm/TaskSnapshotPersister.java


## AOSP源码分析

#### 核心类
base/services/core/java/com/android/server/wm/TaskSnapshotController.java