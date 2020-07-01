---
layout: post
title:  "Android jni知识点"
date:   2020-7-1 19:46:25 +0800
categories: Android
---

> 文章将会同步到个人微信公众号：Android部落格

## 1、创建jni环境

> https://developer.android.com/studio/projects/gradle-external-native-builds
>
> https://developer.android.com/training/articles/perf-jni
>
> https://www.jianshu.com/p/127adc130508
>
> https://www.jianshu.com/p/d2e3e5632cc9

### 1.1 配置方式
在使用jni代码之前需要先配置jni，不论使用ndk-build还是cmake都需要在build.gradle文件中配置：

#### 1.1.1 ndkBuild

> 这里以ndkBuild方式为例，cmake可以参照本章开始的第一个链接。

```
android {
    compileSdkVersion 28
    defaultConfig {
        externalNativeBuild {
            ndkBuild {
                // Specifies the ABI configurations of your native
                // libraries Gradle should build and package with your APK.
                abiFilters "armeabi-v7a", "arm64-v8a"
            }
        }
    }
    
    externalNativeBuild {
        ndkBuild {
            path 'src/main/java/jni/Android.mk'
        }
    }
}
```

#### 1.1.2 cmake

> https://developer.android.com/studio/projects/configure-cmake
>
> https://developer.android.com/studio/projects/add-native-code

在module根目录右键，创建file，名称是“CMakeLists.txt”，然后往里面添加配置内容：
```
cmake_minimum_required(VERSION 3.4.1)

add_library( # Specifies the name of the library.
        communicate

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        src/main/java/jni/DynamicRegister.c src/main/java/jni/coffeecatch.c src/main/java/jni/coffeejni.c)

find_library( # Defines the name of the path variable that stores the
        # location of the NDK library.
        log-lib

        # Specifies the name of the NDK library that
        # CMake needs to locate.
        log)

# Links your native library against one or more other native libraries.
target_link_libraries( # Specifies the target library.
        communicate

        # Links the log library to the target library.
        ${log-lib})
```
这里的communicate就是生成的so库名称，最终是libcommunicate.so。

在module目录下的build.gradle文件中添加cmake配置：
```
apply plugin: 'com.android.application'

android {
    compileSdkVersion 28
    defaultConfig {
        externalNativeBuild {
            cmake {
                version "3.10.2"
                abiFilters "armeabi-v7a", "arm64-v8a"
                arguments "-DANDROID_ARM_NEON=TRUE", "-DANDROID_TOOLCHAIN=clang"
            }
        }
    }
    
    externalNativeBuild {
        cmake {
            // Provides a relative path to your CMake build script.
            path "CMakeLists.txt"
        }
    }
}
```
android节点下的externalNativeBuild中，配置的path路径对应新建的CMakeLists.txt文件路径。

### 1.2 调用方式
分为java调用jni层和jni层调用java。下面的代码包括了一般情况下的调用方式：

```c
JNIEXPORT jstring
Java_com_example_chuckapptestdemo_TestJniActivity_communicate2Jni(JNIEnv *env, jobject obj, jstring message) {
    const char *nativeString = (*env)->GetStringUTFChars(env, message, 0);
    __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, "%s", nativeString);
    (*env)->ReleaseStringUTFChars(env, message, nativeString);

    invokeJavaMethod(env, obj);

    jstring response = (*env)->NewStringUTF(env, "jni say hi to java");
    return response;
}

void invokeJavaMethod(JNIEnv *env, jobject obj) {
    //实例化TestBeInvoked类
    jclass testclass = (*env)->FindClass(env, "com/example/chuckapptestdemo/TestBeInvoked");
    //构造函数的方法名为<init>
    jmethodID testcontruct = (*env)->GetMethodID(env, testclass, "<init>", "()V");
    //根据构造函数实例化对象
    jobject testobject = (*env)->NewObject(env, testclass, testcontruct);

    //调用成员方法，需使用jobject对象
    jmethodID test = (*env)->GetMethodID(env, testclass, "test", "(I)V");
    (*env)->CallVoidMethod(env, testobject, test, 1);

    //调用静态方法
    jmethodID testStatic = (*env)->GetStaticMethodID(env, testclass, "testStatic",
                                                     "(Ljava/lang/String;)V");
    //创建字符串，不能在CallStaticVoidMethod中直接使用"hello world!"，会报错的
    jstring str = (*env)->NewStringUTF(env, "hello world!");
    //调用静态方法使用的是jclass，而不是jobject
    (*env)->CallStaticVoidMethod(env, testclass, testStatic, str);

    jmethodID innerClassInstance = (*env)->GetMethodID(env, testclass, "getInnerClass",
                                                       "()Lcom/example/chuckapptestdemo/TestBeInvoked$TestInnerClass;");
    if (innerClassInstance == 0) {
        __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, "Communicate innerClassInstance == 0");
    }
    jobject innerObject = (*env)->CallObjectMethod(env, testobject, innerClassInstance);
    if (innerObject == 0) {
        __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, "Communicate innerObject == 0");
    }
    //实例化InnerClass子类
    jclass innerclass = (*env)->FindClass(env,
                                          "com/example/chuckapptestdemo/TestBeInvoked$TestInnerClass");
    //调用子类的成员方法
    jmethodID setInt = (*env)->GetMethodID(env, innerclass, "testInner", "(I)V");
    (*env)->CallVoidMethod(env, innerObject, setInt, 2);
}
```
上述调用对应的java层的代码如下：

```java
public class TestBeInvoked {
    final String TAG = "Communicate";

    public TestBeInvoked() {
        innerClass = new TestInnerClass();
        Log.d(TAG, "TestBeInvoked constructor is invoked");
    }

    public void test(int number) {
        Log.d(TAG, "test is invoked:" + number);
    }

    public static void testStatic(String message) {
        Log.d("Communicate", "testStatic is invoked:" + message);
    }

    private TestInnerClass innerClass;

    public TestInnerClass getInnerClass() {
        Log.d("Communicate", "getInnerClass is invoked");
        return innerClass;
    }

    public class TestInnerClass {

        public TestInnerClass() {
            Log.d(TAG, "TestInnerClass constructor is invoked");
        }

        public void testInner(int number) {
            Log.d(TAG, "testInner is invoked:" + number);
        }
    }
}
```

