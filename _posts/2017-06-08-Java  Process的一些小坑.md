---
layout:     post
title:      "Java  Process的一些小坑"
date:       2017-06-08
author:     "示水"
catalog:    true
tags:
    - java
---

# Java  Process的一些小坑

## 背景

在图片或者视频处理， 或者其他需求，需要调用其他工具做些操作或者处理（如shell）等， 用Java的Process来调用其他进程处理， 当使用`process.destroyForcibly()`时， 会有一些坑在， 父进程被kill， 子进程未被kill， 造成机器过载， 影响服务。

## 实例分析

Process.destroyForcibly代码片段

```java
//UNIXProcess
@Override
public Process destroyForcibly() {
    destroy(true);
    return this;
}
```

对应底层native代码

```c
JNIEXPORT void JNICALL
Java_java_lang_UNIXProcess_destroyProcess(JNIEnv *env,
                                          jobject junk,
                                          jint pid,
                                          jboolean force)
{
    int sig = (force == JNI_TRUE) ? SIGKILL : SIGTERM;
    kill(pid, sig);
}
```

从底层C代码，可以看到， 当使用force时， 发送的中`SIGKILL`也就是`-9`强杀， 这个信号是不能被捕捉的（内核的安全机制原因）。`SIGTERM`也是终止进程的信号， 但可以被阻塞、处理和忽略。

[SIGKILL vs SIGTERM参照这遍文章，很详细]( [SIGTERM vs. SIGKILL - major.io](https://major.io/2010/03/18/sigterm-vs-sigkill/) )

## 解决

在process.destroyForcibly()之前做下cleanup工作。 

cleanup的工作是拿到父进程， 再kill所有的子进程

```java
//kill sub processes
private void cleanup(Process p) {
    int pid = ProcessUtils.getPid(p);
    if (pid != -1) {
        try {
            Process cleanUpProcess = new ProcessBuilder("/usr/bin/pkill", "-9", "-P", String.valueOf(pid)).start();
            int exitcode = cleanUpProcess.waitFor();
            ProcessLogger.warn("CLEANUP", String.format("pid: %d exitcode: %d", pid, exitcode));
        } catch (Exception e) {
            ProcessLogger.warn("CLEANUP pid: " + pid, e);
        }
    } else {
        ProcessLogger.warn("CLEANUP", "get pid error");
    }
 }
```

**说明：**获取`pid`的方式， 在JDK9之前只能通过反射机制， 在JDK9可以直接通过接口拿到。

## 题外话

如果是直接通过脚本处理， 可以通过`trap`+`pkill`， 捕捉信息做清理工作

```bash
cleanup() {
        #do clean up
        >&2 echo "interrupted!"
        exit 2
}

trap cleanup SIGHUP SIGINT SIGQUIT SIGTERM
```





