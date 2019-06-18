
### 创建插件
##### 1. 新建一个Java Library的module(也可以是Android Library).

##### 2. 修改build.gradle

```
apply plugin: 'groovy'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
}

repositories {
    mavenCentral()
}

```


##### 3. 建立plugin的目录结构及实现插件

把java包名改为groovy，插件代码放在 src/main/groovy下

如果没有resources目录则新建，在src/main/resources/META-INF/gradle-plugins 里声明plugin信息：

文件名为：插件名.properties; 如fireplugin.properties
```
implementation-class=com.wyq.firehelper.plugin.MyPlugin
```
这里com.wyq.firehelper.plugin.MyPlugin为插件实现类（实现Plugin<Project> 接口）：

```
package com.wyq.firehelper.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project

class MyPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {

        println "MyPlugin exec"

        project.task("plugin test"){
            doLast{
                println "hello gradle!"
            }
        }
    }
}
```
##### 4.使用：
build 之后就会在module/build/libs包下生成对应插件的jar包。

在需要使用的地方apply编写的插件即可（名称为之前编写的properties文件的名称）：

```
apply plugin:'fireplugin'
```


### 插件发布

##### 1.引入 mavenDeplayer插件，修改build.gradle如下：
```
apply plugin: 'groovy'
//添加maven plugin, 用于发布我们的jar
apply plugin: 'maven'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
}

repositories {
    mavenCentral()
}

uploadArchives {
    repositories {
        mavenDeployer{
            pom.groupId = 'com.wyq.firehelper'
            pom.artifactId = 'fireplugin'
            pom.version = 0.1

            //暂时发布在本地文件夹
            repository(url:uri('plugin/release'))
        }
    }
}
```

##### 用uploadArchices发布
运行uploadArchives. 就能在设置的仓库路径中生成对应的插件了