### 1.3 注册方式
> https://www.cnblogs.com/zl1991/p/6688914.html

#### 1.3.1 静态注册
静态注册时jni定义的方法名称要跟java下调用jni代码的类名路径对应，比如当前项目TestJniActivity中调用jni方法，路径是src/main/java/com/example/chuckapptestdemo/TestJniActivity，可以将cmd定位到java这一层，通过执行：

```javah -d jni com.example.chuckapptestdemo.TestJniActivity```

在java目录下生成`com_example_chuckapptestdemo_TestJniActivity.h`文件，如下：

```c
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class com_example_chuckapptestdemo_TestJniActivity */

#ifndef _Included_com_example_chuckapptestdemo_TestJniActivity
#define _Included_com_example_chuckapptestdemo_TestJniActivity
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_example_chuckapptestdemo_TestJniActivity
 * Method:    communicate2Jni
 * Signature: (Ljava/lang/String;)Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_example_chuckapptestdemo_TestJniActivity_communicate2Jni
  (JNIEnv *, jclass, jstring);

#ifdef __cplusplus
}
#endif
#endif

```

接下来在对应的c文件中include这个.h文件，并实现定义的方法就行了。


#### 1.3.2 动态注册
在上一小节中，可以看到定义的方法名称都是跟java包下的类路径对应的，这样会导致定义的方法名称比较长。其实可以通过动态注册的方式解决这个问题。之前有分析过dart底层的代码，都是通过动态注册方法。

要使用 `RegisterNatives`，请执行以下操作：

- 提供 `JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved)` 函数。
- 在 JNI_OnLoad 中，使用 RegisterNatives 注册所有原生方法。
- 使用 -fvisibility=hidden 进行构建，以便只从您的库中导出 JNI_OnLoad。这将生成速度更快且更小的代码，并避免与加载到应用中的其他库发生潜在冲突（但如果应用在原生代码中崩溃，则创建的堆栈轨迹的用处不大）。

看看动态注册的例子：

```c
#include "DynamicRegister.h"

static JNINativeMethod activity_method_table[] = {
        {"communicate2Jni", "(Ljava/lang/String;)Ljava/lang/String;", (void *) communicate2Jni},
};

JNIEXPORT jstring JNICALL communicate2Jni(JNIEnv *env, jobject obj, jstring message) {
    const char *nativeString = (*env)->GetStringUTFChars(env, message, 0);
    __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, "%s", nativeString);
    (*env)->ReleaseStringUTFChars(env, message, nativeString);
    jstring response = (*env)->NewStringUTF(env, "jni say hi to java");
    return response;
}

JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, "JNI_OnLoad");

    JNIEnv *env = NULL;
    jint result = -1;

    if ((*vm)->GetEnv(vm, (void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return -1;
    }

    if (!registerNatives(env)) //注册
    {
        return -1;
    }

    result = JNI_VERSION_1_6;

    return result;
}

static int registerNatives(JNIEnv *env) {
    if (!registerNativeMethods(env, JNIREG_ACTIVITY_CLASS, activity_method_table,
                               sizeof(activity_method_table) / sizeof(activity_method_table[0]))) {
        return JNI_FALSE;
    }

    return JNI_TRUE;
}

static int registerNativeMethods(JNIEnv *env, const char *className, JNINativeMethod *gMethods,
                                 int numMethods) {
    jclass clazz;
    clazz = (*env)->FindClass(env, className);
    if (clazz == NULL) {
        return JNI_FALSE;
    }

    if ((*env)->RegisterNatives(env, clazz, gMethods, numMethods) < 0) {
        return JNI_FALSE;
    }

    return JNI_TRUE;
}

JNIEXPORT void JNICALL JNI_OnUnload(JavaVM *vm, void *reserved) {
    __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, "JNI_OnUnload");
    JNIEnv *env;
    if ((*vm)->GetEnv(vm, (void **) &env, JNI_VERSION_1_6) != JNI_OK)
        __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, "JNI load GetEnv failed");
    else {
//        (*env)->DeleteGlobalRef(env, object);
//        (*env)->DeleteGlobalRef(env, object);
    }
}
```

动态注册方法时先将java层调用jni的方法名称与jni本地实际执行的方法对应起来，这里使用结构体`JNINativeMethod`，对应

> sdk\ndk-bundle\sysroot\usr\include\jni.h

```c
typedef struct {
    const char* name; //Java jni函数的名称
    const char* signature; //描述了函数的参数和返回值
    void* fnPtr; //函数指针，jni函数的对应的c函数
} JNINativeMethod;
```

上边两种注册方式对应在Activity中的代码是：

```java
public class TestJniActivity extends Activity {

    final String TAG = "Communicate";

    static {
        System.loadLibrary("communicate");
    }
    
        @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_child_recycler_view);

        String response = communicate2Jni("say hello to jni");
        Log.d(TAG, "response:" + response);
    }

    public static native String communicate2Jni(String message);
}
```

