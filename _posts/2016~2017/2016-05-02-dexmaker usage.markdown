---
layout: post
title: 动态生成dalvik字节码——dexmaker
categories: Android
description: 动态生成dalvik字节码——dexmaker
keywords: 
---

## 介绍

&emsp;&emsp;<https://android.googlesource.com/platform/external/dexmaker.git>
该开源项目可以通过编程的方式，把java语义直接转换成android上dalvik字节码，正常的java代码编译生成class，而要转换成dalvik字节码才能给android使用，这里介绍的黑科技就是比android上直接加载dex更为底层——直接在android上编译出dalvik字节码再加载

## 样例
&emsp;&emsp;想生成如下java代码的dalvik字节码：

```Java
package com.google.dexmaker.examples.Fibonacci;
public class Fibonacci {
    public static int fib(int i) {
        if (i < 2) {
            return i;
        }
        return fib(i - 1) + fib(i - 2);
    }
}
```

那么我需要写一个对应的逻辑去在android中生成上述逻辑，下面的代码我称为预编译代码

```Java
// 对应package com.google.dexmaker.examples.Fibonacci;
       TypeId<?> fibonacci = TypeId.get("Lcom/google/dexmaker/examples/Fibonacci;");

       String fileName = "Fibonacci.generated";
       DexMaker dexMaker = new DexMaker();
//对应 public class Fibonacci
       dexMaker.declare(fibonacci, fileName, Modifier.PUBLIC, TypeId.OBJECT);

//对应public static int fib(int i)
       MethodId<?, Integer> fib = fibonacci.getMethod(TypeId.INT, "fib", TypeId.INT);
       Code code = dexMaker.declare(fib, Modifier.PUBLIC | Modifier.STATIC);

//声明局部变量
       Local<Integer> i = code.getParameter(0, TypeId.INT);
       Local<Integer> constant1 = code.newLocal(TypeId.INT);
       Local<Integer> constant2 = code.newLocal(TypeId.INT);
       Local<Integer> a = code.newLocal(TypeId.INT);
       Local<Integer> b = code.newLocal(TypeId.INT);
       Local<Integer> c = code.newLocal(TypeId.INT);
       Local<Integer> d = code.newLocal(TypeId.INT);
       Local<Integer> result = code.newLocal(TypeId.INT);

       code.loadConstant(constant1, 1);//=1
       code.loadConstant(constant2, 2);//=2
       Label baseCase = new Label();
       code.compare(Comparison.LT, baseCase, i, constant2);//if (i < 2) {
       code.op(BinaryOp.SUBTRACT, a, i, constant1);//fib(i - 1)
       code.op(BinaryOp.SUBTRACT, b, i, constant2);//fib(i - 2)
       code.invokeStatic(fib, c, a);
       code.invokeStatic(fib, d, b);
       code.op(BinaryOp.ADD, result, c, d);
       code.returnValue(result);//return fib(i - 1) + fib(i - 2);
       code.mark(baseCase);//}
       code.returnValue(i);//return i
```

```Java
package example;
import com.google.dexmaker.BinaryOp;
import com.google.dexmaker.Code;
import com.google.dexmaker.Comparison;
import com.google.dexmaker.DexMaker;
import com.google.dexmaker.Label;
import com.google.dexmaker.Local;
import com.google.dexmaker.MethodId;
import com.google.dexmaker.TypeId;
import java.io.File;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;

public final class FibonacciMaker {
    public static void main(String[] args) throws Exception {
        TypeId<?> fibonacci = TypeId.get("Lcom/google/dexmaker/examples/Fibonacci;");

        String fileName = "Fibonacci.generated";
        DexMaker dexMaker = new DexMaker();
        dexMaker.declare(fibonacci, fileName, Modifier.PUBLIC, TypeId.OBJECT);

        MethodId<?, Integer> fib = fibonacci.getMethod(TypeId.INT, "fib", TypeId.INT);
        Code code = dexMaker.declare(fib, Modifier.PUBLIC | Modifier.STATIC);

        Local<Integer> i = code.getParameter(0, TypeId.INT);
        Local<Integer> constant1 = code.newLocal(TypeId.INT);
        Local<Integer> constant2 = code.newLocal(TypeId.INT);
        Local<Integer> a = code.newLocal(TypeId.INT);
        Local<Integer> b = code.newLocal(TypeId.INT);
        Local<Integer> c = code.newLocal(TypeId.INT);
        Local<Integer> d = code.newLocal(TypeId.INT);
        Local<Integer> result = code.newLocal(TypeId.INT);

        code.loadConstant(constant1, 1);
        code.loadConstant(constant2, 2);
        Label baseCase = new Label();
        code.compare(Comparison.LT, baseCase, i, constant2);
        code.op(BinaryOp.SUBTRACT, a, i, constant1);
        code.op(BinaryOp.SUBTRACT, b, i, constant2);
        code.invokeStatic(fib, c, a);
        code.invokeStatic(fib, d, b);
        code.op(BinaryOp.ADD, result, c, d);
        code.returnValue(result);
        code.mark(baseCase);
        code.returnValue(i);

        ClassLoader loader = dexMaker.generateAndLoad(
                FibonacciMaker.class.getClassLoader(), getDataDirectory());

        Class<?> fibonacciClass = loader.loadClass("com.google.dexmaker.examples.Fibonacci");
        Method fibMethod = fibonacciClass.getMethod("fib", int.class);
        System.out.println(fibMethod.invoke(null, 8));
    }

    public static File getDataDirectory() {
        String envVariable = "ANDROID_DATA";
        String defaultLoc = "/data";
        String path = System.getenv(envVariable);
        return path == null ? new File(defaultLoc) : new File(path);
    }
}
```

应该可以建立一个翻译工具，将java代码翻译为上面的预编译代码，从而实现动态下载java代码，在android上运行