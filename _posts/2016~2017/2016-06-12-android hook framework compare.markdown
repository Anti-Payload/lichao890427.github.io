---
layout: post
title: Android Hook框架总结
categories: Android
description: android hook框架总结
keywords: Fiddler
---

# Android Hook框架总结

## Java正常方式调用逻辑：
* dvmCallVoidMethod
* dvmCallMethod
* dvmCallMethodV
* dvmInterpret
* dvmMterpStd
* dvmMterpStdRun
* opcode => HANDLE_OPCODE(OP_INVOKE_STATIC)
* GOTO_invoke(invokeStatic)
* GOTO_invokeMethod
* dvmMterp_invokeMethod
* GOTO_TARGET(invokeMethod)
* Pc=methodToCall->insns/methodToCall->nativeFunc

## Java反射方式调用逻辑：
```Txt
Constructor getDeclaredConstructor = clazz. getDeclaredConstructor()
Method m = clazz.getDeclaredMethod()
m.Invoke() =>
Constructor. newInstance =>
Dalvik_java_lang_reflect_Constructor_constructNative =>
dvmInvokeMethod =>
method->nativeFunc/dvmInterpret insns
Method.invoke =>
Method.invokeNative =>
Dalvik_java_lang_reflect_Method_invokeNative =>
dvmInvokeMethod =>
method->nativeFunc/dvmInterpret
nativeFunc => dvmResolveNativeMethod
dfunc = dvmLookupInternalNativeMethod
dfunc()
GOTO_TARGET(invokeMethod, bool methodCallRange, const Method* _methodToCall, u2 count, u2 regs)
```

## Android Hook方式：
* 1.静态修改apk中的函数调用，插入语句(droidbox)
* 2.用反射获取java函数Method类型对应c层结构Method*，修改insns域的dex字节码(未发现)
* 3.修改method->nativeFunc域，自己实现dvmResolveNativeMethod以重新映射(xposed/substrate)

## Hook类型
* IXposedHookZygoteInit 在Zygote进程启动初始化时执行hook，framework/system
* IXposedHookLoadPackage 在包加载时刻hook，此时可以hook用户app的函数
* IXposedHookInitPackageResources 在资源操作时刻回调
* IXposedHookCmdInit 在执行am,pm等时回调

## Xposed文件结构
* app_process
* Xposed_service.cpp 提供文件操作接口(libc.so尚未加载)
* Libxposed_dalvik.cpp 提供dalvik虚拟机hook接口(jni层实现)
* Libxposed_art.cpp 提供art虚拟机hook接口(jni层实现)
* Libxposed_command.cpp 提供其他通用接口
* Xposed.cpp C++层一些实现，用于app_main
* App_main.cpp app_process入口，用于替换app_process(重写main和AppRuntime)  api>=21
* App_main2.cpp app_process入口，用于替换app_process(重写main和AppRuntime)  api<21
* Xposedbridge => xposedbridge.jar
* XposedHelpers.java 提供反射能力
* XposedBridge.java xposedbridge入口，代替系统ZygoteInit类，主要的hook逻辑
* XC_MethodHook.java 为插件提供hook接口(java层实现)

## 问题1：何时加载framework.jar?
```Java
ActivityThread.class => const-class ActivityThread => OP_CONST_CLASS.cpp
dvmResolveClass 
    dvmFindClassNoInit
        findClassFromLoaderNoInit
            dvmCallMethod(loadClass)
        dvmFindSystemClassNoInit
            findClassNoInit
            searchBootPathForClass
```

## 问题2：为何IXposedHookZygoteInit可以hook framework/system api
```Java
XposedBridge.main
    initForZygote
         findAndHookMethod 加载系统jar
    loadModules
        loadModule        
            moduleInstance.initZygote()
            hookLoadPackage(moduleInstance)
```

## 问题3：为何IXposedHookLoadPackage可以hook app自身函数
首次加载：
```Java
android.app.ActivityThread.bindApplication
handleBindApplication
    ActivityStack.realStartActivityLocked
        scheduleLaunchActivity
            getPackageInfoNoCheck
getPackageInfoNoCheck
    LoadedApk()构造
        LoadedApk.getClassLoader
            ApplicationLoaders.getDefault().getClassLoader(zip,libpath,null)
```

## 问题4：何时加载xposedbridge?
```Java
Xposed app_main->main():
    xposed::initialize()        ->  env中加入xposedbridge.jar
    runtime.start(“de.robv.android.xposed.XposedBridge”) -> AndroidRuntime
         startVm()
         onVmCreated()
             xposed:nVmCreated        加载xposedbridge.jar
         startReg()
         XposedBridge.main()
```
				
