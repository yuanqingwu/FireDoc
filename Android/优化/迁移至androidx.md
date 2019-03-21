## 了解AndroidX
谷歌的文档工作做得比国内任何一家互联网公司都要好，直接参考安卓开发者网站：[AndroidX Overview ](https://developer.android.google.cn/jetpack/androidx/)

## 迁移到AndroidX
[Migrating to AndroidX ](https://developer.android.google.cn/jetpack/androidx/migrate)

1. 保存当前工作空间状态，万一迁移出错可以恢复当前状态。当然Android Studio已经想到了这一点，在开始迁移（migrate）之前可以选择帮你打一个zip包进行另存（Backup project as Zip file）。
2. 如果当前Android Studio是3.2及以上版本，直接点菜单栏Refactor > Migrate to AndroidX

## 报错及修改：

1. You need to have at least have compileSdk 28 set in your module build.gradle to migrate to AndroidX.

之前创建的老项目依赖版本一般都不会到28，故需要修改到最新的28，新创建的项目也得注意依赖版本得是28以上才行。

修改：build.gradle中 

```
compileSdkVersion 28
```

2. The gradle plugin version in your project build.gradle file needs to be set to at least com.android.tools.build:gradle:3.2.0 in order to migrate to AndroidX.

同理保证gradle插件是最新版本。

修改： classpath 'com.android.tools.build:gradle:3.2.0' 

报错如下：
```
Minimum supported Gradle version is 4.6. Current version is 4.4.
```
修改：

```
distributionUrl=https\://services.gradle.org/distributions/gradle-4.6-all.zip
```

3. 找不到android.support.annotation.CallSuper; android.support.annotation.UiThread

以上提醒事项都完成之后编译还是出错了。发现在build目录自动生成的所有XXXactivity_ViewBinding都报找不到注解 @UiThread @CallSuper，看这个自动生成类的名字大概就可以猜到是ButterKnife出问题了

想到之前看到过JakeWharton早就为ButterKnife兼容了androidX,
找到ButterKnife的仓库看到已经升级到9.0.0-rc1了，升级至9.0.0-rc1后再次编译一次。

4. 此时已经可以运行了，但是build还是会报错
```
ERROR: [TAG] Failed to resolve variable '${animal.sniffer.version}'

ERROR: [TAG] Failed to resolve variable '${junit.version}'
```
上网一查：[stackoverflow](https://stackoverflow.com/questions/51797219/failed-to-resolve-variable-animal-sniffer-version-when-migrate-to-androidx)

直接File->Invalidate Caches / restart,不再报错了。

- 如果是老项目中引入了第三方框架，在gradle.properties已经自动为我们添加了两个配置：

```
With Android Studio 3.2 and higher, you can quickly migrate an existing project to use AndroidX by selecting Refactor > Migrate to AndroidX from the menu bar.

If you have any Maven dependencies that have not been migrated to the AndroidX namespace, the Android Studio build system also migrates those dependencies for you when you set the following two flags to true in your gradle.properties file:

android.useAndroidX=true
android.enableJetifier=true

To migrate an existing project that does not use any third-party libraries with dependencies that need converting, you can set the android.useAndroidX flag to true and the android.enableJetifier flag to false.
```


### 修正优化gradle中的版本
如果你的工程中将version都集中到了versions.gradle文件中或其他地方，android studio的自动迁移功能只会修改build.gradle文件中的相应版本，所以需要自己再次整理gradle文件。



```
com.android.support:design 	com.google.android.material:material:1.0.0-rc01
```

