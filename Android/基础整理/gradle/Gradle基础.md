https://docs.gradle.org/current/userguide/tutorial_using_tasks.html

https://docs.gradle.org/current/userguide/java_plugin.html

[Gradle Android插件用户指南翻译](https://avatarqing.github.io/Gradle-Plugin-User-Guide-Chinese-Verision/index.html)

[深入理解Android之Gradle](https://blog.csdn.net/innost/article/details/48228651)

[plugin 实战包体积瘦身](https://blog.csdn.net/ziwang_/article/details/80474952)

[Android 热修复 Tinker Gradle Plugin解析](https://blog.csdn.net/lmj623565791/article/details/72667669)
### Groovy 

[Gradle从入门到实战 - Groovy基础](https://blog.csdn.net/singwhatiwanna/article/details/76084580)

###### 字符串
1. 单引号''中的内容严格对应Java中的String，不对$符号进行转义
2. 双引号""的内容则和脚本语言的处理有点像，如果字符中有$号的话，则它会$表达式先求值。
3. 三个引号'''xxx'''中的字符串支持随意换行


###### 调用方法时括号是可选的。

###### List 和 Map
- List：链表，其底层对应Java中的List接口，一般用ArrayList作为真正的实现类。
- Map：键-值表，其底层对应Java中的LinkedHashMap。
- Range：范围，它其实是List的一种拓展。


```

变量定义：List变量由[]定义，比如
def aList = [5,'string',true] //List由[]定义，其元素可以是任何对象
变量存取：可以直接通过索引存取，而且不用担心索引越界。如果索引超过当前链表长度，List会自动
往该索引添加元素
assert aList[1] == 'string'
assert aList[5] == null //第6个元素为空
aList[100] = 100 //设置第101个元素的值为10
assert aList[100] == 100
那么，aList到现在为止有多少个元素呢？
println aList.size  ===>结果是101
```

```
容器变量定义
变量定义：Map变量由[:]定义，比如
def aMap = ['key1':'value1','key2':true]
Map由[:]定义，注意其中的冒号。冒号左边是key，右边是Value。key必须是字符串，value可以是任何对象。另外，key可以用''或""包起来，也可以不用引号包起来。比如
def aNewMap = [key1:"value",key2:true]//其中的key1和key2默认被
处理成字符串"key1"和"key2"
不过Key要是不使用引号包起来的话，也会带来一定混淆，比如
def key1="wowo"
def aConfusedMap=[key1:"who am i?"]
aConfuseMap中的key1到底是"key1"还是变量key1的值“wowo”？显然，答案是字符串"key1"。如果要是"wowo"的话，则aConfusedMap的定义必须设置成：
def aConfusedMap=[(key1):"who am i?"]
Map中元素的存取更加方便，它支持多种方法：
println aMap.keyName    <==这种表达方法好像key就是aMap的一个成员变量一样
println aMap['keyName'] <==这种表达方法更传统一点
aMap.anotherkey = "i am map"  <==为map添加新元素
```

```
def aRange = 1..5 <==Range类型的变量 由begin值+两个点+end值表示左边这个aRange包含1,2,3,4,5这5个值
如果不想包含最后一个元素，则
def aRangeWithoutEnd = 1..<5  <==包含1,2,3,4这4个元素
println aRange.from
println aRange.to
```


左移位(<<)表示向List中添加新元素

```
def list = [1,2]
list << 3
assert list == [1,2,3]
```

###### 闭包
闭包，英文叫Closure，是一种数据类型，它代表了一段可执行的代码。

```
def aClosure = {//闭包是一段代码，所以需要用花括号括起来..
    String param1, int param2 ->  //这个箭头很关键。箭头前面是参数定义，箭头后面是代码
    println"this is code" //这是代码，最后一句是返回值，
   //也可以使用return，和Groovy中普通函数一样
}

def xxx = {paramters -> code} //或者 def xxx = {无参数，纯code} 这种case不需要->符号


调用：
aClosure.call("this is string",100)  或者
aClosure("this is string", 100)
```
如果闭包没定义参数的话，则隐含有一个参数，这个参数名字叫it，和this的作用类似。it代表闭包的参数。


###### ==和equals
在Groovy中，==相当于Java的equals，，如果需要比较两个对象是否是同一个，需要使用.is()。

```
Object a = new Object()
Object b = a.clone()

assert a == b
assert !a.is(b)
```


###### 缓存
为了提高响应速度，默认情况下 Gradle 会缓存所有已编译的脚本。这包括所有构建脚本，初始化脚本和其他脚本。你第一次运行一个项目构建时， Gradle 会创建 .gradle 目录，用于存放已编译的脚本。下次你运行此构建时， 如果该脚本自它编译后没有被修改，Gradle 会使用这个已编译的脚本。否则该脚本会重新编译，并把最新版本存在缓存中。如果您通过 --recompile-scripts 选项运行 Gradle ，会丢弃缓存的脚本，然后重新编译此脚本并将其存在缓存中。通过这种方式，您可以强制 Gradle 重新生成缓存。

- 使用 mkdir 创建目录
- 通过使用方法 hasProperty('propertyName') 来进行检查项目的属性，它返回 true 或 false

### 任务
一个Gradle可包含多个Project，一个 Project可包含多个Task，即每个Project是由多个Task组成的；Task是Project的属性，属性名就是任务名

##### task 声明
task 的基本写法可以是如下四种：

```
task myTask
task myTask { configure closure }
task (myTask) { configure closure }
task (name: myTask) { configure closure }
```
调用上面的 task：

```
./gradlew myTask

如果不同Module下有名字重复task：
./gradlew app:myTask（调用 app module 的 myTask）

./gradlew a:myTask（调用 a module 的 myTask）
```
###### task 的创建是可以声明参数的，有如下几种：
- **name**：task的名字。不能为空，必须指定
- **type**：默认为 DefaultTask。类似于父类。
- **dependsOn**：默认为[]。希望依赖的 tasks，等同于 Task.dependsOn(Object... path) 中的 path。
- **action**：默认为 null。等同于 Task.doFirst { Action } 中的 Action。
- **override**：默认为 false。是否替换已存在的 task。
- **group**：默认为 null。task 的分组类型。
- **description**：默认为 null。task 描述。
- **constructorArgs**：默认为 null。传给 task 构造函数的参数。

###### 定义task
直接写在task中的代码会在配置阶段就执行，也就是说，只要我执行任何一个task，那段代码都会执行，因为每个task执行之前都需要进行一遍完整的配置。我们想要括号内的代码仅仅在执行我们的task的时候才执行，这个时候可以通过doFirst或者doLast来完成。
- doFirst：task执行时，最开始的操作
- doLast：task执行时，最后的操作

doLast还有一个等价的操作leftShift，leftShift还可以缩写为<<，因此下面的三种实现效果等价：

```
myTask.doLast {
    println "after execute myTask"
}

myTask.leftShift {
    println "after execute myTask"
}

myTask << {
    println "after execute myTask"
}
```
Gradle本身还提供了一些已有的Task供我们使用，比如Copy、Delete、Sync等。因此我们定义Task的时候是可以继承已有的Task，比如我们可以继承自系统的Copy Task来完成文件的拷贝操作。

```
task myTask(type: Copy) { configure closure }
```


##### 任务排序
###### 添加 '必须在之后运行 ' 的任务排序
build.gradle
```
task taskX << {
    println 'taskX'
}
task taskY << {
    println 'taskY'
}
taskY.mustRunAfter taskX

> gradle -q taskY taskX
taskX
taskY 
```
###### 添加 '应该在之后运行 ' 的任务排序

```
task taskX << {
    println 'taskX'
}
task taskY << {
    println 'taskY'
}
taskY.shouldRunAfter taskX  

> gradle -q taskY taskX
taskX
taskY 
```
###### 任务排序并不意味着任务执行

如果想指定两个任务之间的“必须在之后运行”和“应该在之后运行”排序，可以使用 Task.mustRunAfter() 和 Task.shouldRunAfter() 方法。这些方法接受一个任务实例、 任务名称或 Task.dependsOn()所接受的任何其他输入作为参数。

###### 当引入循环时，“应该在其之后运行”的任务排序会被忽略


##### 重写任务

```
task copy(type: Copy)
task copy(overwrite: true) << {
    println('I am the new one.')
}  

> gradle -q copy
I am the new one. 
```

### 文件

##### 查找文件

```
// Using a relative path
File configFile = file('src/config.xml')
// Using an absolute path
configFile = file(configFile.absolutePath)
// Using a File object with a relative path
configFile = file(new File('src/config.xml')) 
```

##### 文件树
文件树是按层次结构排序的文件集合。例如，文件树可能表示一个目录树或 ZIP 文件的内容。它通过 FileTree 接口表示。FileTree 接口继承自 FileCollection，所以你可以用对待文件集合一样的方式来对待文件树。Gradle 中的几个对象都实现了 FileTree 接口，例如 source sets。

使用 Project.fileTree()方法是获取一个 FileTree 实例的其中一种方法。它将定义一个基目录创建 FileTree 对象，并可以选择加上一些 Ant 风格的包含与排除模式。

```
// Create a file tree with a base directory
FileTree tree = fileTree(dir: 'src/main')
// Add include and exclude patterns to the tree
tree.include '**/*.java'
tree.exclude '**/Abstract*'
// Create a tree using path
tree = fileTree('src').include('**/*.java')
// Create a tree using closure
tree = fileTree('src') {
    include '**/*.java'
}
// Create a tree using a map
tree = fileTree(dir: 'src', include: '**/*.java')
tree = fileTree(dir: 'src', includes: ['**/*.java', '**/*.xml'])
tree = fileTree(dir: 'src', include: '**/*.java', exclude: '**/*test*/**')  
```
##### 复制文件

```
task copyTask(type: Copy) {
    from 'src/main/webapp'
    into 'build/explodedWar'
} 
```
##### 选择要复制的文件

```
task copyTaskWithPatterns(type: Copy) {
    from 'src/main/webapp'
    into 'build/explodedWar'
    include '**/*.html'
    include '**/*.jsp'
    exclude { details -> details.file.name.endsWith('.html') && details.file.text.contains('staging') }
}  
```
##### 使用有最新状态检查的 copy() 方法复制文件

```
task copyMethodWithExplicitDependencies{
    inputs.file copyTask // up-to-date check for inputs, plus add copyTask as dependency
    outputs.dir 'some-dir' // up-to-date check for outputs
    doLast{
        copy {
            // Copy the output of copyTask
            from copyTask
            into 'some-dir'
        }
    }
} 
```
##### 重命名复制的文件

```
task rename(type: Copy) {
    from 'src/main/webapp'
    into 'build/explodedWar'
    // Use a closure to map the file name
    rename { String fileName ->
        fileName.replace('-staging-', '')
    }
    // Use a regular expression to map the file name
    rename '(.+)-staging-(.+)', '$1$2'
    rename(/(.+)-staging-(.+)/, '$1$2')
}  
```
##### 创建归档文件
###### 创建一个 ZIP 文件

```
apply plugin: 'java'
task zip(type: Zip) {
    from 'src/dist'
    into('libs') {
        from configurations.runtime
    }
} 
```



### 构建生命周期
Gradle 构建周期分为：初始化，配置，执行三个阶段：

- **Initialization**：Gradle支持单个和多项目构建。在 Initialization 阶段，Gradle 将会确定哪些项目将参与构建，并为每个项目创建一个 Project 对象实例。对于 Android 项目来说即为执行 setting.gradle 文件。那么日常开发中 setting.gradle 文件是如何书写的呢，假设当前应用中除 app 以外还有一个 a module 和 b module，那么 setting.gradle 文件应类似如下：

```
include ':app', ':a', ':b'
```
Gradle 将会为它们三个分别创建一个 Project 对象实例。

- **Configuration**：在这一阶段项目配置对象，所有项目的构建脚本将会被「执行」，这样才能够知道各个 task 之间的依赖关系。需要说的一点是，这里提到的「执行」可能会稍有一些歧义：

```
task a {

}

task testBoth {
	// 依赖 a task 先执行
	dependsOn("a")
	println '我会在 Configuration 和 Execution 阶段都会执行'
    doFirst {
      println '我仅会在 testBoth 的 Execution 阶段执行'
    }
    doLast {
      println '我仅会在 testBoth 的 Execution 阶段执行'
    }
}
```
写在 task 闭包中的内容是会在 Configuration 中就执行的，例如上面的 dependsOn("a") 和 println 内容；而 doFirst {}/doLast {} 闭包中的内容是在 Execution 阶段才会执行到（doFirst {}/ doLast {} 实际上就是给当前 task 添加 Listener，这些 Listeners 只会在当前 task Execution 阶段才会执行）。

 配置完了以后，有一个重要的回调project.afterEvaluate，它表示所有的模块都已经配置完了，可以准备执行task了； 

- **Execution**：task 的执行阶段。首先执行 doFirst {} 闭包中的内容，最后执行 doLast {} 闭包中的内容。


![image](https://img-blog.csdn.net/2018051022062378?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ppd2FuZ18=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 项目结构

Gradle使用一个叫做源集(Source set) 的概念。Gradle的官方文档是这么解释的：一个源集就是一组文件，它们会被一起执行和编译。对于一个android项目而言，main就是一个源集，它包含了所有的源代码和资源，是应用程序默认版本的源集。当你开始为Android应用程序编写测试代码时，你可以把所有的测试相关的源代码都放在一个独立的源集中，该源集被叫做androidTest,它只包含测试代码。

### Gradle Wrapper
Gradle Wrapper为微软的Windows操作系统提供了一个batch文件，为其他操作系统提供了一个shell脚本。当你运行这段脚本时，需要的Gradle版本会被自动下载（如果不存在）和使用。其原理是，每个需要构建应用的开发者或自构建系统可以仅仅运行Wrapper,然后由Wrapper搞定剩余部分。

Gradle建议把Wrapper文件添加到你的版本控制系统中。

### 多项目构建

定义一个多项目构建工程需要在根目录创建一个setting 配置文件来指明构建包含哪些项目。并且这个文件必需叫 settings.gradle 本例的配置文件如下:

```
include "shared", "api", "services:webservice", "services:shared"
```
