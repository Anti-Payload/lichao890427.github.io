---
layout: post
title: no symbol directories found
categories: Android
description: no symbol directories found错误解决
keywords: 
---

# 解决NDK无法调试的Bug

&emsp;&emsp;我出现问题的版本是Android Studio2.2.3，之前项目是正常的，可以调试JNI代码，但是突然有一次不知道什么原因就无法调试，断点无法断下，调试时有这样的警告：

```
Now Launching Native Debug Session
Attention! No symbol directories found - please check your native debug configuration</font>

Starting LLDB server: /data/local/tmp/lldb/bin/start_lldb_server.sh /data/local/tmp/lldb unix-abstract /data/local/tmp/lldb/tmp platform-1504578240179.sock "lldb process:gdb-remote packets"
```'

&emsp;&emsp;为了研究这个问题，我们需要用到beyond compare，先新建一个工程，把原工程实际代码和资源拷贝进去，包括manifest，cmakelist等文件。这样重新编译调试我们会发现是可以进行Jni调试的。然后我们对这两个工程做二进制对比，发现app.iml中

```Xml
    <facet type="native-android-gradle" name="Native-Android-Gradle">
      <configuration>
        <option name="SELECTED_BUILD_VARIANT" value="release" />
      </configuration>
    </facet>
```

&emsp;&emsp;注意这个value，改成debug，便一切正常！