##### 动态注册与静态注册
jni方法注册方式有两种：

- 通过javah产生函数头，该种方式产生的方法具有固定的格式且一般很长。初次调用Native方法时需要建立关联，影响效率。该方式使逆向分析人员比较容易获取java层native方法对应的本地方法。

- 在JNI_OnLoad方法中手动注册jni方法，不易查找对应关系。可以实现混淆java层native方法和so函数的对应关系。

###### 动态注册实现示例
JNI中使用JNINativeMethod结构体来记录java的Native方法和JNI方法的关联关系
```
typedef struct {
    const char* name;//java方法的名字
    const char* signature;//java方法的签名
    void*       fnPtr;//JNI中对应的方法指针
} JNINativeMethod;
```


```
//定义一个JNINativeMethod类型的数组gMethods，存储java的Native方法和JNI方法的对应关系：

static const JNINativeMethod gMethods[] = {
        {"loadDal", "(Ljava/lang/Object;)Ljava/lang/Object;", (void *) Java_com_pax_neptunelite_api_ApiLib_loadDal}
};

int register_native_methods(JNIEnv *env, const char *className,
                            JNINativeMethod *gMethods, int numMethods) {
    jclass clazz;
    clazz = env->FindClass(className);
    if (clazz == NULL) {
        return JNI_FALSE;
    }
    if (env->RegisterNatives(clazz, gMethods, numMethods) < 0) {
        return JNI_FALSE;
    }

    return JNI_TRUE;
}

int register_on_ndk_load(JNIEnv *env) {
    return register_native_methods(env, "com/pax/neptunelite/api/ApiLib", gMethods, NELEM(gMethods));
}

JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    ptrace(PTRACE_TRACEME, 0, 0, 0);//反调试

    JNIEnv *env = NULL;
    jint result = -1;

    if (vm->GetEnv((void **) &env, JNI_VERSION_1_4) != JNI_OK) {
        LOGI("ERROR: GetEnv failed\n");
        return result;
    }


    register_on_ndk_load(env);
    return JNI_VERSION_1_4;
}

```


##### 日志打印与控制

```
#include <android/log.h>

/**
 * 设置是否是测试状态，测试状态会打印log
 * 1:测试状态 0：发布状态
 */
#define is_debug 1
#define LOG_TAG "System.out.c"
#if (is_debug == 1)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,LOG_TAG ,__VA_ARGS__)
#else
#define LOGI(...) NULL
#endif
```

##### 方法名与方法签名
方法签名格式：(参数签名格式...)返回值签名格式

- 构造函数方法名为：<init> 方法签名为：(参数签名...)V

- 可变参数当数组处理

###### 使用javap生成方法签名

```
javac xxx.java
javap -s -p xxx.class

s:表示输出内部类型签名
p:表示打印出所有的方法和成员(默认打印public成员)
```

##### 对象引用类型
以”L”开头，以”;”结尾中间是用”/” 隔开的包及类名（如 Landroid/view/Surface;）如果内部类则使用$连接内部类；

不要忘记最后的;结尾

##### so反调试
同Linux反调试基本原理相同，这里提供一种方式就是在JNI_Onload中调用ptrace方法，ptrace被广泛用于调试（比如IDA）和进程代码注入（比如LBE,金山等权限管理功能实现），一个进程只能被一个进程ptrace，如果你自己调用ptarce，这样其它程序就无法通过ptrace调试或者向您的程序进程注入代码。

ptrace(PTRACE_TRACEME,0 ,0 ,0);


##### 指针参数，实参形参

##### string 与char*

##### 内存管理