> 以下内容来自
> [https://www.cnblogs.com/zl1991/p/6688914.html](https://www.cnblogs.com/zl1991/p/6688914.html)

当程序在java层运行System.loadLibrary("communicate");这行代码后，程序会去载入libcommunicate.so文件，与此同时，产生一个"Load"事件，这个事件触发后，程序默认会在载入的.so文件的函数列表中查找JNI_OnLoad函数并执行，与"Load"事件相对，当载入的.so文件被卸载时，"Unload"事件被触发，此时，程序默认会去在载入的.so文件的函数列表中查找JNI_OnUnload函数并执行，然后卸载.so文件。需要注意的是，JNI_OnLoad与JNI_OnUnload这两个函数在.so组件中并不是强制要求的，用户也可以不去实现，java代码一样可以调用到C组件中的函数。

**动态注册方法的作用**

应用层的Java类别通过VM而调用到native函数。一般是通过VM去寻找*.so里的native函数。如果需要连续调用很多次，每次都需要寻找一遍，会多花许多时间。

此时，C组件开发者可以将本地函数向VM进行注册,以便能加快后续调用native函数的效率。可以这么想象一下，假设VM内部一个native函数链表，初始时是空的，在未显式注册之前此native函数链表是空的，每次java调用native函数之前会首先在此链表中查找需要查找需要调用的native函数，如果找到就直接使用，如果未找到，得再通过载入的.so文件中的函数列表中去查找，且每次java调用native函数都是进行这样的流程，因此，效率就自然会下降，为了克服这样现象，我们可以通过在.so文件载入初始化时，即JNI_OnLoad函数中，先行将native函数注册到VM的native函数链表中去，这样一来，后续每次java调用native函数时都会在VM中的native函数链表中找到对应的函数，从而加快速度.

**java与jni的基本类型**

很多文档都会有jni和java层的变量对应表，其实是来自于java的官方文档：[https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/types.html#wp9502](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/types.html#wp9502)

[https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html)

![jni变量继承图](https://ftp.bmp.ovh/imgs/2020/06/9458ba1effd18d9f.png)

jni层变量类型与java的变量类型对应表：

| Java Type	| Native Type| 	Description |
|:---:|:---:|:---:|
|boolean	|jboolean	|unsigned 8 bits|
|byte	|jbyte	|signed 8 bits|
|char	|jchar	|unsigned 16 bits|
|short	|jshort	|signed 16 bits|
|int	|jint	|signed 32 bits|
|long	|jlong	|signed 64 bits|
|float	|jfloat	|32 bits|
|double	|jdouble	|64 bits|
|void	|void	|not applicable|

变量签名对应的是：

| Type Signature | Java Type |
|:---:|:---:|
|Z|boolean|
|B|byte|
|C|char|
|S|short|
|I|int|
|J|long|
|F|float|
|D|double|
|L fully-qualified-class ;|fully-qualified-class|
|[ type|type[]|
|( arg-types ) ret-type|method type|

例如java定义的方法是:

```long f (int n, String s, int[] arr); ```

对应在jni层定义的签名对应的是:

```
(ILjava/lang/String;[I)J
```

括号里面对应的参数的签名，括号后面对应的是返回的数据类型。

### 1.4 jni的变量
> https://developer.android.com/training/articles/perf-jni
>
> https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/design.html

#### 1.4.1 JavaVM 和 JNIEnv

JNI 定义了两个关键数据结构，即“JavaVM”和“JNIEnv”。两者本质上都是指向函数表的二级指针。（在 C++ 版本中，它们是一些类，这些类具有指向函数表的指针，并具有每个通过该函数表间接调用的 JNI 函数的成员函数。）JavaVM 提供“调用接口”函数，您可以利用此类来函数创建和销毁 JavaVM。理论上，每个进程可以有多个 JavaVM，但 Android 只允许有一个。

![](https://ftp.bmp.ovh/imgs/2020/06/0053210760830b9b.png)

JNIEnv 提供了大部分 JNI 函数。您的原生函数都会收到 JNIEnv 作为第一个参数。

该 JNIEnv 将用于线程本地存储。因此，您无法在线程之间共享 JNIEnv。如果一段代码无法通过其他方法获取自己的 JNIEnv，您应该共享相应 JavaVM，然后使用 GetEnv 发现线程的 JNIEnv。

#### 1.4.2 局部引用和全局引用
传递给原生方法的每个参数，以及 JNI 函数返回的几乎每个对象都属于“局部引用”。这意味着，局部引用在当前线程中的当前原生方法运行期间有效。 在原生方法返回后，即使对象本身继续存在，该引用也无效。

这适用于 jobject 的所有子类，包括 jclass、jstring 和 jarray。（启用扩展的 JNI 检查时，运行时会针对大部分引用误用问题向您发出警告。）

获取非局部引用的唯一方法是通过 NewGlobalRef 和 NewWeakGlobalRef 函数。

如果您希望长时间保留某个引用，则必须使用“全局”引用。NewGlobalRef 函数将局部引用作为参数，然后返回全局引用。在调用 DeleteGlobalRef 之前，全局引用保证有效。

所有 JNI 方法都接受局部和全局引用作为参数。对同一对象的引用可能具有不同的值。例如，对同一对象连续调用 NewGlobalRef 所返回的值可能有所不同。 要了解两个引用是否引用同一对象，必须使用 IsSameObject 函数。 

程序员需要“不过度分配”局部引用。实际上，这意味着如果您要创建大量局部引用（也许是在运行对象数组时），应该使用 DeleteLocalRef 手动释放它们，而不是让 JNI 为您代劳。

该实现仅需要为 16 个局部引用保留槽位，因此如果您需要更多槽位，则应该按需删除，或使用 EnsureLocalCapacity/PushLocalFrame 保留更多槽位。

### 1.5 jni变量
> 以下内容引用自[https://blog.csdn.net/xyang81/article/details/44657385](https://blog.csdn.net/xyang81/article/details/44657385)
 
JNI定义了三种类型的变量，分别是：局部引用（Local Reference）、全局引用（Global Reference）、弱全局引用（Weak Global Reference）。

#### 1.5.1 局部引用
通过NewLocalRef和各种JNI接口创建（FindClass、NewObject、GetObjectClass和NewCharArray等），会阻止GC回收所引用的对象，不在本地函数中跨函数使用，不能跨线程使用。函数返回后局部引用所引用的对象会被JVM自动释放，或调用DeleteLocalRef释放。

具体示例如下：
```c
JNIEXPORT void JNICALL testObjectCreateAndRelease(JNIEnv *env, jobject obj) {
    jclass testclass = (*env)->FindClass(env, "com/example/chuckapptestdemo/TestBeInvoked");
    jmethodID testcontruct = (*env)->GetMethodID(env, testclass, "<init>", "()V");
    jobject testobject = (*env)->NewObject(env, testclass, testcontruct);
    jmethodID test = (*env)->GetMethodID(env, testclass, "test", "(I)V");
    (*env)->CallVoidMethod(env, testobject, test, 1);
    (*env)->DeleteLocalRef(env, testobject);
    (*env)->DeleteLocalRef(env, testclass);
}
```

局部引用也称本地引用，通常是在函数中创建并使用。会阻止GC回收所引用的对象。比如，调用NewObject接口创建一个新的对象实例并返回一个对这个对象的局部引用。局部引用只有在创建它的本地方法返回前有效，本地方法返回到Java层之后，如果Java层没有对返回的局部引用使用的话，局部引用就会被JVM自动释放。

##### 1.5.1.1 局部引用错误的使用方式
你可能会为了提高程序的性能，在函数中将局部引用存储在静态变量中缓存起来，供下次调用时使用。这种方式是错误的，因为函数返回后局部引很可能马上就会被释放掉，静态变量中存储的就是一个被释放后的内存地址，成了一个野针对，下次再使用的时候就会造成非法地址的访问，使程序崩溃。

错误的释放方法示例如下：
> Activity中调用

```java
public class TestJniActivity extends Activity {

    final String TAG = "Communicate";

    static {
        System.loadLibrary("communicate");
    }
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        String testString = testMistakeLocalRefCache();
        Log.d(TAG, "testString index 0:" + testString);
        testString = testMistakeLocalRefCache();
        Log.d(TAG, "testString index 1:" + testString);
    }
    
    public static native String testMistakeLocalRefCache();
}
```

jni中代码实现如下：

```c
JNIEXPORT jstring JNICALL testMistakeLocalRefCache(JNIEnv *env, jobject obj) {
    static jclass testclass;
    jstring j_str = NULL;
    if (testclass == NULL) {
        testclass = (*env)->FindClass(env, "com/example/chuckapptestdemo/TestBeInvoked");
        if (testclass == NULL) {
            return NULL;
        }
    }

    jmethodID testcontruct = (*env)->GetMethodID(env, testclass, "<init>", "()V");
    jobject testobject = (*env)->NewObject(env, testclass, testcontruct);

    jmethodID testReturnValue = (*env)->GetMethodID(env, testclass, "testReturnValue",
                                                    "(Ljava/lang/String;)V");
    j_str = (*env)->NewStringUTF(env, "testMistakeLocalRefCache");
    (*env)->CallVoidMethod(env, testobject, testReturnValue, j_str);

    (*env)->DeleteLocalRef(env, testobject);

    return j_str;
}
```
第一次调用完毕之后，`testclass`被释放；第二次再调用的时候，testclass本身不为空，但是指向为空，导致报错。

报错内容是：
```
JNI DETECTED ERROR IN APPLICATION: use of deleted local reference 0x91
```

##### 1.5.1.2 局部引用释放

释放一个局部引用有两种方式，一个是本地方法执行完毕后JVM自动释放，另外一个是自己调用DeleteLocalRef手动释放。既然JVM会在函数返回后会自动释放所有局部引用，为什么还需要手动释放呢？大部分情况下，我们在实现一个本地方法时不必担心局部引用的释放问题，函数被调用完成后，JVM 会自动释放函数中创建的所有局部引用。尽管如此，以下几种情况下，为了避免内存溢出，我们应该手动释放局部引用：
- 1、不需要的局部引用应立即删除

JNI会将创建的局部引用都存储在一个局部引用表中，如果这个表超过了最大容量限制，就会造成局部引用表溢出，使程序崩溃。经测试，Android上的JNI局部引用表最大数量是512个。当我们在实现一个本地方法时，可能需要创建大量的局部引用，如果没有及时释放，就有可能导致JNI局部引用表的溢出，所以，在不需要局部引用时就立即调用DeleteLocalRef手动删除。

尤其是在循环里面创建局部引用时，应当在循环内使用完毕之后立即删除。

- 2、工具类的函数中应当及时删除局部引用

在编写JNI工具函数时，工具函数在程序当中是公用的，被谁调用你是不知道的。上面newString这个函数演示了怎么样在工具函数中使用完局部引用后，调用DeleteLocalRef删除。不这样做的话，每次调用newString之后，都会遗留两个引用占用空间。

- 3、消息循环中应当及时删除局部引用

如果你的本地函数不会返回。比如一个接收消息的函数，里面有一个死循环，用于等待别人发送消息过来，如下：
```java
while(true) { 
    if (有新的消息) ｛ 
        处理之。。。。
    ｝ else { 
        等待新的消息。。。
    }
}
```
如果在消息循环当中创建的引用你不显示删除，很快将会造成JVM局部引用表溢出。

- 4、函数开始分配的局部引用应当及早释放

局部引用会阻止所引用的对象被GC回收。比如你写的一个本地函数中刚开始需要访问一个大对象，因此一开始就创建了一个对这个对象的引用，但在函数返回前会有一个大量的非常复杂的计算过程，而在这个计算过程当中是不需要前面创建的那个大对象的引用的。但是，在计算的过程当中，如果这个大对象的引用还没有被释放的话，会阻止GC回收这个对象，内存一直占用者，造成资源的浪费。所以这种情况下，在进行复杂计算之前就应该把引用给释放了，以免不必要的资源浪费。

##### 1.5.1.3 管理局部引用
JNI提供了一系列函数来管理局部引用的生命周期。这些函数包括：EnsureLocalCapacity、NewLocalRef、PushLocalFrame、PopLocalFrame、DeleteLocalRef。JNI规范指出，任何实现JNI规范的JVM，必须确保每个本地函数至少可以创建16个局部引用（可以理解为虚拟机默认支持创建16个局部引用）。

- EnsureLocalCapacity

可以通过调用EnsureLocalCapacity函数，确保在当前线程中创建指定数量的局部引用，如果创建成功则返回0，否则创建失败，并抛出OutOfMemoryError异常。

如下：
```c
JNIEXPORT jstring JNICALL testLocalRefCapacity(JNIEnv *env, jobject obj, jcharArray charArray) {
    LOGE("testLocalRefCapacity start");
    jsize size = (*env)->GetArrayLength(env, charArray);
    if (size == 0) {
        return (*env)->NewStringUTF(env, "size is error");
    }
    LOGE("testLocalRefCapacity size:%d", size);
    if ((*env)->EnsureLocalCapacity(env, size) != 0) {
        return (*env)->NewStringUTF(env, "memory size is invalid");
    }
    jchar *jcharPointer = (*env)->GetCharArrayElements(env, charArray, NULL);
    if (jcharPointer == NULL) {
        return (*env)->NewStringUTF(env, "jchar pointer is error");
    }
    int index = 0;
    for (; index < size; index++) {
        const jchar *jcharItem = (jcharPointer + index);
        LOGE("testLocalRefCapacity current char:%s",
             jcharItem);
        jstring str = (*env)->NewStringUTF(env, jcharItem);
        (*env)->DeleteLocalRef(env, str);
    }
    (*env)->ReleaseCharArrayElements(env, charArray, jcharPointer, 0);//mode,0 for copy back the content and free the elems buffer
    return (*env)->NewStringUTF(env, "testLocalRefCapacity success");
}
```

- Push/PopLocalFrame

除了EnsureLocalCapacity函数可以扩充指定容量的局部引用数量外，我们也可以利用Push/PopLocalFrame函数对创建作用范围层层嵌套的局部引用。例如，我们把上面那段处理字符串数组的代码用Push/PopLocalFrame函数对重写：

```c
JNIEXPORT jstring JNICALL testLocalRefFrame(JNIEnv *env, jobject obj, jcharArray charArray) {
    LOGE("testLocalRefFrame start");
    jsize size = (*env)->GetArrayLength(env, charArray);
    if (size == 0) {
        return (*env)->NewStringUTF(env, "size is error");
    }
    LOGE("testLocalRefFrame size:%d", size);
    if ((*env)->EnsureLocalCapacity(env, size) != 0) {
        return (*env)->NewStringUTF(env, "memory size is invalid");
    }
    jchar *jcharPointer = (*env)->GetCharArrayElements(env, charArray, NULL);
    if (jcharPointer == NULL) {
        return (*env)->NewStringUTF(env, "jchar pointer is error");
    }
    int index = 0;
    for (; index < size; index++) {
        if ((*env)->PushLocalFrame(env, N_REFS) != 0) {
            return (*env)->NewStringUTF(env, "ref size over flow");
        }
        const jchar *jcharItem = (jcharPointer + index);
        LOGE("testLocalRefFrame current char:%s",
             jcharItem);
        jstring str = (*env)->NewStringUTF(env, jcharItem);
        (*env)->PopLocalFrame(env, NULL);
    }
    (*env)->ReleaseCharArrayElements(env, charArray, jcharPointer,
                                     0);//mode,0 for copy back the content and free the elems buffer
    return (*env)->NewStringUTF(env, "testLocalRefFrame success");
}
```

PushLocalFrame为当前函数中需要用到的局部引用创建了一个引用堆栈，（如果之前调用PushLocalFrame已经创建了Frame，在当前的本地引用栈中仍然是有效的)每遍历一次调用(*env)->GetObjectArrayElement(env, arr, i);返回一个局部引用时，JVM会自动将该引用压入当前局部引用栈中。而PopLocalFrame负责销毁栈中所有的引用，这样一来，Push/PopLocalFrame函数对提供了对局部引用生命周期更方便的管理，而不需要时刻关注获取一个引用后，再调用DeleteLocalRef来释放引用。在上面的例子中，如果在处理jstr的过程当中又创建了局部引用，则PopLocalFrame执行时，这些局部引用将全都会被销毁。在调用PopLocalFrame销毁当前frame中的所有引用前，如果第二个参数result不为空，会由result生成一个新的局部引用，再把这个新生成的局部引用存储在上一个frame中。

局部引用不能跨线程使用，只在创建它的线程有效。不要试图在一个线程中创建局部引用并存储到全局引用中，然后在另外一个线程中使用。


#### 1.5.2 全局引用
调用NewGlobalRef基于局部引用创建，会阻GC回收所引用的对象。可以跨方法、跨线程使用。JVM不会自动释放，必须调用DeleteGlobalRef手动释放`(*env)->DeleteGlobalRef(env,g_cls_string);`。

```c
JNIEXPORT jstring JNICALL testGlobalRef(JNIEnv *env, jobject obj, jint number) {
    LOGE("testGlobalRef number %d", number);
    static jclass testClass;
    if (testClass == NULL) {
        LOGE("testGlobalRef first time testClass == NULL");
        jclass curTestClass = (*env)->FindClass(env, "com/example/chuckapptestdemo/TestBeInvoked");
        testClass = (*env)->NewGlobalRef(env, curTestClass);
        (*env)->DeleteLocalRef(env, curTestClass);
    } else {
        LOGE("testGlobalRef second time testClass != NULL");
    }

    jmethodID testConstruct = (*env)->GetMethodID(env, testClass, "<init>", "()V");
    jobject testObject = (*env)->NewObject(env, testClass, testConstruct);
    jmethodID test = (*env)->GetMethodID(env, testClass, "test", "(I)V");
    (*env)->CallVoidMethod(env, testObject, test, 1);
    return (*env)->NewStringUTF(env, "testGlobalRef success");
}
```
##### 1.5.2.1 管理全局引用

全局引用可以跨方法、跨线程使用，直到它被手动释放才会失效。同局部引用一样，也会阻止它所引用的对象被GC回收。与局部引用创建方式不同的是，只能通过NewGlobalRef函数创建。

当我们的本地代码不再需要一个全局引用时，应该马上调用DeleteGlobalRef来释放它。如果不手动调用这个函数，即使这个对象已经没用了，JVM也不会回收这个全局引用所指向的对象。


#### 1.5.3 弱全局引用
调用NewWeakGlobalRef基于局部引用或全局引用创建，不会阻止GC回收所引用的对象，可以跨方法、跨线程使用。引用不会自动释放，在JVM认为应该回收它的时候（比如内存紧张的时候）进行回收而被释放。或调用DeleteWeakGlobalRef手动释放。`(*env)->DeleteWeakGlobalRef(env,g_cls_string)`。

```c
JNIEXPORT jstring JNICALL testWeakGlobalRef(JNIEnv *env, jobject obj, jint number) {
    LOGE("testWeakGlobalRef number %d", number);
    static jclass weakTestClass;
    if (weakTestClass == NULL) {
        LOGE("testWeakGlobalRef first time testClass == NULL");
        jclass curTestClass = (*env)->FindClass(env, "com/example/chuckapptestdemo/TestBeInvoked");
        weakTestClass = (*env)->NewWeakGlobalRef(env, curTestClass);
        (*env)->DeleteLocalRef(env, curTestClass);
    } else {
        LOGE("testWeakGlobalRef second time testClass != NULL");
    }

    jmethodID testConstruct = (*env)->GetMethodID(env, weakTestClass, "<init>", "()V");
    jobject testObject = (*env)->NewObject(env, weakTestClass, testConstruct);
    jmethodID test = (*env)->GetMethodID(env, weakTestClass, "test", "(I)V");
    (*env)->CallVoidMethod(env, testObject, test, 1);
    return (*env)->NewStringUTF(env, "testWeakGlobalRef success");
}
```

##### 1.5.3.1 管理弱全局引用

弱全局引用使用NewGlobalWeakRef创建，使用DeleteGlobalWeakRef释放。下面简称弱引用。与全局引用类似，弱引用可以跨方法、线程使用。但与全局引用很重要不同的一点是，弱引用不会阻止GC回收它引用的对象。

当我们的本地代码不再需要一个弱全局引用时，也应该调用DeleteWeakGlobalRef来释放它，如果不手动调用这个函数来释放所指向的对象，JVM仍会回收弱引用所指向的对象，但弱引用本身在引用表中所占的内存永远也不会被回收。


#### 1.5.4 引用比较
给定两个引用（不管是全局、局部还是弱全局引用），我们只需要调用IsSameObject来判断它们两个是否指向相同的对象。例如：（*env)->IsSameObject(env, obj1, obj2)，如果obj1和obj2指向相同的对象，则返回JNI_TRUE（或者1），否则返回JNI_FALSE（或者0）。有一个特殊的引用需要注意：NULL，JNI中的NULL引用指向JVM中的null对象。如果obj是一个局部或全局引用，使用(*env)->IsSameObject(env, obj, NULL) 或者 obj == NULL 来判断obj是否指向一个null对象即可。

但需要注意的是，IsSameObject用于弱全局引用与NULL比较时，返回值的意义是不同于局部引用和全局引用的：

```c
JNIEXPORT jstring JNICALL testSameObject(JNIEnv *env, jobject obj, jint number) {
    jclass curTestClass = (*env)->FindClass(env, "com/example/chuckapptestdemo/TestBeInvoked");
    jmethodID testConstruct = (*env)->GetMethodID(env, curTestClass, "<init>", "()V");
    jobject testObject = (*env)->NewObject(env, curTestClass, testConstruct);

    jclass weakGlobalTestObject = (*env)->NewWeakGlobalRef(env, testObject);
    jclass globalTestObject = (*env)->NewGlobalRef(env, testObject);

    jboolean isGlobalObjectEqual = (*env)->IsSameObject(env, testObject, weakGlobalTestObject);
    jboolean isWeakGlobalObjectEqual = (*env)->IsSameObject(env, testObject, globalTestObject);

    LOGE("testSameObject isGlobalObjectEqual %d", isGlobalObjectEqual);
    LOGE("testSameObject isWeakGlobalObjectEqual %d", isWeakGlobalObjectEqual);

//    (*env)->DeleteWeakGlobalRef(env, weakGlobalTestObject);
    jboolean isWeakGlobalObjectNull = (*env)->IsSameObject(env, weakGlobalTestObject, NULL);
    LOGE("testSameObject isWeakGlobalObjectNull %d", isWeakGlobalObjectNull);
    return (*env)->NewStringUTF(env, "testSameObject success");
}
```

在上面的IsSameObject调用中，如果weakGlobalTestObject指向的引用testObject已经被回收，会返回JNI_TRUE，否则会返回JNI_FALSE。

### 1.6 jni崩溃排查
报错代码如下：
```c
JNIEXPORT void JNICALL testError(JNIEnv *env, jobject obj) {
    jclass curTestClass = (*env)->FindClass(env, "com/example/chuckapptestdemo/TestBeInvoked");
    jmethodID testConstruct = (*env)->GetMethodID(env, curTestClass, "<init>", "()V");
    jobject testObject = (*env)->NewObject(env, curTestClass, testConstruct);

    (*env)->DeleteLocalRef(env, testObject);

    jmethodID test = (*env)->GetMethodID(env, curTestClass, "test", "(I)V");
    (*env)->CallVoidMethod(env, testObject, test, 1);
}
```

报错信息如下：
```
2020-07-03 11:29:18.808 12279-12279/com.example.chuckapptestdemo A/zygote64: java_vm_ext.cc:534]   native: #09 pc 0000000000100a40  /system/lib64/libart.so (art::CheckJNI::CallVoidMethod(_JNIEnv*, _jobject*, _jmethodID*, ...)+156)
2020-07-03 11:29:18.808 12279-12279/com.example.chuckapptestdemo A/zygote64: java_vm_ext.cc:534]   native: #10 pc 0000000000001c18  /data/app/com.example.chuckapptestdemo-xS2K0ByIriN_rE4Udf-ufw==/lib/arm64/libcommunicate.so (testError+280)
2020-07-03 11:29:18.808 12279-12279/com.example.chuckapptestdemo A/zygote64: java_vm_ext.cc:534]   native: #11 pc 00000000000001e0  /data/app/com.example.chuckapptestdemo-xS2K0ByIriN_rE4Udf-ufw==/oat/arm64/base.odex (Java_com_example_chuckapptestdemo_TestJniActivity_testError__+144)
2020-07-03 11:29:18.808 12279-12279/com.example.chuckapptestdemo A/zygote64: java_vm_ext.cc:534]   at com.example.chuckapptestdemo.TestJniActivity.testError(Native method)
```

#### 1.6.1 addr2line
首先需要在sdk manager中安装ndk。

查看测试机器的CPU架构：`adb shell cat /proc/cpuinfo`，输出：

```
Processor       : AArch64 Processor rev 4 (aarch64)
processor       : 0
BogoMIPS        : 38.40
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x51
CPU architecture: 8
CPU variant     : 0xa
CPU part        : 0x801
CPU revision    : 4
```
AArch 64位机器。

ndk安装完成之后，根据设备CPU架构，找到addr2line在ndk目录下的地址。以当前测试机器的架构为例，对应的工具地址是：`D:\android-adt\sdk\ndk-bundle\toolchains\aarch64-linux-android-4.9\prebuilt\windows-x86_64\bin\aarch64-linux-android-addr2line.exe`。

找到本地生成的so库在项目下的路径，当前测试项目的路径是：`E:\android\workspace\Dev\ChuckAppTestDemo\app\build\intermediates\ndkBuild\debug\obj\local\arm64-v8a\libcommunicate.so`。

执行指令：
`arm-linux-androideabi-addr2line.exe -e E:\android\workspace\Dev\ChuckAppTestDemo\app\build\intermediates\ndkBuild\debug\obj\local\arm64-v8a\libcommunicate.so 0000000000100a40 0000000000001c18 00000000000001e0`

输出：
```
??
??:0
testError
E:/android/workspace/Dev/ChuckAppTestDemo/app/src/main/java/jni/DynamicRegister.c:186
??
??:0
```
对于这些??可以试着更新ndk版本解决。

另外如果不知道addr2line后跟的参数含义，可以执行`aarch64-linux-android-addr2line.exe -h`，输出如下：

```
Usage: aarch64-linux-android-addr2line.exe [option(s)] [addr(s)]
 Convert addresses into line number/file name pairs.
 If no addresses are specified on the command line, they will be read from stdin
 The options are:
  @<file>                Read options from <file>
  -a --addresses         Show addresses
  -b --target=<bfdname>  Set the binary file format
  -e --exe=<executable>  Set the input file name (default is a.out)
  -i --inlines           Unwind inlined functions
  -j --section=<name>    Read section-relative offsets instead of addresses
  -p --pretty-print      Make the output easier to read for humans
  -s --basenames         Strip directory names
  -f --functions         Show function names
  -C --demangle[=style]  Demangle function names
  -h --help              Display this information
  -v --version           Display the program's version

aarch64-linux-android-addr2line.exe: supported targets: elf64-littleaarch64 elf64-bigaarch64 elf32-littleaarch64 elf32-bigaarch64 elf32-littlearm elf32-bigarm elf64-little elf64-big elf32-little elf32-big plugin srec symbolsrec verilog tekhex binary ihex
Report bugs to <http://www.sourceware.org/bugzilla/>
```

#### 1.6.2 ndk-stack

> 以下内容参考：[https://developer.android.com/ndk/guides/ndk-stack](https://developer.android.com/ndk/guides/ndk-stack)

第一步，在下载的ndk目录下找到ndk-stack.cmd。

第二步，执行指令：`adb logcat | $NDK/ndk-stack -sym $PROJECT_PATH/obj/local/armeabi-v7a`

`|` 后面过滤的内容分别是：`$NDK/ndk-stack`表示ndk-stack所在的路径；-sym后边是项目生成的so地址，对应当前测试项目的执行指令是：
```adb logcat | ndk-stack.cmd -sym E:\android\workspace\Dev\ChuckAppTestDemo\app\build\intermediates\ndkBuild\debug\obj\local\arm64-v8a```。

第三步，运行程序，当运行==5.6.1==中的错误代码后，在cmd输出：
```
********** Crash dump: **********
Build fingerprint: '360/QK1807/QK1807:8.1.0/OPM1/8.1.109.PX.181221.360OS_360OS_QK1807_CN:user/release-keys'
pid: 15437, tid: 15437, name: huckapptestdemo  >>> com.example.chuckapptestdemo <<<
signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #00 pc 000000000001dec8  /system/lib64/libc.so (abort+104)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #01 pc 000000000047523c  /system/lib64/libart.so (art::Runtime::Abort(char const*)+552)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #02 pc 0000000000571a00  /system/lib64/libart.so (android::base::LogMessage::~LogMessage()+996)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #03 pc 00000000002ff090  /system/lib64/libart.so (art::JavaVMExt::JniAbort(char const*, char const*)+1700)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #04 pc 00000000002ff2e4  /system/lib64/libart.so (art::JavaVMExt::JniAbortF(char const*, char const*, ...)+180)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #05 pc 00000000004a1568  /system/lib64/libart.so (art::Thread::DecodeJObject(_jobject*) const+464)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #06 pc 000000000010f074  /system/lib64/libart.so (art::ScopedCheck::CheckInstance(art::ScopedObjectAccess&, art::ScopedCheck::InstanceKind, _jobject*, bool)+96)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #07 pc 000000000010dcd8  /system/lib64/libart.so (art::ScopedCheck::Check(art::ScopedObjectAccess&, bool, char const*, art::JniValueType*)+644)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #08 pc 0000000000113154  /system/lib64/libart.so (art::CheckJNI::CheckCallArgs(art::ScopedObjectAccess&, art::ScopedCheck&, _JNIEnv*, _jobject*, _jclass*, _jmethodID*, art::InvokeType, art::VarArgs const*)+132)
Stack frame 07-03 14:34:37.991 15478 15478 F DEBUG   :     #09 pc 000000000011200c  /system/lib64/libart.so (art::CheckJNI::CallMethodV(char const*, _JNIEnv*, _jobject*, _jclass*, _jmethodID*, std::__va_list, art::Primitive::Type, art::InvokeType)+664)
Stack frame 07-03 14:34:37.992 15478 15478 F DEBUG   :     #10 pc 0000000000100a40  /system/lib64/libart.so (art::CheckJNI::CallVoidMethod(_JNIEnv*, _jobject*, _jmethodID*, ...)+156)
Stack frame 07-03 14:34:37.992 15478 15478 F DEBUG   :     #11 pc 0000000000001c18  /data/app/com.example.chuckapptestdemo-DcWQH_KnoZj9ObTNecX4-w==/lib/arm64/libcommunicate.so (testError+280): Routine testError at E:/android/workspace/Dev/ChuckAppTestDemo/app/src/main/java/jni/DynamicRegister.c:186
Stack frame 07-03 14:34:37.992 15478 15478 F DEBUG   :     #12 pc 000000000001d1e0  /data/app/com.example.chuckapptestdemo-DcWQH_KnoZj9ObTNecX4-w==/oat/arm64/base.odex (offset 0x1d000)
```
这里就可以定位到`DynamicRegister.c:186`。

也可以使用 -dump 选项将 logcat 指定为输入文件。例如：
`adb logcat > /tmp/foo.txt`

`$NDK/ndk-stack -sym $PROJECT_PATH/obj/local/armeabi-v7a -dump foo.txt`

对应当前测试项目的指令是：

运行存在报错的代码，将崩溃的日志输出到本地磁盘：
`adb logcat > D:\log\foo.txt`

接下来，运用`ndk-stack`解析foo.txt文件中的日志，该工具会在开始解析 logcat 输出时查找第一行星号。执行指令：
`ndk-stack.cmd -sym E:\android\workspace\Dev\ChuckAppTestDemo\app\build\intermediates\ndkBuild\debug\obj\local\arm64-v8a -dump D:\log\foo.txt`

输出如下：
```
********** Crash dump: **********
Build fingerprint: '360/QK1807/QK1807:8.1.0/OPM1/8.1.109.PX.181221.360OS_360OS_QK1807_CN:user/release-keys'
pid: 15692, tid: 15692, name: huckapptestdemo  >>> com.example.chuckapptestdemo <<<
signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #00 pc 000000000001dec8  /system/lib64/libc.so (abort+104)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #01 pc 000000000047523c  /system/lib64/libart.so (art::Runtime::Abort(char const*)+552)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #02 pc 0000000000571a00  /system/lib64/libart.so (android::base::LogMessage::~LogMessage()+996)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #03 pc 00000000002ff090  /system/lib64/libart.so (art::JavaVMExt::JniAbort(char const*, char const*)+1700)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #04 pc 00000000002ff2e4  /system/lib64/libart.so (art::JavaVMExt::JniAbortF(char const*, char const*, ...)+180)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #05 pc 00000000004a1568  /system/lib64/libart.so (art::Thread::DecodeJObject(_jobject*) const+464)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #06 pc 000000000010f074  /system/lib64/libart.so (art::ScopedCheck::CheckInstance(art::ScopedObjectAccess&, art::ScopedCheck::InstanceKind, _jobject*, bool)+96)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #07 pc 000000000010dcd8  /system/lib64/libart.so (art::ScopedCheck::Check(art::ScopedObjectAccess&, bool, char const*, art::JniValueType*)+644)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #08 pc 0000000000113154  /system/lib64/libart.so (art::CheckJNI::CheckCallArgs(art::ScopedObjectAccess&, art::ScopedCheck&, _JNIEnv*, _jobject*, _jclass*, _jmethodID*, art::InvokeType, art::VarArgs const*)+132)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #09 pc 000000000011200c  /system/lib64/libart.so (art::CheckJNI::CallMethodV(char const*, _JNIEnv*, _jobject*, _jclass*, _jmethodID*, std::__va_list, art::Primitive::Type, art::InvokeType)+664)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #10 pc 0000000000100a40  /system/lib64/libart.so (art::CheckJNI::CallVoidMethod(_JNIEnv*, _jobject*, _jmethodID*, ...)+156)
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #11 pc 0000000000001c18  /data/app/com.example.chuckapptestdemo-DcWQH_KnoZj9ObTNecX4-w==/lib/arm64/libcommunicate.so (testError+280): Routine testError at E:/android/workspace/Dev/ChuckAppTestDemo/app/src/main/java/jni/DynamicRegister.c:186
Stack frame 07-03 14:44:12.092 15729 15729 F DEBUG   :     #12 pc 000000000001d1e0  /data/app/com.example.chuckapptestdemo-DcWQH_KnoZj9ObTNecX4-w==/oat/arm64/base.odex (offset 0x1d000)
```

这样也可以定位到。

同样的，可以通过-h查看后面追加参数的含义：
```
D:\android-adt\sdk\ndk-bundle>ndk-stack.cmd -h
Usage: ndk-stack -sym PATH [-dump PATH]
Symbolizes the stack trace from an Android native crash.

  -sym PATH   sets the root directory for symbols
  -dump PATH  sets the file containing the crash dump (default stdin)

See <https://developer.android.com/ndk/guides/ndk-stack.html>.
```

#### 1.6.3 objdump

当前报错的堆栈是：
```
2020-07-03 14:44:11.990 15692-15692/com.example.chuckapptestdemo A/zygote64: java_vm_ext.cc:534]   native: #09 pc 0000000000100a40  /system/lib64/libart.so (art::CheckJNI::CallVoidMethod(_JNIEnv*, _jobject*, _jmethodID*, ...)+156)
2020-07-03 14:44:11.990 15692-15692/com.example.chuckapptestdemo A/zygote64: java_vm_ext.cc:534]   native: #10 pc 0000000000001c18  /data/app/com.example.chuckapptestdemo-DcWQH_KnoZj9ObTNecX4-w==/lib/arm64/libcommunicate.so (testError+280)
2020-07-03 14:44:11.990 15692-15692/com.example.chuckapptestdemo A/zygote64: java_vm_ext.cc:534]   native: #11 pc 00000000000001e0  /data/app/com.example.chuckapptestdemo-DcWQH_KnoZj9ObTNecX4-w==/oat/arm64/base.odex (Java_com_example_chuckapptestdemo_TestJniActivity_testError__+144)
2020-07-03 14:44:11.990 15692-15692/com.example.chuckapptestdemo A/zygote64: java_vm_ext.cc:534]   at com.example.chuckapptestdemo.TestJniActivity.testError(Native method)
```

与==5.6.1==步骤类似，根据机器CPU架构找到objdump所在PC的路径：
`D:\android-adt\sdk\ndk-bundle\toolchains\aarch64-linux-android-4.9\prebuilt\windows-x86_64\bin\aarch64-linux-android-objdump.exe`

接下里执行指令：
`aarch64-linux-android-objdump.exe -S -D E:\android\workspace\Dev\ChuckAppTestDemo\app\build\intermediates\ndkBuild\debug\obj\local\arm64-v8a\libcommunicate.so > D:\log\error.dump`

-D：后面跟so库在项目中的地址。

这个指令的主要任务是将jni中的方法转成内存地址的形式转存到本地磁盘文件中。

对应报错的内存地址是0000000000001c18，在输出的error.dump文件中找到内存地址对应的方法：
```
JNIEXPORT void JNICALL testError(JNIEnv *env, jobject obj) {
    (*env)->CallVoidMethod(env, testObject, test, 1);
    1bfc:	00074d19 	.word	0x00074d19
    1c00:	00b81900 	.word	0x00b81900
    1c04:	ad190000 	.word	0xad190000
    1c08:	19000000 	.word	0x19000000
    1c0c:	0000145d 	.word	0x0000145d
    1c10:	170d001c 	.word	0x170d001c
    1c14:	1800001c 	.word	0x1800001c
    1c18:	000014bb 	.word	0x000014bb
}
```
可以定位到testError的最后一行报错。还有其他的内存地址都可以在这个error.dump文件中找到。

### 1.7 jni崩溃catch

> https://github.com/xroche/coffeecatch
>
> https://stackoverflow.com/questions/33777370/android-jni-exception-handling
>
> https://stackoverflow.com/questions/8415032/catching-exceptions-thrown-from-native-code-running-on-android