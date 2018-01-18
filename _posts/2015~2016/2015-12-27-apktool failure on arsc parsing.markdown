---
layout: post
title: 解决修改arsc同名文件导致apktool反编译失败
categories: Reverse_Engineering
description: 解决修改arsc同名文件导致apktool反编译失败
keywords: 
---

# 解决修改arsc同名文件导致apktool反编译失败

## 背景

&emsp;&emsp;有些软件，会修改apk的resources.arsc文件，做成同名资源，导致apktool出现崩溃：

```Txt
Exception in thread "main" brut.androlib.AndrolibException: Multiple res specs: drawable/
  at brut.androlib.res.data.ResTypeSpec.addResSpec(ResTypeSpec.java:78)
  at brut.androlib.res.decoder.ARSCDecoder.readEntry(ARSCDecoder.java:248)
  at brut.androlib.res.decoder.ARSCDecoder.readTableType(ARSCDecoder.java:212)
  at brut.androlib.res.decoder.ARSCDecoder.readTableTypeSpec(ARSCDecoder.java:154)
  at brut.androlib.res.decoder.ARSCDecoder.readTablePackage(ARSCDecoder.java:116)
  at brut.androlib.res.decoder.ARSCDecoder.readTableHeader(ARSCDecoder.java:78)
  at brut.androlib.res.decoder.ARSCDecoder.decode(ARSCDecoder.java:47)
  at brut.androlib.res.AndrolibResources.getResPackagesFromApk(AndrolibResources.java:544)
  at brut.androlib.res.AndrolibResources.loadMainPkg(AndrolibResources.java:63)
  at brut.androlib.res.AndrolibResources.getResTable(AndrolibResources.java:55)
  at brut.androlib.Androlib.getResTable(Androlib.java:66)
  at brut.androlib.ApkDecoder.setTargetSdkVersion(ApkDecoder.java:198)
  at brut.androlib.ApkDecoder.decode(ApkDecoder.java:96)
  at brut.apktool.Main.cmdDecode(Main.java:165)
  at brut.apktool.Main.main(Main.java:81)
```

## 代码

&emsp;&emsp;为了解决这个问题，就要修改apktool，编译过程如上一篇帖子所述http://www.0xaa55.com/thread-1659-1-1.html，发生异常后，程序断在：

```Java
    public void addResSpec(ResResSpec spec) throws AndrolibException {
        if (mResSpecs.put(spec.getName(), spec) != true) {
            throw new AndrolibException(String.format(
                    "Multiple res specs: %s/%s", getName(), spec.getName()));
        }
    }
```

再看ApkTool源码

```Java
//ResType.java
package brut.androlib.res.data;

import brut.androlib.AndrolibException;
import brut.androlib.err.UndefinedResObject;
import java.util.*;

/**
 * @author Ryszard Wiśniewski <[email]brut.alll@gmail.com[/email]>
 */
public final class ResType {
    private final String mName;
    private final Map<String, ResResSpec> mResSpecs = new LinkedHashMap<String, ResResSpec>();

    private final ResTable mResTable;
    private final ResPackage mPackage;

    public ResType(String name, ResTable resTable, ResPackage package_) {
        this.mName = name;
        this.mResTable = resTable;
        this.mPackage = package_;
    }

    public String getName() {
        return mName;
    }

    public boolean isString() {
        return mName.equalsIgnoreCase("string");
    }

    public Set<ResResSpec> listResSpecs() {
        return new LinkedHashSet<ResResSpec>(mResSpecs.values());
    }

    public ResResSpec getResSpec(String name) throws AndrolibException {
        ResResSpec spec = mResSpecs.get(name);
        if (spec == null) {
            throw new UndefinedResObject(String.format("resource spec: %s/%s",
                    getName(), name));
        }
        return spec;
    }

    public void addResSpec(ResResSpec spec) throws AndrolibException {
        if (mResSpecs.put(spec.getName(), spec) != null) {
            throw new AndrolibException(String.format(
                    "Multiple res specs: %s/%s", getName(), spec.getName()));
        }
    }

    @Override
    public String toString() {
        return mName;
    }
```

&emsp;&emsp;可以发现mResSpecs是map结构的，键值一一对应，所以对方如果在arsc中放了2个同名资源的话，apktool不会继续处理，那么比较好的方法是将mResSpecs做成multimap结构，因此改动如下：

```Java
package brut.androlib.res.data;
import com.google.common.collect.HashMultimap;
import brut.androlib.AndrolibException;
import brut.androlib.err.UndefinedResObject;
import java.util.*;

/**
 * @author Ryszard Wiśniewski <[email]brut.alll@gmail.com[/email]>
 */
public final class ResType {
    private final String mName;
    //private final Map<String, ResResSpec> mResSpecs = new LinkedHashMap<String, ResResSpec>();
    private static HashMultimap<String,ResResSpec> mResSpecs = null;
    private final ResTable mResTable;
    private final ResPackage mPackage;

    public ResType(String name, ResTable resTable, ResPackage package_) {
        this.mName = name;
        this.mResTable = resTable;
        this.mPackage = package_;
        mResSpecs = HashMultimap.create();
    }

    public String getName() {
        return mName;
    }

    public boolean isString() {
        return mName.equalsIgnoreCase("string");
    }

    public Set<ResResSpec> listResSpecs() {
        return new LinkedHashSet<ResResSpec>(mResSpecs.values());
    }

    public ResResSpec getResSpec(String name) throws AndrolibException {
        //ResResSpec spec = mResSpecs.get(name);
        Set<ResResSpec> spec = mResSpecs.get(name);
        if (spec.size() == 0) {
            throw new UndefinedResObject(String.format("resource spec: %s/%s",
                    getName(), name));
        }
        //return spec;
        return spec.iterator().next();
    }

    public void addResSpec(ResResSpec spec) throws AndrolibException {
        //if (mResSpecs.put(spec.getName(), spec) != null) {
        if (mResSpecs.put(spec.getName(), spec) != true) {
            throw new AndrolibException(String.format(
                    "Multiple res specs: %s/%s", getName(), spec.getName()));
        }
    }

    @Override
    public String toString() {
        return mName;
    }
}
```

## 结论
&emsp;&emsp;在android studio中调试成功解密出文件之后替换 安卓改之理 中的apktool，完美恢复！