## 问题5：底层实现hook？
&emsp;&emsp;强制吧函数设置为native，并修改native函数使其返调java函数，实现于libxposed_dalvik.cpp
```Java
hookInfo->reflectedMethod = dvmDecodeIndirectRef(dvmThreadSelf(), env->NewGlobalRef(reflectedMethodIndirect));
hookInfo->additionalInfo = dvmDecodeIndirectRef(dvmThreadSelf(), env->NewGlobalRef(additionalInfoIndirect));
SET_METHOD_FLAG(method, ACC_NATIVE);//设置Method->AccessFlag强制为native函数
method->nativeFunc = &hookedMethodCallback;//修改默认回调dvmResolveNativeMethod为自定义函数，该函数原先从系统函数和so中的jni函数中寻找java对应的c层方法，hookedMethodCallback函数则调用dvmCallMethod执行java层方法
method->insns = (const u2*) hookInfo;//该域原用于非native模式下保存dex字节码用于解释执行，现用于存储Method指针
method->registersSize = method->insSize;
method->outsSize = 0;
```

## Xposed Hook框架特点
* 1.兼容性方面，支持selinux，对某些厂商做适配
* 2.提供多种hook功能，可以hook系统api，应用app函数，资源操作，am pm等调用，插件扩展方便
* 3.开源，不断更新，支持art虚拟机，android5.x/6.x

## Cydia Substrate Hook框架特点
* 1.hook时机早，因此支持hook so函数
* 2.重定向liblog.so，较易适配
* 3.支持ios、android平台
* 4.不支持dalvik，不支持selinux，不支持api>=5.0

文件：
```Txt
substrate.h             //c++ header file used in JNI layer hook 
substrate-api.jar       //import package used in java layer hook
substrate-bless.jar     //used to remove properties(private,protect,etc...) in java layer hook
com.saurik.substrate.apk//host apk, we can only develop plugin for it to install package
\lib\armeabi  \lib\x86  //real operation for hooking
libAndroidBootstrap0.so //used to fake /system/lib/liblog.so and pull up libAndroidLoader.so    
libAndroidLoader.so     //used to pull all *.cy.so
        //MSLoadExtensions
libAndroidCydia.cy.so   //still in research
libDalvikLoader.cy.so   //still in research
libsubstrate.so         //provide jni layer hook low-level api
        //MSFindSymbol MSGetImageByName MSCloseFunction MSDebug MSHookFunction
libsubstrate-dvm.so     //provide java layer hook low-level api
        //MSDecodeIndirectReference MSJavaHookClassLoad MSJavaHookBridge MSJavaHookMethod 
        // MSJavaCreateObjectKey MSJavaReleaseObjectKey MSJavaGetObjectKey MSJavaSetObjectKey MSJavaBlessClassLoader
libSubstrateJNI.so      //used by substrate.apk to do c++ layer work
        //getppid readlink grep unlink symlink mkdir kill chown chmod
libSubstrateRun.so      //used by substrate.apk to do patch/unpatch/link/unlink operation
        //patch unpatch link unlink nm rpl
update-binary.so        //used by substrate.apk to recover patch/link operation
```

## Hook框架对比

|substrate			 |框架	   					|xposed框架						|dexposed框架         |
|--------------------|--------------------------|-------------------------------|---------------------|
|dalvik/art虚拟机支持|dalvik					|dalvik/art						|dalvik/art           |
|android版本支持	 |2.x 3.x 4.x				|2.x 3.x 4.x 5.x 6.x			|2.x 3.x 4.x 5.x 6.x  |
|hook能力			 |java/c api				|java api						|java/c api(自身模块) |
|修改文件			 |app_process				|liblog.so						|自身文件             |
|hook时机			 |app_process启动时			|app_process启动前加载so的时刻	|未知                 |
|hook方式			 |修改method结构体			|重新映射java层对应的jni函数	|修改method结构体     |
|c层hook类型		 |无						|inline							|未知                 |
|是否需要root		 |是						|是								|否                   |
|使用形式			 |宿主+插件					|宿主+插件						|未知                 |
|风险				 |app_process随每个版本变化 |加载so较多						|未知                 |
|ABI				 |x86/arm		 			|x86/arm						|未知                 |
|操作系统			 |ios/android				|android						|                     |
