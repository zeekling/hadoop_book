
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
### initNativePosix详解

initNativePosix的JNI在NativeIO.c里面，函数的定义如下，其中，`JNIEnv *env, jclass clazz`为JNI默认需要带的参数，`jboolean doThreadsafeWorkaround`是函数initNativePosix的入参。
```c
JNIEXPORT void JNICALL
Java_org_apache_hadoop_io_nativeio_NativeIO_initNative(
  JNIEnv *env, jclass clazz, jboolean doThreadsafeWorkaround) 
```

在initNativePosix里面核心函数有：
- nioe_init(env)：主要是初始化NativeIOException异常对象。
- fd_init(env);初始化java.io.FileDescriptor
- workaround_non_threadsafe_calls_init(env);初始化一个Object对象，用于实现加锁。


## 底层IO操作

NativeIO提供了很多底层IO操作的JNI。主要包括：

| 函数名称 | 作用 |
|---|---|
| getPmdkLibPath()                                            | 获取HADOOP_PMDK_LIBRARY的路径 |
| isPmemCheck(long address, long length)                      | 用于判断一段内存区域是否位于真正的持久内存上  |
| pmemMapFile(String path, long length, boolean isFileExist); | 是将持久内存（Persistent Memory，PMEM）上的文件映射到进程的虚拟地址空间,调用库函数pmem_map_file |
| pmemUnMap(long address, long length)                        | pmem_unmap是持久内存编程中的一个关键函数，它就像一位负责收尾的清道夫，安全地解除之前建立的内存映射关系，并确保数据的持久化。  |

其他实现可自行查看NativeIO.c，基本都是对操作系统函数的封装，不再重复列出用途。
