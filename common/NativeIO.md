
# 简介

NativeIO主要用于实现一些Java未实现的IO相关的接口。通过JNI的的方式直接调用底层操作系统的系统函数，提升效率和性能。

# 源码详解

主要分下面几部分：
- JNI初始化，包括底层JNI代码。

## 初始化
核心初始化的代码是在NativeIO里面的静态代码块里面实现的，通过参数hadoop.workaround.non.threadsafe.getpwuid控制是否支持线程安全，默认是线程安全的。
初始化只会做一次，不会重复初始化，关键代码如下：
```java
static {
      if (NativeCodeLoader.isNativeCodeLoaded()) {
        //确保只加初始化一次。
        try {
          Configuration conf = new Configuration();
          boolean workaroundNonThreadSafePasswdCalls = conf.getBoolean(
              WORKAROUND_NON_THREADSAFE_CALLS_KEY,
              WORKAROUND_NON_THREADSAFE_CALLS_DEFAULT);

          initNativePosix(workaroundNonThreadSafePasswdCalls);
          nativeLoaded = true;
          // 省略。。。。
        } catch (Throwable t) {
         // 省略。。。。
        }
      }
    }
```

