---
layout: post
title: 在Mac&iOS App中使用ollvm
categories: Reverse_Engineering
description: 在Mac&iOS App中使用ollvm
keywords: 
---

# 在Mac&iOS App中使用ollvm

Ollvm，是C/C++的跨平台混淆方式，它提供了指令平坦化的能力，可以吧逻辑变得相当复杂从而阻止逆向工程。网上有很多关于在Android平台使用ollvm的方法，而Windows/iOS/Mac/Linux上则很少介绍，下面对于其他系统简略介绍操作方法，重点介绍Mac/iOS APP的混淆方法

* 指令替换 -mllvm -sub
* 控制流伪造 -mllvm -bcf
* 控制流平坦化 -mllvm -fla

默认会变换所有函数，加密单个函数：
```
int foo() __attribute((__annotate__(("fla"))));
int foo() {
   return 2;
}
```

## 获得OLLVM

目前有以下版本在git上放出：

(1) 作者放出的原版OLLVM  
包括llvm-3.3, llvm-3.4, llvm-3.5, llvm-3.6.1这几个版本，下载编译：
```
$ git clone -b llvm-4.0 https://github.com/obfuscator-llvm/obfuscator.git
$ mkdir build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Release ../obfuscator/
$ make
```
(2) 上海交大改版OLLVM  
增加了字符串编译时混淆-mllvm -sobf，对应llvm4.0版本，下载编译：
```
$ git clone https://github.com/GoSSIP-SJTU/Armariris.git
$ mkdir build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Release ../obfuscator/
$ make
```

(3) 第三方改版OLLVM  
好事者在上海交大的基础上，增加了llvm5.0 llvm6.0版本，下载编译方式同上  
ollvm6.0 https://github.com/Qrilee/Obfuscator-LLVM.git   
ollvm5.0 https://github.com/Qrilee/llvm-obfuscator.git  

## 编译OLLVM

注意，LLVM支持Windows/iOS/Mac/Linux/Android等全平台，OLLVM同理。只要你的工程使用LLVM-Clang进行编译，自然可以用OLLVM做混淆。如果只支持GCC编译则需要做调整  

上述编译方式是普通编译，在编译后得到可执行文件Clang就是用作混淆的，用它来替换编译使用的clang程序即可，clang路径可以从编译时命令行参数得到。如果使用make/cmake的工程，可以通过设置某些选项设置clang编译器。然后通过暴力替换clang达到同样的效果

* 直接编译
path_to_the/build/bin/clang test.c -o test -mllvm -sub

* make/cmake编译
```
CC=path_to_the/build/bin/clang
CFLAGS+="-mllvm -fla"     或     CXXFLAGS+="-mllvm -fla" 
./configure
make
```

## android ndk使用ollvm
首先要设置编译器为clang  
* * ndk-build方式，修改android.mk
NDK_TOOLCHAIN_VERSION := clang
* * cmake方式，修改build.gradle
```
android {
    ...
    defaultConfig {
        ...
        externalNativeBuild {
            cmake {
                arguments '-DANDROID_PLATFORM=android-9',
                          '-DANDROID_TOOLCHAIN=clang'
                // explicitly build libs
                targets 'gmath', 'gperf'
            }
        }
    }
    ...
}
```

cmake下替换clang程序，在CMakeLists.txt中修改 `set(LOCAL_CFLAGS " -mllvm -fla")`  
ndkbuild下替换clang程序，在Android.mk中加入参数`LOCAL_CFLAGS := -mllvm -fla`  

## visual studio使用ollvm

编译ollvm的时候，使用cmake-gui选择visual studio2015或者命令行选择```cmake -G "Visual Studio 14 2015" -DCMAKE_BUILD_TYPE=Release ../obfuscator/```  
然后cmake会产生一个visual studio工程，用vs编译即可！  
至于将Visual Studio的默认编译器换成clang编译，参考https://www.ishani.org/projects/ClangVSX/

Visual Studio2015起官方开始支持Clang，具体做法：  
新建项目->已安装->Visual C++->跨平台->安装Clang with Microsoft CodeGen  
Clang是一个完全不同的命令行工具链，这时候可以在工程配置中，平台工具集选项里找到Clang，然后使用ollvm的clang替换该clang即可

## xcode编译(用于编译OSX/iOS app)

编译方式比较特殊，由于xcode本身比较特殊，需要编译一套工具链给xcode

```
cmake -G Ninja -DLLVM_CREATE_XCODE_TOOLCHAIN=On -DCMAKE_INSTALL_PREFIX=$PWD/install
ninja install-xcode-toolchain
```

最后得到一个后缀含toolchain的大文件（如LLVM6.0.0.git.xctoolchain），大概8G，将文件mv到/Application/Xcode.app/Contents/Developer/Toolchains/下。重启Xcode将会在菜单->Xcode出现Toolchains菜单项。选择里面的llvm即可切换到新的工具链。如果直接编译，会产生`error:cannot specify -o when generating multiple output files`错误，这时候通过逐步排查可以得知将`Build Settings->Build Options->Enable  Index-While-Building Functionality`设置为No即可成功编译

