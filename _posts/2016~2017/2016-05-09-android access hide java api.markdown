---
layout: post
title: Java如何访问隐藏API
categories: Android
description: java如何访问隐藏api
keywords: 
---

# Java如何访问隐藏API

## 背景

&emsp;&emsp;Android编程，会遇到这样的问题：某函数未在android-xx.jar导出，即为隐藏api，但是运行时是实际有该函数的，导致我们无法直接调用。解决方式也有2个，一个是通过反射，下面提供第二种简单快速的方法：

## 例子

&emsp;&emsp;要调用android.os.SystemProperties这个类的get函数，可以这样做

### Eclipse建立工程
```Java
package android.os;
public class SystemProperties {
    public static String get(String var0) {
        return null;
    }
}
```

### 导出为jar包，添加到目标工程
&emsp;&emsp;在工程属性Dependencies中设置为Provided(这样就会把该jar当做引用而不把他打包进去！)，编译运行即可。
