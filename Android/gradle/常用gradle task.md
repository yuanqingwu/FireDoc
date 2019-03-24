### 安装并启动APP
com.android.application 自带 installDebug task，开发者可以使用 installDebug 安装当前项目 apk：

##### 安装的时候能够顺带启动该 app:
```
task installAndRun(dependsOn: 'assembleDebug') {
  doFirst {
    exec {
      workingDir "${buildDir}/outputs/apk/debug"
      commandLine 'adb', 'install', '-r', 'app-debug.apk'
    }
    exec {
      def path = "${buildDir}/intermediates/manifests/full/debug/AndroidManifest.xml"
      // xml 解析
      def parser = new XmlParser(false, false).parse(new File(path))
      // application 下的每一个 activity 结点
      parser.application.activity.each { activity ->
        // activity 下的每一个 intent-filter 结点
        activity.'intent-filter'.each { filter ->
          // intent-filter 下的 action 结点中的 @android:name 包含 android.intent.action.MAIN
          if (filter.action.@"android:name".contains("android.intent.action.MAIN")) {
            def targetActivity = activity.@"android:name"
            commandLine 'adb', 'shell', 'am', 'start', '-n',
                "${android.defaultConfig.applicationId}/${targetActivity}"
          }
        }
      }
    }
  }
}
```

